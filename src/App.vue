<template>
  <div v-if="showCamera" class="modal d-flex" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered modal-xl">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title">Preview Result</h5>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"
            @click="showCamera = false"></button>
        </div>
        <div class="modal-body p-0" style="position: relative;">
          <camera :resolution="{ width: CAMERA_WIDTH, height: CAMERA_WIDTH }" ref="cameraRef" autoplay @started="previewCameraHandle"
            @stopped="stopPreview">
          </camera>
          <canvas ref="canvas" :width="CAMERA_WIDTH" :height="CAMERA_WIDTH" style="position: absolute; top: 0; left: 0;"></canvas>
        </div>
        <div class="modal-footer">
          <p>{{ resultMessage }}</p>
        </div>
      </div>
    </div>
  </div>
  <!-- <div v-if="showCamera" class="modal" id="exampleModalToggle" aria-hidden="true"
    aria-labelledby="exampleModalToggleLabel" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="exampleModalToggleLabel">Modal 1</h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">
          <button type="button" class="btn-close" aria-label="Close" @click="showCamera = false"></button>
          <camera :resolution="{ width: 80, height: 80 }" ref="cameraRef" autoplay @started="previewCameraHandle"
            @stopped="stopPreview"></camera>
          <button class="btn btn-primary" @mousedown="startRecord" @mouseup="stopRecord">Hold to record</button>
        </div>
        <div class="modal-footer">
          <button class="btn btn-primary" data-bs-target="#exampleModalToggle2" data-bs-toggle="modal">Open second
            modal</button>
        </div>
      </div>
    </div>
  </div> -->
  <!-- <div v-if="showCamera" class="modal-dialog modal-dialog-centered d-flex flex-column p-2">
    <button type="button" class="btn-close" aria-label="Close" @click="showCamera = false"></button>
    <camera :resolution="{ width: 80, height: 80 }" ref="cameraRef" autoplay @started="previewCameraHandle"
      @stopped="stopPreview"></camera>
    <button class="btn btn-primary" @mousedown="startRecord" @mouseup="stopRecord">Hold to record</button>
  </div> -->
  <div class="container-fluid d-flex flex-row justify-content-between"
    style="overflow: auto; min-height: 100vh; background-color: var(--color-background);">
    <div class="d-flex flex-column p-5">
      <ClassCard v-for="(n, index) in classList" :key=index :id="`class-card-${index}`" :name="n.name" :images="n.images"
        :disable="n.isDisable" class="mb-3" @remove="removeClass(index)" @disable="n.isDisable = true"
        @enable="n.isDisable = false"></ClassCard>
      <button class="btn-add fw-semibold" @click="addClass">
        Add a class
      </button>
    </div>
    <div class="center-vertical" style="left: 750px">
      <div class="card shadow" style="width: 150px" id="train-card">
        <div class="d-flex flex-column p-3 text-center">
          <p class="fw-bold">Training</p>
          <button type="button" class="btn btn-outline-secondary fw-semibold" @click="trainModelHandle">Train
            model</button>
        </div>
      </div>
    </div>
    <div class="center-vertical" style="left: 1050px">
      <div class="card shadow" style="width: 300px" id="preview-card">
        <div class="card-header d-flex flex-row justify-content-between  align-items-center">
          <span class="fw-bold">Preview</span>
          <button v-if="!showCamera" type="button" class="btn btn-outline-secondary fw-semibold"
            @click="showCamera = true">Export model</button>
        </div>
        <div class="card-body p-0">
          <p class="p-2">You must train a model on the left before you can preview it here.</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUpdated } from 'vue';
import ClassCard from './components/ClassCard.vue';
import LeaderLine from 'leader-line-new';
import Camera from 'simple-vue-camera';

const showCamera = ref(false);
const cameraRef = ref(null);
const CAMERA_WIDTH = '600';
const CAMERA_HEIGHT = '600';

const canvas = ref(null);
const resultMessage = ref('');

const classList = ref([
  {
    name: 'Class 1',
    images: [],
    isDisable: false,
  },
  {
    name: 'Class 2',
    images: [],
    isDisable: false,
  },
]);

const addClass = () => {
  classList.value.push({
    name: `Class ${classList.value.length + 1}`,
    images: [],
    isDisable: false
  })
}

const removeClass = (index) => {
  classList.value.splice(index, 1);
}

let lines = [];
const drawCurvedLines = () => {
  const parentRect = document.getElementById('train-card');
  classList.value.map((item, index) => {
    const childRect = document.getElementById(`class-card-${index}`)
    const line = new LeaderLine(childRect, parentRect, {
      color: '#708292',
      size: 1,
      path: 'fluid',
    });
    lines.push(line);
  })

  const previewRect = document.getElementById('preview-card');
  lines.push(new LeaderLine(parentRect, previewRect, {
    color: '#708292',
    size: 1,
  }))
};

const trainModelHandle = () => {
  console.log(classList.value);
}

let previewInterval;
const previewCameraHandle = async () => {
  previewInterval = setInterval(async () => {
    const image = await cameraRef.value?.snapshot();

    let imgFormData = new FormData();
    imgFormData.append('image', image);
    const config = {
      method: 'POST',
      mode: "cors",
      body: imgFormData
    };
    const response = await fetch('http://translator.paraceltech.com/api/detect', config);
    const jsonResponse = await response.json();
    resultMessage.value = jsonResponse.mss.result_text;

    const frame = jsonResponse.mss.image_bbox;
    // Accessing the canvas 2D context
    const ctx = canvas.value.getContext('2d');
    ctx.clearRect(0, 0, CAMERA_WIDTH, CAMERA_HEIGHT);
    // Draw a rectangle
    ctx.strokeStyle = 'red';
    ctx.strokeRect(frame[0], frame[1], frame[2], frame[3]); // Rectangle position (x, y) and dimensions (width, height)
  }, 500)
}

const stopPreview = () => {
  clearInterval(previewInterval);
}

onMounted(() => {
  drawCurvedLines();
  window.addEventListener('scroll', () => {
    lines.forEach(line => line.position());
  })
})

onUpdated(() => {
  lines.forEach(line => line.remove());
  lines = [];
  drawCurvedLines();
});
</script>

<style lang="scss" scoped>
.center-vertical {
  position: fixed;
  top: 50%;
  transform: translate(0px, -50%);
}
</style>
