
<template>
  <mazePopup :showPopup="showPopup" @close="showPopup = false"/>
  <div class="content">
    <div class="options">
      <button type="button" @click="train">准备</button>
      <span>损失：{{ logLoss }}</span>
      <button type="button" @click="predict">开始</button>
      <button type="button" @click="showPopup = true">game?</button>
    </div>
    <div class="video-img">
      <div class="self-video">
        <video autoplay playsinline muted id="webcam" width="224" height="224"></video>
        <div>
          <span>class num:</span>
          <input v-model.number="classNum" type="number">
        </div>
        <div>
          <span>学习率：</span>
          <select v-model="studyRate">
            <option :value="0.00001">0.00001</option>
            <option :value="0.0001">0.0001</option>
            <option :value="0.001">0.001</option>
            <option :value="0.003">0.003</option>
          </select>
        </div>
        <div>
          <span>Batch size: </span>
          <select v-model="batchSize">
            <option :value="0.05">0.05</option>
            <option :value="0.1">0.1</option>
            <option :value="0.4">0.4</option>
            <option :value="1">1</option>
          </select>
        </div>
        <div>
          <span>Epochs: </span>
          <select v-model="epochs">
            <option :value="10">10</option>
            <option :value="20">20</option>
            <option :value="40">40</option>
          </select>
        </div>
        <div>
          <span>Hidden units:</span>
          <select v-model="units">
            <option :value="10">10</option>
            <option :value="100">100</option>
            <option :value="200">200</option>
          </select>
        </div>
      </div>
      <div class="img-datas" v-if="showResult">
          <div
            :class="index === classIdResult && isPredicting ? 'right-card' : 'not-card'"
            v-for="(label, index) in controls"
            :key="label"
          >
            <div class="result">
              <canvas :id="label + '-thumb'" width="224" height="224"></canvas>
              <button type="button" @mousedown="handlerDown(label)" @mouseup="handlerUp(label)">
                {{ label }}: {{ imgCounts[index] }}
              </button>
            </div>
          </div>
          <i class="placeholder-element" key="placeholder-element1"></i>
          <i class="placeholder-element" key="placeholder-element2"></i>
          <i class="placeholder-element" key="placeholder-element3"></i>
          <i class="placeholder-element" key="placeholder-element4"></i>
          <i class="placeholder-element" key="placeholder-element5"></i>
          <i class="placeholder-element" key="placeholder-element6"></i>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, watch } from 'vue';
import * as tf from '@tensorflow/tfjs';
import * as tfd from '@tensorflow/tfjs-data';
// @ts-ignore
import { ControllerDataset } from '../utils/controller_dataset';
// @ts-ignore
import { drawThumb } from '../utils/ui';
import mazePopup from './mazePopUp.vue';
import { throttle } from 'lodash';
// 类型初值
const logLoss = ref('');
const controls = ref(['1', '2', '3', '4']);
const classNum = ref(4);
const isPredicting = ref(false);
let controllerDataset = new ControllerDataset(classNum.value);
let truncatedMobileNet: any = null;
let model: any = null;
let webcam: any = null;
const isAddingImg = ref(false);
const imgCounts = ref([0, 0, 0, 0]);
const classIdResult = ref(-1);
const showResult = ref(true);
// 超参数
const studyRate = ref(0.0001);
const batchSize = ref(0.4);
const epochs = ref(20);
const units = ref(100);
const showPopup = ref(false);
const channel = new BroadcastChannel("channel-BroadcastChannel");
watch(
  classNum,
  (val) => {
    reInit(val);
  }
);
function reInit(val: number) {
  showResult.value = false;
  if (val < 2) {
    classNum.value = 2;
  }
  if (val > 20) {
    classNum.value = 20;
  }
  controllerDataset = new ControllerDataset(classNum.value);
  isAddingImg.value = false;
  isPredicting.value = false;
  imgCounts.value = new Array(classNum.value).fill(0);
  classIdResult.value = -1;
  controls.value.forEach((label) => {
    const canvas: any = document.getElementById(label + '-thumb');
    if (canvas) {
      canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
    }
  });
  controls.value = new Array(classNum.value).fill('').map((_, i) => `${i + 1}`);
  console.log('reInit', imgCounts.value, controls.value);
  showResult.value = true;
}

