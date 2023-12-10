<template>
  <div class="card shadow class-card" :class="{'card-disabled shadow-none': disable}" style="width: 600px;">
    <div class="card-header d-flex justify-content-between align-items-center">
      <span class="fw-bold">{{ name }}</span>
      <div class="menu-btn" ref="menuRef">
        <button class="btn" @click="showMenu = !showMenu">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="#01ACB4"
            class="bi bi-three-dots-vertical" viewBox="0 0 16 16">
            <path
              d="M9.5 13a1.5 1.5 0 1 1-3 0 1.5 1.5 0 0 1 3 0m0-5a1.5 1.5 0 1 1-3 0 1.5 1.5 0 0 1 3 0m0-5a1.5 1.5 0 1 1-3 0 1.5 1.5 0 0 1 3 0" />
          </svg>
        </button>
        <div v-if="showMenu" class="list-group menu-list shadow">
          <button class="list-group-item list-group-item-action" @click="showMenu = false;emit('remove')">Remove sample</button>
          <button class="list-group-item list-group-item-action" @click="showMenu = false;emit('disable')">Disable sample</button>
          <button class="list-group-item list-group-item-action" @click="showMenu = false;emit('enable')">Enable sample</button>
        </div>
      </div>
    </div>
    <div class="card-body p-0">
      <div v-if="!showWebcam" class="p-3 fw-semibold">
        <p v-if="images.length == 0">Add image sample:</p>
        <p v-else>{{images.length}} image samples</p>
        <div class="d-flex flex-row">
          <button type="button" class="btn btn-primary text-white me-2 fw-semibold p-0 flex-shrink-0" style="width: 80px; height: 80px; font-size: 14px;"
            @click="showWebcam = true">Webcam</button>
          <button type="button" class="btn btn-primary text-white fw-semibold p-0 me-2 flex-shrink-0" @click="imageUploadRef.click()"
            style="width: 80px; height: 80px; font-size: 14px;">Upload</button>
          <input ref="imageUploadRef" type="file" accept="image/*" multiple style="display: none;" @change="onFileChange">
          <div class="d-flex flex-row" style="overflow-x: auto;">
            <img v-for="image in imageUrls" :src="image" class="me-2 rounded-2" style="width: 80px; height: 80px; object-fit: contain;">
          </div>
        </div>
      </div>
      <div v-else class="d-flex flex-row" style="height: 380px;">
        <div class="d-flex flex-column p-2" style="flex: 1; background-color: #c2e4e5;">
          <button type="button" class="btn-close" aria-label="Close" @click="showWebcam = false"></button>
          <camera :resolution="{ width: 80, height: 80 }" ref="cameraRef" autoplay></camera>
          <button class="btn btn-primary" @mousedown="startRecord" @mouseup="stopRecord">Hold to record</button>
        </div>
        <div class="d-flex flex-column p-2" style="flex: 1;">
          <p v-if="images.length == 0">Add image sample:</p>
          <p v-else>{{images.length}} image samples</p>
          <div class="row g-2" style=" overflow-y: auto;">
            <img v-for="image in imageUrls" :src="image" class="col-3 rounded-2"
              style="object-fit: contain;">
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup>
import { ref, computed } from 'vue';
import Camera from 'simple-vue-camera';
import detectOutsideClick from '../utils/detectOutsideClick'

const props = defineProps({
  name: {
    type: String,
    required: true
  },
  images: {
    type: Array,
    default: []
  },
  disable: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['remove', 'disable', 'enable', 'update:images'])

const showMenu = ref(false);
const showWebcam = ref(false);
const imageUploadRef = ref(null);
const cameraRef = ref(null);
const imageUrls = ref([]);

const images = computed({
  get() {
    return props.images;
  },

  set(value) {
    emit('update:images', value);
  }
});

const onFileChange = (e) => {
  const files = e.target.files;
  for (let index = 0; index < files.length; index++) {
    images.value.push(files[index]);
    imageUrls.value.push(URL.createObjectURL(files[index]));
  }
  console.log(images.value);
  console.log(imageUrls.value);
}

let recordInterval;
const startRecord = async () => {
  recordInterval = setInterval(async()=>{
    const value = await cameraRef.value.snapshot();
    images.value.push(value);
    imageUrls.value.push(URL.createObjectURL(value));
  }, 100)
}
const stopRecord = ()=> {
  console.log('stop');
  clearInterval(recordInterval)
}

const menuRef = ref(null);
detectOutsideClick(menuRef, ()=>{
  showMenu.value = false;
})
</script>
<style lang="scss">
.menu-btn {
  position: relative;
}

.menu-list {
  position: absolute;
  right: 0;
  width: fit-content;
  white-space: nowrap;
}
</style>