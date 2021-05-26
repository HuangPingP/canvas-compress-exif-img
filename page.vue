<template>
  <div>
    <input
      type="file"
      id="avatar"
      name="avatar"
      accept="image/png, image/jpeg"
      @change="loadFile"
    />
  </div>
</template>

<script>
//@ canvas 压缩图片并保存原图片exif信息 核心思路
//@ 保存原图 Exif 信息，待图片压缩完成后，将原图 Exif 信息拼接到压缩图上。
//@ 校验图片base64图片信息   地址：  http://code.ciaoca.com/javascript/exif-js/demo/base64
//@ 参考： http://www.dirk.wang/2019/08/16/canvas%E5%8E%8B%E7%BC%A9jpg%E4%B8%A2%E5%A4%B1exif/
import EXIF from "exif-js";
var BasePage = zj.widgets.BasePage;
export default {
  mixins: [BasePage],
  components: {},
  data() {
    return {
      exifInfo: "",
    };
  },
  methods: {
    loadFile(event) {
      let file = event.target.files[0];
      //读取图片的元信息
      var orientation;
      EXIF.getData(file, function () {
        orientation = EXIF.getTag(this, "Orientation"); // 图片旋转的方向
      });

      let reader = new FileReader();
      let that = this;
      reader.onload = function () {
        let result = this.result;
        that.imgSrc = result;

        // 保存原有图片 exif信息
        let orignBuffer = that.base64ToArrayBuffer(result);
        let segments = that.getSegments(orignBuffer); //分割片段
        that.exifInfo = that.getEXIF(segments);

        //使用exif
        that.getImgData(this.result, orientation, (data) => {
          //这里可以使用校正后的图片data了
          var img = new Image();
          img.src = data;

          //图片加载完毕之后进行压缩，然后上传
          if (img.complete) {
            callback();
          } else {
            img.onload = callback;
          }

          function callback() {
            var data = that.compress(img);

            // 将原始图片exif信息拼接到压缩后的图片base64之后
            let orignBuffer = that.base64ToArrayBuffer(data);
            let newImg = that.insertEXIF(orignBuffer, that.exifInfo);
            let base64Img = that.transformArrayBufferToBase64(newImg);
            console.log("base64Img", base64Img);
            that.upload(base64Img, file.type, file.name);
          }
        });
      };

      reader.readAsDataURL(file);
      return false;
    },
    //上传图片
    upload(basestr, type, name) {
      let text = window.atob(basestr.split(",")[1]);
      let buffer = new ArrayBuffer(text.length);
      let ubuffer = new Uint8Array(buffer);

      for (let i = 0; i < text.length; i++) {
        ubuffer[i] = text.charCodeAt(i);
      }

      let Builder = window.WebKitBlobBuilder || window.MozBlobBuilder;

      let blob;
      if (Builder) {
        let builder = new Builder();
        builder.append(buffer);
        blob = builder.getBlob(type);
      } else {
        blob = new window.Blob([buffer], { type: type });
      }

      let formdata = new FormData();
      // formdata.append(model, blob, name);
    },
    //压缩图片
    compress(img) {
      //用于压缩图片的canvas
      let canvas = document.createElement("canvas");
      let ctx = canvas.getContext("2d");

      // 瓦片canvas
      var tCanvas = document.createElement("canvas");
      var tctx = tCanvas.getContext("2d");

      let initSize = img.src.length;
      let width = img.width;
      let height = img.height;

      //如果图片大于四百万像素，计算压缩比并将大小压至400万以下
      var ratio;
      if ((ratio = (width * height) / 4000000) > 1) {
        ratio = Math.sqrt(ratio);
        width /= ratio;
        height /= ratio;
      } else {
        ratio = 1;
      }
      canvas.width = width * 2;
      canvas.height = height * 2;
      //铺底色
      ctx.fillStyle = "#fff";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      //如果图片像素大于100万则使用瓦片绘制
      var count;
      if ((count = (width * height) / 4000000) > 1) {
        count = ~~(Math.sqrt(count) + 1); //计算要分成多少块瓦片
        //计算每块瓦片的宽和高
        var nw = ~~(width / count);
        var nh = ~~(height / count);
        tCanvas.width = nw;
        tCanvas.height = nh;
        for (var i = 0; i < count; i++) {
          for (var j = 0; j < count; j++) {
            tctx.drawImage(
              img,
              i * nw * ratio,
              j * nh * ratio,
              nw * ratio * 2,
              nh * ratio * 2,
              0,
              0,
              nw,
              nh
            );
            ctx.drawImage(tCanvas, i * nw, j * nh, nw * 2, nh * 2);
          }
        }
      } else {
        ctx.drawImage(img, 0, 0, width * 2, height * 2);
      }
      //进行最小压缩
      let ndata = canvas.toDataURL("image/jpeg", 1);

      console.log(
        "压缩前：" +
          initSize +
          "，压缩后：" +
          ndata.length +
          "，压缩率：" +
          ~~((100 * (initSize - ndata.length)) / initSize) +
          "%"
      );

      return ndata;
    },
    getImgData(img, dir, next) {
      // @param {string} img 图片的base64
      // @param {int} dir exif获取的方向信息
      // @param {function} next 回调方法，返回校正方向后的base64
      var image = new Image();
      image.onload = function () {
        var degree = 0,
          drawWidth,
          drawHeight,
          width,
          height;
        drawWidth = this.naturalWidth;
        drawHeight = this.naturalHeight;
        //以下改变一下图片大小
        var maxSide = Math.max(drawWidth, drawHeight);
        if (maxSide > 1024) {
          var minSide = Math.min(drawWidth, drawHeight);
          minSide = (minSide / maxSide) * 1024;
          maxSide = 1024;
          if (drawWidth > drawHeight) {
            drawWidth = maxSide;
            drawHeight = minSide;
          } else {
            drawWidth = minSide;
            drawHeight = maxSide;
          }
        }
        var canvas = document.createElement("canvas");
        canvas.width = width = drawWidth;
        canvas.height = height = drawHeight;
        var context = canvas.getContext("2d");
        //判断图片方向，重置canvas大小，确定旋转角度，iphone默认的是home键在右方的横屏拍摄方式
        switch (dir) {
          //iphone横屏拍摄，此时home键在左侧
          case 3:
            degree = 180;
            drawWidth = -width;
            drawHeight = -height;
            break;
          //iphone竖屏拍摄，此时home键在下方(正常拿手机的方向)
          case 6:
            canvas.width = height;
            canvas.height = width;
            degree = 90;
            drawWidth = width;
            drawHeight = -height;
            break;
          //iphone竖屏拍摄，此时home键在上方
          case 8:
            canvas.width = height;
            canvas.height = width;
            degree = 270;
            drawWidth = -width;
            drawHeight = height;
            break;
        }
        //使用canvas旋转校正
        context.rotate((degree * Math.PI) / 180);
        context.drawImage(this, 0, 0, drawWidth, drawHeight);
        //返回校正图片
        next(canvas.toDataURL("image/jpeg", 0.4));
      };
      image.src = img;
    },

    //@ canvas 压缩图片保存原图片exif核心代码
   
    // 工具函数 将 base64 转 ArrayBuffer
    base64ToArrayBuffer(base64, contentType) {
      contentType =
        contentType || base64.match(/^data\:([^\;]+)\;base64,/im)[1] || ""; // e.g. 'data:image/jpeg;base64,...' => 'image/jpeg'
      base64 = base64.replace(/^data\:([^\;]+)\;base64,/gim, "");
      // btoa是binary to ascii，将binary的数据用ascii码表示，即Base64的编码过程
      // atob则是ascii to binary，用于将ascii码解析成binary数据
      var binary = atob(base64);
      var len = binary.length;
      var buffer = new ArrayBuffer(len);
      var view = new Uint8Array(buffer);
      for (var i = 0; i < len; i++) {
        view[i] = binary.charCodeAt(i);
      }
      return buffer;
    },
    transformArrayBufferToBase64(buffer) {
      var binary = "";
      var bytes = new Uint8Array(buffer);
      for (var len = bytes.byteLength, i = 0; i < len; i++) {
        binary += String.fromCharCode(bytes[i]);
      }
      return `data:image/jpeg;base64,${window.btoa(binary)}`;
    },

    // 获取 0xFFE0~0xFFEF 开头的应用标记片段
    getSegments(arrayBuffer) {
      var head = 0,
        segments = [];
      var length, endPoint, seg;
      var arr = [].slice.call(new Uint8Array(arrayBuffer), 0);

      while (true) {
        // SOS(Start of Scan, 由 0xff 0xda 开头)
        // 遍历到 SOS 表示已经遍历完所有标记，再往下就是图像数据流了，直接 break
        if (arr[head] === 0xff && arr[head + 1] === 0xda) {
          break;
        }

        // SOI(Start of Image)是 JPG 文件的开头内容，由 0xff 0xd8 开头
        if (arr[head] === 0xff && arr[head + 1] === 0xd8) {
          head += 2;
        }
        // 找出每个标记片段
        else {
          // 每个标记开头后跟着的两个字节记录了该标记所记录内容的长度
          length = arr[head + 2] * 256 + arr[head + 3]; // 内容长度
          endPoint = head + length + 2; // 内容结束位置
          // 从0xff开头，到标记数据内容结束全部截出来
          seg = arr.slice(head, endPoint);
          head = endPoint;
          // push整个标记信息
          segments.push(seg);
        }
        if (head > arr.length) {
          break;
        }
      }
      return segments;
    },
    // 从标记片段筛选 & 取出 exif 信息
    getEXIF(segments) {
      if (!segments.length) {
        return [];
      }
      var seg = [];
      for (var x = 0; x < segments.length; x++) {
        var s = segments[x];
        // 0xff 0xe1开头的才是 exif数据(即app1)
        if (s[0] === 0xff && s[1] === 0xe1) {
          // app1 exif 0xff 0xe1
          seg = seg.concat(s);
        }
      }
      return seg;
    },
    // 拼接 Exif 到压缩后的 base64 中：
    // 插入 Exif 信息
    insertEXIF(resizedImg, exifArr) {
      var arr = [].slice.call(new Uint8Array(resizedImg), 0);
      //不是标准的JPEG文件
      if (arr[2] !== 0xff || arr[3] !== 0xe0) {
        return resizedImg;
      }
      var app0_length = arr[4] * 256 + arr[5]; //两个字节

      // 拼接文件 SOI + EXIF + 去除APP0的图像信息
      var newImg = [0xff, 0xd8].concat(exifArr, arr.slice(4 + app0_length));
      return new Uint8Array(newImg);
    },
  },
};
</script>

<style lang='less' scoped>
</style>
