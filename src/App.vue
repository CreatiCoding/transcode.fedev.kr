<template>
  <div id="app">
    <div>
      <input type="file" @change="previewFile" />
      <br />
      <img id="preview1" src height="200" alt="이미지 미리보기..." />
      <video id="preview2" src height="200" alt="동영상 미리보기..." />
      <div>{{ size / 1000 }}KB</div>
      <button @click="doConvert">다운로드</button>
    </div>
  </div>
</template>

<script>
import { createWorker, setLogging } from "@ffmpeg/ffmpeg";
setLogging(true);

export default {
  name: "App",
  data() {
    return {
      gif_data: null,
      percent: 0,
      size: 0
    };
  },
  methods: {
    previewFile() {
      var preview1 = document.querySelector("#preview1");
      var preview2 = document.querySelector("#preview2");
      var file = document.querySelector("input[type=file]").files[0];
      var reader = new FileReader();
      reader.addEventListener(
        "load",
        () => {
          preview1.src = reader.result;
          preview2.src = reader.result;
          this.gif_data = reader.result;
        },
        false
      );
      if (file) {
        reader.readAsDataURL(file);
      }
    },
    doConvert() {
      const worker = createWorker({
        logger: a => {
          if (a.message.indexOf("frame") !== -1) {
            if (a.message.indexOf("size=") !== -1) {
              const s = a.message.indexOf("size=") + 5;
              const e = a.message.indexOf("bytes");
              const size = parseInt(a.message.substring(s, e));
              if (!isNaN(size)) {
                this.size += size;
              }
            }
          }
        }
      });
      (async () => {
        await worker.load();
        await worker.write("test.gif", this.gif_data);
        await worker.transcode(
          "test.gif",
          "result.mp4",
          " -pix_fmt yuv420p -c:v libx264 -v:b 1000000"
        );
        const { data } = await worker.read("result.mp4");

        const blob = new Blob([data], { type: "octet/stream" });
        const url = window.URL.createObjectURL(blob);
        const a = document.createElement("a");
        document.body.appendChild(a);
        a.href = url;
        a.download = "result.mp4";
        a.click();
        setTimeout(() => {
          window.URL.revokeObjectURL(url);
          document.body.removeChild(a);
        }, 0);
        await worker.terminate();
      })();
    }
  }
};
</script>
