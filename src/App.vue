<template>
  <div v-if="showCamera" class="modal d-flex" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered modal-xl">
      <div class="modal-content">
        <div class="modal-header p-1 px-3">
          <h5 class="modal-title">Preview Result</h5>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"
            @click="showCamera = false"></button>
        </div>
        <div class="modal-body p-0" style="position: relative;">
          <camera :resolution="{ width: CAMERA_WIDTH, height: CAMERA_HEIGHT }" ref="cameraRef" autoplay
            @started="previewCameraHandle" @stopped="stopPreview">
          </camera>
          <canvas ref="canvas" :width="CAMERA_WIDTH" :height="CAMERA_HEIGHT"
            style="position: absolute; top: 0; left: 0;"></canvas>
        </div>
      </div>
    </div>
  </div>

  <div class="container-fluid d-flex flex-row justify-content-around align-items-center"
  style="overflow: auto; min-height: 100vh; background-color: var(--color-background); padding: 50px;">
  <!-- CLASS ITEM -->
    <div class="d-flex flex-column" style="width: 45%;">
      <ClassCard :name="classItem.name" :images="classItem.images"
        :disable="classItem.isDisable" class="mb-3" @update:name="(name) => classItem.name = name"
        @preview="showCamera = true" @remove="removeClass(index)"
        @disable="classItem.isDisable = true" @enable="classItem.isDisable = false"></ClassCard>
    </div>

    <!-- TRAINING -->
    <div class="card shadow" id="train-card">
      <div class="card-header d-flex flex-row justify-content-between align-items-center">
        <span class="fw-bold">Training</span>
      </div>
      <div class="card-body d-flex flex-column p-3 text-center align-items-center">
        <button v-if="!isTraining" type="button" class="btn btn-outline-secondary fw-semibold"
          @click="trainModelHandle">Send images</button>
        <div v-else class="spinner-border" role="status">
          <span class="visually-hidden">Loading...</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import ClassCard from './components/ClassCard.vue';
import Camera from 'simple-vue-camera';

import {useToast} from 'vue-toast-notification';
import 'vue-toast-notification/dist/theme-sugar.css';
const $toast = useToast();

import successSound from './assets/sounds/success.mp3'
const successAudio = new Audio(successSound)

const showCamera = ref(false);
const cameraRef = ref(null);
const CAMERA_WIDTH = '400';
const CAMERA_HEIGHT = '400';

const canvas = ref(null);
const resultMessage = ref([]);

const isTraining = ref(false);

const classItem = ref(
  {
    name: 'Student 123',
    images: [],
    isDisable: false,
  }
);

const trainModelHandle = async () => {
  isTraining.value = true;
  try {
    let promises = [];
    if (classItem.value.images.length > 0) {
      let imgFormData = new FormData();
      classItem.value.images.map(async (image) => {
        imgFormData.append('files', image);
      })
      const config = {
        method: 'POST',
        mode: "cors",
        body: imgFormData
      };
     await fetch(`https://86db-203-205-51-20.ngrok-free.app/upload/?folder_name=${classItem.value.name}`, config);
    }
  } finally {
    isTraining.value = false;
  }
}

let previewInterval;
const previewCameraHandle = async () => {
  previewInterval = setInterval(async () => {
    const image = await cameraRef.value?.snapshot();

    let imgFormData = new FormData();
    imgFormData.append('file', image);
    const config = {
      method: 'POST',
      mode: "cors",
      body: imgFormData
    };
    const response = await fetch('https://86db-203-205-51-20.ngrok-free.app/detect', config);
    const jsonResponse = await response.json();


    const frame = jsonResponse.data.image_box;
    // Accessing the canvas 2D context
    const ctx = canvas.value?.getContext('2d');
    ctx.clearRect(0, 0, CAMERA_WIDTH, CAMERA_HEIGHT);
    // Draw a rectangle
    ctx.strokeStyle = 'red';
    ctx.strokeRect(frame[0], frame[1], frame[2], frame[3]); // Rectangle position (x, y) and dimensions (width, height)

    const resultText = jsonResponse.data.name;

    // Write text above the rectangle
    ctx.fillStyle = 'red';
    ctx.font = '14px Arial';
    const textWidth = ctx.measureText(resultText).width;
    const textX = frame[0] + (frame[2] - textWidth) / 2; // Center text horizontally
    const textY = frame[1] - 10; // Place text 10 pixels above the rectangle
    ctx.fillText(resultText, textX, textY);


    if (jsonResponse.data.name != 'UNKNOWN' && !jsonResponse.data?.name?.includes('FakeFace')) {
      if (!resultMessage.value.includes(resultText)) {
        resultMessage.value.push(resultText);
        $toast.success(resultText, {position: 'bottom'});
        successAudio.play();
      }
    }
  }, 500)
}

const stopPreview = () => {
  clearInterval(previewInterval);
}

</script>

<style lang="scss" scoped>
.center-vertical {
  position: fixed;
  top: 50%;
  transform: translate(0px, -50%);
}
</style>
