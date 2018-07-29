<template>
  <div>
    <span id="status">{{status}}</span>
    <a href="#" @click="getSources">getSources</a>
    <a href="#" @click="rec">Rec</a>
    <a href="#" @click="stop">Stop</a>
    <a id="download">Download</a>
    <div><input class="size-input" type="text" :value="width" placeholder="width"> x
    <input class="size-input" type="text" :value="height" placeholder="height"></div>
    <select id="sourceSelector" @change="selectChange" :disabled="mediaRecorder">
      <option v-for="option in options" :value="option.id" :key="option.id" >
        {{ option.name }}
        <!-- <img :href="option.thumbnail.toDataURL()"> -->
      </option>
    </select>
    <video id="screen"></video>
  </div>
</template>

<script>
import { desktopCapturer } from "electron";

const MODULE_NAME = `Recorder.vue`;

function _getSources() {
  return new Promise(function(s, t) {
    desktopCapturer.getSources(
      { types: ["window", "screen"] },
      (error, sources) => {
        if (error) {
          t(error);
        } else {
          s(sources);
        }
      }
    );
  });
}

function _previewStream(stream) {
  const video = document.querySelector("#screen");
  // video.srcObject = stream;
  video.src = URL.createObjectURL(stream);
  video.autoplay = true;
  // video.onloadedmetadata = e => video.play();
  return video;
}

function _recordStream(stream) {
  const options = { mimeType: "video/webm" };
  const recordedChunks = [];
  const mediaRecorder = new MediaRecorder(stream, options);
  mediaRecorder.ondataavailable = e => {
    recordedChunks.push(e.data);
  };
  mediaRecorder.onstop = e => {
    const downloadLink = document.querySelector("#download");
    downloadLink.href = URL.createObjectURL(new Blob(recordedChunks));
    downloadLink.download = `video_${Date.now()}.webm`;
  };
  mediaRecorder.start();
  return mediaRecorder; // mediaRecorder.stop()
}

function _getUserMedia(source) {
  return navigator.mediaDevices
    .getUserMedia({
      audio: false,
      video: {
        mandatory: {
          chromeMediaSource: "desktop",
          chromeMediaSourceId: source.id,
          minWidth: source.width,
          maxWidth: source.width,
          minHeight: source.height,
          maxHeight: source.height
        }
      }
    })
    .catch(e => {
      console.error(`[${MODULE_NAME}._getUserMedia]`, e);
      return Promise.reject(e);
    });
}

export default {
  data() {
    return {
      mediaRecorder: undefined,
      mediaPreview: undefined,
      options: [],
      width: 1920,
      height: 1080,
      status: "■"
    };
  },
  created(params) {
    const self = this;
    _getSources().then(function(sources) {
      self.options = sources;
      document
        .querySelector("#sourceSelector")
        .dispatchEvent(new Event("change"));
      // self.selectChange()
    });
  },
  methods: {
    rec(ev) {
      const self = this;
      if (self.mediaRecorder) {
        return;
      }
      Promise.resolve()
        .then(function() {
          if (self.mediaPreview) {
            return self.mediaPreview.clone();
          } else {
            const s = {
              id: ev.target.value,
              width: this.width,
              height: this.height
            };
            return _getUserMedia(s);
          }
        })
        .then(_recordStream)
        .then(function(mediaRecorder) {
          self.status = "▶";
          self.mediaRecorder = mediaRecorder;
        })
        .catch(function(err) {
          console.error(`[${MODULE_NAME}.rec]`, e);
        });
    },
    stop(ev) {
      const self = this;
      if (!self.mediaRecorder) {
        return;
      }
      self.mediaRecorder.stop();
      self.status = "■";
      self.mediaRecorder = null;
    },
    // download(ev) {
    //   getData();
    // },
    selectChange(ev) {
      const self = this;
      const s = {
        id: ev.target.value,
        width: this.width,
        height: this.height
      };
      let mediaPreview;
      _getUserMedia(s)
        .then(function(mediaRecorder) {
          mediaPreview = mediaRecorder;
          return _previewStream(mediaRecorder);
        })
        .then(function(domVideo) {
          self.mediaPreview = mediaPreview;
        })
        .catch(function(err) {
          console.error(`[${MODULE_NAME}.selectChange]`, e);
        });
    },
    getSources(ev) {
      _getSources().then(function(sources) {
        self.options = sources;
      });
    }
  }
};
</script>

<style scoped>
video {
  border: #6a6a6a solid 1px;
  /* margin: auto; */
  width: 210px;
  height: 140px;
}
select {
  width: 95%;
}
.size-input {
  width: 4em;
}
/* 

.item {
  display: flex;
  margin-bottom: 6px;
}

.item .name {
  color: #6a6a6a;
  margin-right: 6px;
}

.item .value {
  color: #35495e;
  font-weight: bold;
} */
</style>
