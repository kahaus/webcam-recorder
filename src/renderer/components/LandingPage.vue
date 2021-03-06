<template>
  <div id="wrapper">
    <h1>Camera recorder</h1>
    <div id="gui" v-show="interactive">
      <select id="videoSource" ref="videoSelect" v-model="deviceID" :disabled="recording">
      </select>
      <select v-model="resolutionID" :disabled="recording">
        <option v-for="(option, index) in resolutions" v-bind:value="index">
          {{ option.name }}
        </option>
      </select>
      <button id="autotest" v-on:click="startAutoTest" :disabled="recording">
        autotest
      </button>
      <button v-if="!recording" v-on:click="startRecording">
        record
      </button>
      <button v-else v-on:click="stopRecording">
        stop
      </button>
      <button v-on:click="download" :disabled="recording">
        download
      </button>
      <input type="checkbox" id="check-monitor" v-model="displayMonitor">
      <label for="check-monitor">Monitor</label>
      <input type="checkbox" id="check-playback" v-model="displayPlayback">
      <label for="check-playback">Playback</label>
    </div>
    <main>
      <div v-show="displayMonitor" class="monitor">
        <div>Monitor</div>
        <video ref="monitor" playsinline></video>
      </div>
      <div v-show="displayPlayback && interactive" class="playback">
        <div>Playback</div>
        <video ref="playback" playsinline controls></video>
      </div>
    </main>
  </div>
</template>

<script>
  import settings from '@/lib/settings'
  import recorder from '@/lib/recorder'

  let VERBOSE = false

  export default {
    name: 'landing-page',
    data () {
      return {
        stream: null,
        duration: 5000,
        autotesting: false,
        resolutionID: 2,
        deviceID: '',
        recording: false,
        displayMonitor: true,
        displayPlayback: true
      }
    },
    computed: {
      cameraSettings () {
        if (this.stream) {
          return this.stream.getVideoTracks()[0].getSettings()
        } else {
          return null
        }
      },
      currentResolution () {
        return settings.camera.resolutions[this.resolutionID]
      },
      resolutions () {
        return settings.camera.resolutions
      },
      interactive () {
        return !this.autotesting
      }
    },
    watch: {
      resolutionID (id) {
        this.initWebcam()
      },
      deviceID (id) {
        this.initWebcam()
      },
      interactive (value) {
        document.onkeypress = value ? this.keyPressed : null
      }
    },
    mounted () {
      // Using the new API, @see
      // https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
      // for backward compatibility
      if (navigator.mediaDevices === undefined) {
        console.warn('Browser\'s version is too old')
        return
      }

      this.commands = {
        ' ': this.toggleRecording,
        'p': this.togglePlayback,
        'd': this.download
      }

      window.recorder = recorder
      window.vm = this

      navigator.mediaDevices.enumerateDevices().then(this.initDevices)

      this.initWebcam()
    },
    methods: {
      getCameraName (id) {
        let opts = this.$refs.videoSelect.options
        for (let i = 0; i < opts.length; i++) {
          let opt = opts[i]
          if (opt.value === id) {
            return opt.text
          }
        }
      },
      initDevices (deviceInfos) {
        let videoSelect = this.$refs.videoSelect
        for (let i = 0; i !== deviceInfos.length; ++i) {
          let deviceInfo = deviceInfos[i]
          let option = document.createElement('option')
          option.value = deviceInfo.deviceId
          if (deviceInfo.kind === 'videoinput') {
            option.text = deviceInfo.label || 'camera ' + (videoSelect.length + 1)
            videoSelect.appendChild(option)
          }
        }
      },
      initWebcam () {
        if (this.stream) {
          recorder.flush()
          this.$refs.playback.src = ''
          this.stream.getTracks().forEach(track => {
            track.stop()
          })
        }

        let constraints = {
          audio: false,
          video: this.currentResolution.value
        }
        if (this.deviceID !== '') {
          constraints.video.deviceId = this.deviceID
        }

        // console.log(JSON.stringify(constraints, null, 2))

        let vm = this
        navigator.mediaDevices.getUserMedia(constraints)
          .then(function (stream) {
            vm.stream = stream
            let settings = stream.getVideoTracks()[0].getSettings()
            console.log('Found camera with settings: ' + JSON.stringify(settings, null, 2))
            if (vm.deviceID !== settings.deviceId) {
              vm.deviceID = settings.deviceId
              return
            }
            vm.playWebcam(stream)
          })
          .catch(function (err) {
            console.error('Cannot access the webcam:', err)
          })
      },
      playWebcam (stream) {
        VERBOSE && console.log('playWebcam')
        this.$refs.monitor.srcObject = stream
        this.$refs.monitor.play()

        if (this.autotesting) {
          this.startRecording()
        }
      },
      startAutoTest () {
        this.autotesting = true
        if (this.resolutionID === 0) {
          this.startRecording()
        } else {
          this.resolutionID = 0
        }
      },
      nextTest () {
        if (this.resolutionID >= this.resolutions.length - 1) {
          this.autotesting = false
          return
        }
        this.resolutionID = this.resolutionID + 1
      },
      startRecording () {
        VERBOSE && console.log('startRecording')
        recorder.startRecording(this.stream)
        this.recording = true

        if (this.autotesting) {
          setTimeout(this.stopRecording, settings.autotest.duration)
        }
      },
      stopRecording () {
        VERBOSE && console.log('stopRecording')
        recorder.stopRecording()
        this.$refs.playback.src = recorder.url()
        this.recording = false

        if (this.autotesting) {
          this.download()
          setTimeout(this.nextTest, settings.autotest.wait)
        }
      },
      download () {
        if (!recorder.isVideoAvailable()) {
          return
        }
        let tag = this.getCameraName(this.deviceID) + '-' + this.currentResolution.name
        recorder.download(tag)
      },
      //
      // - Handle keyboard events
      //
      keyPressed (evt) {
        evt = evt || window.event
        let charCode = evt.keyCode || evt.which
        let charStr = String.fromCharCode(charCode)
        // console.log('key pressed:', charStr)
        if (this.commands[charStr]) {
          this.commands[charStr]()
        }
      },
      toggleRecording () {
        if (recorder.isRecording()) {
          this.stopRecording()
        } else {
          this.startRecording()
        }
      },
      togglePlayback () {
        this.$refs.playback.play()
      }
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body { font-family: 'Source Sans Pro', sans-serif; }

  h1 {
    font-size: 80px;
    font-weight: bold;
    padding-bottom: 2vh;
    text-transform: uppercase;
  }

  button, select {
    margin-right: 1vh;
    font-size: 30px;
    width: 10vw;
    text-transform: uppercase;
    padding: 6px;
    background-color: slategrey;
    color: white;
    border: none;
  }

   #autotest {
     background-color: slateblue;
   }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }

  main {
    /* display: flex; */
    /* justify-content: space-between; */
    font-size: 50px;
    text-transform: uppercase;
    color: darkcyan;
  }

  :disabled {
    opacity: 0.5;
  }

  .playback, .monitor {
    margin-top: 5vh;
  }
</style>