async function loadTruncatedMobileNet() {
  // 加载模型
  const mobilenet = await tf.loadLayersModel(
      'https://storage.googleapis.com/tfjs-models/tfjs/mobilenet_v1_0.25_224/model.json');

  // Return a model that outputs an internal activation.
  const layer = mobilenet.getLayer('conv_pw_13_relu');
  return tf.model({inputs: mobilenet.inputs, outputs: layer.output});
}
async function addExampleHandler(label: string, index: number) {
    let img = await getImage();
    controllerDataset.addExample(truncatedMobileNet.predict(img), index);
    // Draw the preview thumbnail.
    drawThumb(img, label);
    img.dispose();
}
async function handlerDown(label: string) {
  // 一直按着按钮，就一直增加训练张数
  isAddingImg.value = true;
  let count = 0;
  const index = controls.value.indexOf(label);
  while (isAddingImg.value && count < 100) {
    addExampleHandler(label, index);
    await tf.nextFrame();
    count++;
    imgCounts.value[index]++;
  }
}
async function train() {
  isPredicting.value = false;
  await tf.nextFrame();
  await tf.nextFrame();
  // isPredicting = false;
  if (controllerDataset.xs == null) {
    throw new Error('Add some examples before training!');
  }
  // Creates a 2-layer fully connected model. By creating a separate model,
  // rather than adding layers to the mobilenet model, we "freeze" the weights
  // of the mobilenet model, and only train weights from the new model.
  model = tf.sequential({
    layers: [
      // Flattens the input to a vector so we can use it in a dense layer. While
      // technically a layer, this only performs a reshape (and has no training
      // parameters).
      tf.layers.flatten(
          {inputShape: truncatedMobileNet.outputs[0].shape.slice(1)}),
      // Layer 1.
      // units 先写100，后面改成可选，10，100，200
      tf.layers.dense({
        units: units.value,
        activation: 'relu',
        kernelInitializer: 'varianceScaling',
        useBias: true
      }),
      // Layer 2. The number of units of the last layer should correspond
      // to the number of classes we want to predict.
      tf.layers.dense({
        units: classNum.value,
        kernelInitializer: 'varianceScaling',
        useBias: false,
        activation: 'softmax'
      })
    ]
  });
  // Creates the optimizers which drives training of the model
  // TODO： 学习率
  const optimizer = tf.train.adam(studyRate.value);
  // We use categoricalCrossentropy which is the loss function we use for
  // categorical classification which measures the error between our predicted
  // probability distribution over classes (probability that an input is of each
  // class), versus the label (100% probability in the true class)>
  model.compile({optimizer: optimizer, loss: 'categoricalCrossentropy'});

  // We parameterize batch size as a fraction of the entire dataset because the
  // number of examples that are collected depends on how many examples the user
  // collects. This allows us to have a flexible batch size.
  const BatchSize =
      Math.floor(controllerDataset.xs.shape[0] * batchSize.value);
  if (!(BatchSize > 0)) {
    throw new Error(
        `Batch size is 0 or NaN. Please choose a non-zero fraction.`);
  }

  // Train the model! Model.fit() will shuffle xs & ys so we don't have to.
  model.fit(controllerDataset.xs, controllerDataset.ys, {
    batchSize: BatchSize,
    epochs: epochs.value,
    callbacks: {
      onBatchEnd: async (batch: any, logs: any) => {
        logLoss.value = logs.loss.toFixed(5);
        console.log('batch', batch);
        // ui.trainStatus('Loss: ' + logs.loss.toFixed(5));
      }
    }
  });
}
const noticeClass = throttle((classId: string | number) => {
  channel.postMessage(classId);
}, 500);
async function predict() {
  isPredicting.value = true;
  while (isPredicting) {
    // Capture the frame from the webcam.
    const img = await getImage();

    // Make a prediction through mobilenet, getting the internal activation of
    // the mobilenet model, i.e., "embeddings" of the input images.
    const embeddings = truncatedMobileNet.predict(img);

    // Make a prediction through our newly-trained model using the embeddings
    // from mobilenet as input.
    const predictions = model.predict(embeddings);

    // Returns the index with the maximum probability. This number corresponds
    // to the class the model thinks is the most probable given the input.
    const predictedClass = predictions.as1D().argMax();
    const classId = (await predictedClass.data())[0];
    if (showPopup.value && Number(classId) < 4) {
      noticeClass(classId);
    }
    img.dispose();
    classIdResult.value = classId;
    await tf.nextFrame();
  }
}

async function handlerUp(label: string) {
  isAddingImg.value = false;
  console.log('handlerUp', label);
}
async function getImage() {
  const img = await webcam.capture();
  const processedImg =
      tf.tidy(() => img.expandDims(0).toFloat().div(127).sub(1));
  img.dispose();
  return processedImg;
}
const testObj = {};
const refObj = ref(testObj);
onMounted(async () => {
  console.log(refObj.value, testObj, testObj === refObj.value);
  console.log(controllerDataset);
  tf.setBackend('webgl');
  try {
    const webCamDom: any = document.getElementById('webcam');
    webcam = await tfd.webcam(webCamDom);
    await webcam.setup();
    console.log('webcam', webcam);
  } catch (e) {
    console.log(e);
  }
  truncatedMobileNet = await loadTruncatedMobileNet();
  const screenShot = await webcam.capture();
  console.log('truncatedMobileNet', truncatedMobileNet, screenShot);
  // 使用 await 等待 predict 操作完成
  let newScreenShot = await screenShot.expandDims(0);
  // newScreenShot.dataId = newScreenShot.dataId.id;
  console.log('newScreenShot', newScreenShot.dataId);
  const prediction = await truncatedMobileNet.predict(newScreenShot);
  // 处理 prediction 结果
  console.log('Prediction Result:', prediction.dataSync());
  // 释放截图资源
  screenShot.dispose();
  console.log('screenShot', screenShot);
});
</script>

<style scoped lang="scss">
.content {
  display: grid;
  grid-template-rows: 10% 90%;
  gap: 10px;
  width: 100vw;
  height: 100vh;
  .options {
    grid-row-start: 1;
    display: flex;
    justify-content: space-around;
    width: 100%;
    position: fixed;
    top: 0;
  }
  .video-img {
    display: grid;
    grid-row-start: 2;
    grid-template-columns: 20% 80%;
    .self-video {
      grid-column-start: 1;
      width: 224px;
      height: 224px;
    }
  }
  .img-datas {
    grid-column-start: 2;
    display: flex;
    flex-flow: wrap;
    justify-content: space-around;
    width: 100%;
    height: 100%;
    overflow-y: auto;
    overflow-x: hidden;
    .result {
      width: 150px;
      height: 150px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 10px;
      justify-content: center;
    }
  }
}
.right-card {
  padding: 6px;
  transform: scale(.5);
  background-color: greenyellow;
}
.not-card {
  padding: 6px;
  transform: scale(.5);
}
.read-the-docs {
  color: #888;
}
#webcam {
  transform: scaleX(-1);
}
canvas {
  width: 224px;
  height: 224px;
  transform: scaleX(-1);
}
.placeholder-element {
  width: 162px;
}
span {
  color: #000;
}
</style>