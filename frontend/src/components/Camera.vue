<template>
  <div class="video-container" style="background-color: #212121">
    <Detail v-model="detailDialog" />
    <div>
      <v-progress-circular
        indeterminate
        v-if="isLoading"
        class="spinner"
        :size="100"
        :width="7"
      />
    </div>
    <div justify-center align-center class="scanInfo mx-auto" v-if="!isLoading">
      <v-layout
        justify-center
        align-center
        fill-height
        v-if="product.name != null && !pathCheck(3)"
      >
        <div style="text-align: center">
          <div>
            <span class="accentInfo">
              {{ product.name }} ({{ product.type }})
            </span>
            <span class="secondaryInfo"></span>
          </div>
          <div v-if="pathCheck(1)">
            <v-btn icon @click="openDetailDialog()" aria-label="세부 정보 조회">
              <v-icon color="accent">mdi-information</v-icon>
              <span class="detailInfo"> 세부정보 확인하기</span>
            </v-btn>
          </div>
          <div v-else-if="pathCheck(2)">
            <v-btn icon @click="favoriteRegist()" aria-label="즐겨찾기 등록">
              <v-icon color="accent">mdi-clipboard-list-outline</v-icon>
              <span class="detailInfo"> 즐겨찾기 등록</span>
            </v-btn>
          </div>
        </div>
      </v-layout>
      <v-layout justify-center align-center fill-height v-if="pathCheck(3)">
        <span>
          <div class="alertInfo">
            <div class="searchInfo">{{ searchName }} ({{ searchType }})</div>
            <div v-html="replaceHtml(searchAlertText)"></div>
          </div>
        </span>
      </v-layout>
      <v-layout
        justify-center
        align-center
        fill-height
        v-if="!pathCheck(3) && !isLoading && product.name == null"
      >
        <span
          ><div class="alertInfo">{{ this.alertText }}</div></span
        >
      </v-layout>
    </div>
    <video ref="camera" muted="muted" autoplay playisline></video>
    <canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
  </div>
</template>

<script>
import Detail from "./Detail.vue";
import { mapActions, mapGetters } from "vuex";
import * as tf from "@tensorflow/tfjs";
import { loadGraphModel } from "@tensorflow/tfjs-converter";
tf.setBackend("webgl");
import classesDir from "@/assets/js/drink.js";

const MODEL_URL =
  "https://raw.githubusercontent.com/Ji2z/vuetest/master/model10/model.json";
const threshold = 0.75;

let cntMap = new Map();
let nameSet = new Set();

