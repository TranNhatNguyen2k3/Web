<template>
  <q-page class="flex flex-center bg-light">
    <div class="q-gutter-md container">
      <q-btn
        @click="enableCam"
        :label="webcamRunning ? 'Disable Webcam' : 'Enable Webcam'"
        color="primary"
        class="q-mb-md"
        style="width: 200px"
      />
      <div class="video-container">
        <video ref="webcam" autoplay playsinline muted class="video" />
        <canvas ref="outputCanvas" class="canvas" />
      </div>
      <div v-if="gestureOutput" class="q-mt-md">
        <q-card>
          <q-card-section class="text-center">
            <h5 class="text-h6">Gesture Output</h5>
            <div>{{ gestureOutput }}</div>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script>
import {
  GestureRecognizer,
  FilesetResolver,
  DrawingUtils,
} from "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3";

export default {
  data() {
    return {
      gestureRecognizer: null,
      webcamRunning: false,
      gestureOutput: "",
      lastVideoTime: -1,
    };
  },
  async mounted() {
    await this.createGestureRecognizer();
  },
  methods: {
    async createGestureRecognizer() {
      const vision = await FilesetResolver.forVisionTasks(
        "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm"
      );
      this.gestureRecognizer = await GestureRecognizer.createFromOptions(
        vision,
        {
          baseOptions: {
            modelAssetPath:
              "https://storage.googleapis.com/mediapipe-models/gesture_recognizer/gesture_recognizer/float16/1/gesture_recognizer.task",
            delegate: "GPU",
          },
          runningMode: "VIDEO",
        }
      );
    },
    async enableCam() {
      if (!this.gestureRecognizer) {
        alert("Please wait for the gesture recognizer to load");
        return;
      }

      this.webcamRunning = !this.webcamRunning;

      if (this.webcamRunning) {
        this.$refs.webcam.srcObject = await navigator.mediaDevices.getUserMedia(
          { video: true }
        );
        this.$refs.webcam.addEventListener("loadeddata", this.predictWebcam);
      } else {
        this.$refs.webcam.srcObject
          .getTracks()
          .forEach((track) => track.stop());
        this.gestureOutput = "";
      }
    },
    async predictWebcam() {
      let nowInMs = Date.now();

      if (this.lastVideoTime !== this.$refs.webcam.currentTime) {
        this.lastVideoTime = this.$refs.webcam.currentTime;
        const results = this.gestureRecognizer.recognizeForVideo(
          this.$refs.webcam,
          nowInMs
        );

        const canvasCtx = this.$refs.outputCanvas.getContext("2d");
        canvasCtx.clearRect(
          0,
          0,
          this.$refs.outputCanvas.width,
          this.$refs.outputCanvas.height
        );

        const drawingUtils = new DrawingUtils(canvasCtx);
        this.$refs.outputCanvas.width = this.$refs.webcam.videoWidth;
        this.$refs.outputCanvas.height = this.$refs.webcam.videoHeight;

        if (results.landmarks) {
          for (const landmarks of results.landmarks) {
            drawingUtils.drawConnectors(
              landmarks,
              GestureRecognizer.HAND_CONNECTIONS,
              {
                color: "#00FF00",
                lineWidth: 5,
              }
            );
            drawingUtils.drawLandmarks(landmarks, {
              color: "#FF0000",
              lineWidth: 2,
            });
          }
        }

        if (results.gestures.length > 0) {
          const categoryName = results.gestures[0][0].categoryName;
          const categoryScore = (results.gestures[0][0].score * 100).toFixed(2);
          const handedness = results.handednesses[0][0].displayName;
          this.gestureOutput = `GestureRecognizer: ${categoryName}\n Confidence: ${categoryScore}%\n Handedness: ${handedness}`;
        } else {
          this.gestureOutput = "";
        }
      }

      if (this.webcamRunning) {
        requestAnimationFrame(this.predictWebcam);
      }
    },
  },
};
</script>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  background-color: white;
}

.video-container {
  position: relative;
  width: 100%;
  overflow: hidden;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.video {
  width: 100%;
  height: auto;
  border-radius: 10px;
}

.canvas {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  border-radius: 10px;
  pointer-events: none;  
}

.q-card {
  background-color: #f7f7f7;  
  border-radius: 10px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}
</style>
