<template>
  <v-app>
    <v-app-bar app color="primary" dark flat>
      <div class="toggle">
        <v-btn v-if="!mute" @click="changeMute()" aria-label="음성 안내 켜기">
          <v-icon color="accent">mdi-volume-off</v-icon>
          <span>OFF</span>
        </v-btn>
        <v-btn v-else @click="changeMute()" aria-label="음성 안내 끄기">
          <v-icon color="accent">mdi-volume-high</v-icon>
          <span>ON</span>
        </v-btn>
      </div>
      <v-toolbar-title class="mx-auto" tabindex="0">
        {{ value }}
      </v-toolbar-title>
    </v-app-bar>

    <v-main>
      <router-view />
    </v-main>

    <v-bottom-navigation
      app
      v-model="value"
      class="primary"
      active-class="accent primary--text font-weight-bold"
      grow
      dark
    >
      <v-btn value="스캔모드" to="/scan" aria-label="스캔모드" class="navi">
        <span>스캔모드</span>
        <v-icon>mdi-cube-scan</v-icon>
      </v-btn>

      <v-btn value="탐색모드" to="/search" aria-label="탐색모드" class="navi">
        <span>탐색모드</span>
        <v-icon>mdi-magnify</v-icon>
      </v-btn>

      <v-btn
        value="즐겨찾기"
        to="/favorites"
        aria-label="즐겨찾기"
        class="navi"
      >
        <span>즐겨찾기</span>
        <v-icon>mdi-star-outline</v-icon>
      </v-btn>

      <v-btn value="사용법" to="/guide" aria-label="사용법" class="navi">
        <span>사용법</span>
        <v-icon>mdi-information-outline</v-icon>
      </v-btn>

      <v-btn disabled v-show="false" value="404"></v-btn>
    </v-bottom-navigation>
  </v-app>
</template>

<script>
import { mapActions } from "vuex";

export default {
  name: "App",

  data() {
    return {
      value: "스캔모드",
      mute: true,
    };
  },
  methods: {
    ...mapActions(["storeMute"]),
    changeMute() {
      if (this.mute) this.mute = false;
      else this.mute = true;
      this.storeMute(this.mute);
    },
  },
  created() {
    let path = this.$route.path;
    if (path.includes("404")) this.value = "404";
    if (path.includes("scan")) this.value = "스캔모드";
    if (path.includes("search")) this.value = "탐색모드";
    if (path.includes("favorites")) this.value = "즐겨찾기";
    if (path.includes("guide")) this.value = "사용법";
  },
};
</script>

<style src="@/assets/css/main.css" />
<style>
@import url(//fonts.googleapis.com/earlyaccess/jejugothic.css);
* {
  font-family: "Jeju Gothic";
}
.toggle {
  float: left;
  position: absolute;
}
.item-title {
  font-size: 1.25em !important;
}
</style>