export default {
  name: "Camera",
  props: {
    path: String,
  },
  components: {
    Detail,
  },
  data() {
    return {
      detailDialog: false,
      dialog: [],

      streamPromise: null,
      modelPromise: null,
      video: null,

      isVideoStreamReady: false,
      isModelReady: false,
      initFailMessage: "",

      model: null,
      timer: null,
      videoRatio: 1,
      resultWidth: 0,
      resultHeight: 0,

      ttsText: null,

      product: {
        name: null,
        type: null,
      },
      alertText: "감지된 음료가 없습니다.",
      searchAlertText: "탐색 음료가 존재하지 않습니다.",

      searchName: "",
      searchType: "",

      isLoading: true,
      utterance: null,
      count: 0,

      isIOS: false,
    };
  },
  computed: {
    ...mapGetters(["getMute", "getIsDetect"]),
  },
  methods: {
    ...mapActions(["getProductDetail", "storeIsDetect"]),
    iOSTTS(input) {
      if (typeof window === "undefined") {
        return;
      }
      const isiOS =
        navigator.platform && /iPad|iPhone|iPod/.test(navigator.platform);
      if (!isiOS) {
        return;
      }
      const simulateSpeech = () => {
        const lecture = new SpeechSynthesisUtterance(input);
        lecture.volume = 0;
        speechSynthesis.speak(lecture);
        document.removeEventListener("click", simulateSpeech);
      };

      document.addEventListener("click", simulateSpeech);
      this.isIOS = true;
    },
    async tts(input) {
      const isiOS =
        navigator.platform && /iPad|iPhone|iPod/.test(navigator.platform);
      if (isiOS && !this.isIOS) {
        await this.iOSTTS(input);
      }
      if ((this.ttsText != null && this.ttsText == input) || !this.getMute)
        return;
      if (
        this.utterance != null &&
        this.ttsText != null &&
        this.ttsText != input
      )
        window.speechSynthesis.cancel();
      this.ttsText = input;
      this.utterance = new SpeechSynthesisUtterance(input);
      if (!isiOS) this.utterance.rate = 1.9;
      window.speechSynthesis.speak(this.utterance);
    },
    replaceHtml(input) {
      return input.replace("\n", "<br />");
    },
    favoriteRegist() {
      let storeProduct = {
        name: this.product.name,
        type: this.product.type,
      };
      let storeMap = [];
      if (localStorage.getItem("favorite") == null) {
        storeMap.push(storeProduct);
      } else {
        storeMap = JSON.parse(localStorage.getItem("favorite"));
        if (
          !storeMap.some(
            (item) =>
              item.name === storeProduct.name && item.type === storeProduct.type
          )
        ) {
          storeMap.push(storeProduct);
        }
      }
      localStorage.setItem("favorite", JSON.stringify(storeMap));
      this.$router.go(-1);
    },
    pathCheck(pathNum) {
      if (this.path == "scan" && pathNum == 1) return true;
      else if (this.path == "search" && pathNum == 3) return true;
      else if (this.path == "favoriteAdd" && pathNum == 2) return true;
      return false;
    },
    // 세부정보 모달
    async openDetailDialog() {
      if (this.product.name != null) {
        await this.getProductDetail(this.product);
        this.detailDialog = true;
      }
    },
    closeDeleteDialog() {
      this.$set(this.dialog, false);
    },
    // 웹캠 초기화
    initWebcamStream() {
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices
          .getUserMedia({
            audio: false,
            video: { facingMode: "environment" },
          })
          .then((stream) => {
            this.video = this.$refs.camera;
            try {
              this.video.srcObject = stream;
            } catch (error) {
              this.video.src = URL.createObjectURL(stream);
            }

            return new Promise((resolve) => {
              this.video.onloadedmetadata = () => {
                this.videoRatio =
                  this.video.offsetHeight / this.video.offsetWidth;
                this.isVideoStreamReady = true;
                resolve();
              };
            });
          })
          .catch((error) => {
            throw error;
          });
      }
    },
    // 인공지능 모델 불러오기
    loadModel() {
      this.isModelReady = false;
      return loadGraphModel(MODEL_URL)
        .then((model) => {
          this.model = model;
          this.isModelReady = true;
        })
        .catch((error) => {
          throw error;
        });
    },
    // 감지 모델 저장 및 초기화
    initMap() {
      let min = this.count / 2;
      cntMap.forEach(function (item, index) {
        if (item >= min) nameSet.add(index);
      });
      if (nameSet.size == 0) {
        this.product.name = null;
        this.alertText = "감지된 음료가 없습니다.";
        this.searchAlertText = "탐색 음료가 존재하지 않습니다.";
        if (this.path == "search") this.tts(this.searchAlertText);
        else this.tts(this.alertText);
      } else {
        nameSet.forEach((item) => {
          this.product.name = classesDir[item].name;
          this.product.type = classesDir[item].type;
          if (
            this.product.name == this.searchName &&
            this.product.type == this.searchType
          ) {
            if (nameSet.size == 1) {
              this.searchAlertText = "탐색 음료 입니다!";
              this.tts(this.searchAlertText);
            } else {
              this.searchAlertText =
                "탐색 음료가 존재합니다.\n더 가까이 가주세요.";
              this.tts("탐색 음료가 존재합니다. 더 가까이 가주세요.");
            }
          }
        });
        if (nameSet.size > 1 && this.path != "search") {
          this.product.name = null;
          this.alertText = "더 가까이 가주세요.";
          this.tts(this.alertText);
        } else if (nameSet.size == 1 && this.path != "search") {
          this.tts(this.product.name + " " + this.product.type);
        }
      }
      cntMap.clear();
      nameSet.clear();
      this.count = 0;
    },
    // 모델 실시간 감지
    detectFrame(video, model) {
      if (this.getIsDetect) {
        return;
      }
      tf.engine().startScope();
      this.model.executeAsync(this.process_input(video)).then((predictions) => {
        this.renderPredictions(predictions, video);
        requestAnimationFrame(() => {
          this.detectFrame(video, model);
        });
        tf.engine().endScope();
      });
    },
    // 모델 로딩 및 감지 시작
    loadModelAndStartDetecting() {
      this.modelPromise = this.loadModel();
      Promise.all([this.modelPromise, this.streamPromise])
        .then((values) => {
          this.isLoading = false;

          this.detectFrame(this.video, values[0]);
          this.startTimer();
        })
        .catch((error) => {
          this.initFailMessage = error;
        });
    },
    // 비디오 프레임 확장
    process_input(video_frame) {
      const tfimg = tf.browser.fromPixels(video_frame).toInt();
      const expandedimg = tfimg.transpose([0, 1, 2]).expandDims();
      return expandedimg;
    },
    // 모델 예측
    renderPredictions(predictions) {
      const classes = [];
      predictions[4].dataSync().forEach((element) => {
        classes.push(Math.round(element));
      });
      const boxes = predictions[0].arraySync();
      const scores = predictions[2].arraySync();
      this.buildDetectedObjects(scores, threshold, boxes, classes);
    },
    // 모델 예측 결과
    buildDetectedObjects(scores, threshold, boxes, classes) {
      scores[0].forEach((score, i) => {
        if (score > threshold) {
          this.count++;
          let index = classes[i];
          if (cntMap.has(index)) {
            cntMap.set(index, cntMap.get(index) + 1);
          } else cntMap.set(index, 1);
        }
      });
    },
    // 페이지 이동 시 카메라 종료
    stopCameraStream() {
      if (this.$refs.camera.srcObject) {
        let tracks = this.$refs.camera.srcObject.getTracks();
        tracks.forEach((track) => {
          track.stop();
        });
      }
    },
    startTimer() {
      this.timer = setInterval(this.initMap, 1000);
    },
  },
  async created() {
    await this.storeIsDetect(true);
    this.isLoading = true;
    this.isIOS = false;
    if (this.path == "search") {
      if (this.$route.params.name == null) {
        this.$router.push("/search");
      }
      this.searchName = this.$route.params.name;
      this.searchType = this.$route.params.type;
    }
    setTimeout(() => {
      this.storeIsDetect(false);
      this.streamPromise = this.initWebcamStream();
      this.loadModelAndStartDetecting();
    }, 1000);
  },
  destroyed() {
    clearInterval(this.timer);
  },
};
</script>
