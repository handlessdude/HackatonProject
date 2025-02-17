<template>
  <div id="app" class="web-camera-container">
    <div class="camera-button">
      <button type="button" class="button is-rounded" :class="{ 'is-primary' : !isCameraOpen, 'is-danger' : isCameraOpen}" @click="toggleCamera">
        <span v-if="!isCameraOpen">Open Camera</span>
        <span v-else>Close Camera</span>
      </button>
    </div>

    <div v-show="isCameraOpen && isLoading" class="camera-loading">
      <ul class="loader-circle">
        <li></li>
        <li></li>
        <li></li>
      </ul>
    </div>

    <div v-if="isCameraOpen" v-show="!isLoading" class="camera-box" :class="{ 'flash' : isShotPhoto }">

      <div class="camera-shutter" :class="{'flash' : isShotPhoto}"></div>

      <video v-show="!isPhotoTaken" ref="camera" :width="450" :height="337.5" autoplay></video>

      <canvas v-show="isPhotoTaken" id="photoTaken" ref="canvas" :width="450" :height="337.5"></canvas>
    </div>

    <div v-if="isCameraOpen && !isLoading" class="camera-shoot">
      <button type="button" class="button" @click="startCapturing">
        <img src="https://img.icons8.com/ios/50/000000/anonymous-mask.png"/>
      </button>
    </div>

    <!--    <div v-if="isCameraOpen && !isLoading" class="camera-shoot">
          <button type="button" class="button" @click="takePhoto">
            <img src="https://img.icons8.com/material-outlined/50/000000/camera&#45;&#45;v2.png">
          </button>
        </div>

        <div v-if="isPhotoTaken && isCameraOpen" class="camera-download">
          <a id="downloadPhoto" download="my-photo.jpg" class="button" role="button" @click="downloadImage">
            Download
          </a>
        </div>-->
  </div>

</template>

<script>
import Requester from "@/api/requester"

export default {
  data() {
    return {
      isCameraOpen: false,
      isPhotoTaken: false,
      isShotPhoto: false,
      isLoading: false,
      link: '#',
      timeout: 1000,

      // that is what we alter on the front for ml testing:
      interval: 500,
      vidLen: 2000,
      curTask: "neutral",
    }
  },

  methods: {
    toggleCamera() {
      if(this.isCameraOpen) {
        this.isCameraOpen = false;
        this.isPhotoTaken = false;
        this.isShotPhoto = false;
        this.stopCameraStream();
      } else {
        this.isCameraOpen = true;
        this.createCameraElement();
      }
    },

    createCameraElement() {
      this.isLoading = true;

      const constraints = (window.constraints = {
        audio: false,
        video: true
      });


      navigator.mediaDevices
          .getUserMedia(constraints)
          .then(stream => {
            this.isLoading = false;
            this.$refs.camera.srcObject = stream;
          })
          .catch(e => {
            this.isLoading = false;
            alert("May the browser didn't support or there is some errors: " + e.message);
          });
    },

    stopCameraStream() {
      let tracks = this.$refs.camera.srcObject.getTracks();

      tracks.forEach(track => {
        track.stop();
      });
    },

    takePhoto() {
      if(!this.isPhotoTaken) {
        this.isShotPhoto = true;

        const FLASH_TIMEOUT = 50;

        setTimeout(() => {
          this.isShotPhoto = false;
        }, FLASH_TIMEOUT);
      }

      this.isPhotoTaken = !this.isPhotoTaken;

      const context = this.$refs.canvas.getContext('2d');
      context.drawImage(this.$refs.camera, 0, 0, 450, 337.5);
    },

    downloadImage() {
      //const download = document.getElementById("downloadPhoto");
      //const canvas = document.getElementById("photoTaken").toDataURL("image/jpeg")
      //.replace("image/jpeg", "image/octet-stream");
      //download.setAttribute("href", canvas);
      const base64 = document.getElementById("photoTaken").toDataURL("image/jpeg")
      //console.log(base64)
      return base64
    },

    async postNewImg(cnt){
      this.takePhoto()
      const newImageString = this.downloadImage()
      //post to server
      try {
        const response = Requester.postData('http://localhost:8080/receivePhoto',
            {
              imageString: newImageString,
              task: this.curTask,
              photoCount: cnt
            })
        console.log(response)
        return response
      } catch (e) {
        console.log(e)
        return e
      }
    },

    startCapturing() {
      const clbk = this.postNewImg
      const y = this.vidLen
      const x = this.interval

      setTimeout(function() {
        //console.log('x, y = ', x, y )
        const photoCount = parseInt(y / x)
        //console.log('photoNum = ', photoCount)
        var timesRun = 0
        var intervalFn = setInterval(function(){
          console.log(timesRun)
          clbk(photoCount)
          timesRun += 1
          if(timesRun === photoCount){
            clearInterval(intervalFn)
          }
        }, 500);

      }, this.timeout);
    },

  }

}
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

body {
  display: flex;
  justify-content: center;
  align-items: center;
}

.web-camera-container {
  margin-top: 2rem;
  margin-bottom: 2rem;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 500px;


  .camera-button {
    margin-bottom: 2rem;
  }

  .camera-box {
    .camera-shutter {
      opacity: 0;
      width: 450px;
      height: 337.5px;
      background-color: #fff;
      position: absolute;

      &.flash {
        opacity: 1;
      }
    }
  }

  .camera-shoot {
    margin: 1rem 0;

    button {
      height: 60px;
      width: 60px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 100%;

      img {
        height: 35px;
        object-fit: cover;
      }
    }
  }

  .camera-loading {
    overflow: hidden;
    height: 100%;
    position: absolute;
    width: 100%;
    min-height: 150px;
    margin: 3rem 0 0 -1.2rem;

    ul {
      height: 100%;
      position: absolute;
      width: 100%;
      z-index: 999999;
      margin: 0;
    }

    .loader-circle {
      display: block;
      height: 14px;
      margin: 0 auto;
      top: 50%;
      left: 100%;
      transform: translateY(-50%);
      transform: translateX(-50%);
      position: absolute;
      width: 100%;
      padding: 0;

      li {
        display: block;
        float: left;
        width: 10px;
        height: 10px;
        line-height: 10px;
        padding: 0;
        position: relative;
        margin: 0 0 0 4px;
        background: #999;
        animation: preload 1s infinite;
        top: -50%;
        border-radius: 100%;

        &:nth-child(2) {
          animation-delay: .2s;
        }

        &:nth-child(3) {
          animation-delay: .4s;
        }
      }
    }
  }

  @keyframes preload {
    0% {
      opacity: 1
    }
    50% {
      opacity: .4
    }
    100% {
      opacity: 1
    }
  }
}
</style>
