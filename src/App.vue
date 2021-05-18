<template>
  <a-layout class="container">
    <a-layout-header
      :style="`background: #fff;padding: 0!important;line-height: ${headerHeight}px;z-index:233`"
    >
      <a-row class="header" type="flex" justify="space-between">
        <a-col :span="8">
          <h1 class="title">WFQ Simulator</h1>
        </a-col>
      </a-row>
    </a-layout-header>
    <a-layout-content
      :style="`background: #fff;overflow: auto;padding: 20px 50px`"
    >
      <a-card style="margin: 20px 0" :hoverable="true">
        <template v-slot:title>
          <span class="card-title">
            <play-square-outlined style="margin-right: 5px"/>
            Animation
          </span>
        </template>
        <animation-card ref="animationCard"></animation-card>
      </a-card>
    </a-layout-content>
  </a-layout>
</template>

<script>
import {PlaySquareOutlined} from "@ant-design/icons-vue";
import AnimationCard from "@/components/AnimationCard";
export default {
  name: "App",
  components: {
    AnimationCard, 
    PlaySquareOutlined,
  },
  data() {
    return {
      headerHeight: 64,
    };
  },
  methods: {
  
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  height: 100%;
  overflow: hidden;
  background: #fff;
}
.container {
  background: #fff;
  height: 100%;
  overflow: hidden;
}
.header {
  width: 100vw;
  box-shadow: 0 2px 8px #e0e0e0;
  padding: 0 8%;
  font-size: 22px;
}
.title {
  margin: 0;
  font-size: 34px;
  font-weight: bold;
}
.card-title {
  font-size: 18px;
}
</style>
