<template>
  <div class="mask" v-show="props.showPopup" @click="emits('close')"></div>
  <div class="popup" v-show="props.showPopup">
    <div class="title" @click="emits('close')">click to close</div>
    <div id="wrapper">
      <div id="mg" class="mg" style="width: 400px; height: 400px;"><canvas width="400" height="400"></canvas><img src="/src/assets/finish.gif" class="finish-img"><div class="me" classname="me" style="width: 20px; height: 20px; top: 140px; left: 20px;"><div class="inform" classname="inform" style="display: block;"><p>Hello?</p></div><img src="/src/assets/me.gif"></div></div>
      <div id="mg_set">
        <strong>设置：</strong>
        宽：<input type="text" id="mg_width" value="10" size="4" maxlength="3">
        高：<input type="text" id="mg_height" value="10" size="4" maxlength="3">
        <input type="button" value=" 生成新迷宫 " @click="new_mg">
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted } from 'vue';
// @ts-ignore
import { MG } from '../utils/mg'
let mg: any = null;
const emits = defineEmits(['close']);
const props = defineProps({
  showPopup: {
    type: Boolean,
    default: false
  }
});
function new_mg() {
    let w = parseInt((document.getElementById("mg_width") as any).value) || 20,
        h = parseInt((document.getElementById("mg_height") as any).value) || 20;
    mg.set({width: w, height: h}).create().show();
    (document.getElementById("mg_width") as any).value = w;
    (document.getElementById("mg_height") as any).value = h;
}

onMounted(() => {
  mg = new MG('mg');
  new_mg();
});
</script>

<style>
.popup {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 10;
}
.title {
  background: #FFF;
  z-index: 10;
}
.mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,.3);
  z-index: 10;
}
div {
  color: #000;
}
</style>