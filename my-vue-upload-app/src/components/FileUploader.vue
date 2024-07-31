<template>
  <div class="upload-container">
    <div 
      class="drop-zone" 
      @dragover.prevent 
      @drop.prevent="handleDrop"
      :class="{ 'drag-over': isDragging }"
      @dragenter.prevent="isDragging = true"
      @dragleave.prevent="isDragging = false"
    >
      ここに画像をドラッグ&ドロップしてアップロード
      <input 
        type="file" 
        @change="handleFileChange" 
        ref="fileInput" 
        multiple 
        accept=".jpg,.jpeg,.png,.pdf"
        style="display: none;"
      >
    </div>
    <div class="preview-controls">
      <button @click="zoomIn">ズームイン</button>
      <button @click="zoomOut">ズームアウト</button>
      <span>ズーム: {{ zoomLevel }}%</span>
    </div>
    <div class="preview-area">
      <div class="preview-scroll">
        <div v-for="file in validFiles" :key="file.name" class="preview-item">
          <img 
            v-if="file.type.startsWith('image/')" 
            :src="file.preview" 
            class="preview" 
            :style="{ transform: `scale(${zoomLevel / 100})` }"
          />
          <div class="file-info">
            {{ file.name }} - {{ formatFileSize(file.size) }}
          </div>
        </div>
      </div>
    </div>
    <div class="button-group">
      <button @click="$refs.fileInput.click()">ファイルを選択</button>
      <button @click="uploadFiles" :disabled="!isValid">アップロード</button>
    </div>
    <div>最大ファイルサイズ: {{ formatFileSize(MAX_FILE_SIZE) }}</div>
    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
  </div>
</template>


<script setup>
import { ref, computed } from 'vue';

const fileInput = ref(null);
const validFiles = ref([]);
const errorMessage = ref('');
const isDragging = ref(false);

const MAX_FILE_SIZE = 1 * 1024 * 1024; // 1MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];
const MIN_IMAGE_WIDTH = 100;
const MAX_IMAGE_WIDTH = 4000;
const MIN_IMAGE_HEIGHT = 100;
const MAX_IMAGE_HEIGHT = 4000;

const isValid = computed(() => validFiles.value.length > 0 && !errorMessage.value);

async function handleFileChange(event) {
  const files = Array.from(event.target.files);
  await processFiles(files);
}

async function handleDrop(event) {
  isDragging.value = false;
  const files = Array.from(event.dataTransfer.files);
  await processFiles(files);
}

async function processFiles(files) {
  validFiles.value = [];
  errorMessage.value = '';

  for (const file of files) {
    if (await validateFile(file)) {
      validFiles.value.push(file);
    }
  }
}

async function validateFile(file) {
  // ファイルサイズのチェック
  if (file.size > MAX_FILE_SIZE) {
    errorMessage.value = `${file.name} は大きすぎます。${formatFileSize(MAX_FILE_SIZE)}以下のファイルを選択してください。現在のファイルサイズ: ${formatFileSize(file.size)}`;
    return false;
  }

  if (!ALLOWED_TYPES.includes(file.type)) {
    errorMessage.value = `${file.name} は許可されていない形式です。JPG、PNG、PDFのみアップロード可能です。`;
    return false;
  }

  if (file.type.startsWith('image/')) {
    const imgValidation = await validateImage(file);
    if (!imgValidation.valid) {
      errorMessage.value = imgValidation.error;
      return false;
    }
  } else if (file.type === 'application/pdf') {
    const pdfValidation = await validatePDF(file);
    if (!pdfValidation.valid) {
      errorMessage.value = pdfValidation.error;
      return false;
    }
  }

  return true;
}

function validateImage(file) {
  return new Promise((resolve) => {
    const img = new Image();
    img.onload = () => {
      if (img.width < MIN_IMAGE_WIDTH || img.width > MAX_IMAGE_WIDTH ||
          img.height < MIN_IMAGE_HEIGHT || img.height > MAX_IMAGE_HEIGHT) {
        resolve({ 
          valid: false, 
          error: `${file.name} の寸法が許可範囲外です。${MIN_IMAGE_WIDTH}x${MIN_IMAGE_HEIGHT}から${MAX_IMAGE_WIDTH}x${MAX_IMAGE_HEIGHT}ピクセルの範囲内である必要があります。`
        });
      } else {
        file.preview = img.src;
        resolve({ valid: true });
      }
    };
    img.onerror = () => {
      resolve({ valid: false, error: `${file.name} は正しく開けませんでした。` });
    };
    img.src = URL.createObjectURL(file);
  });
}

function validatePDF(file) {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const arr = new Uint8Array(e.target.result).subarray(0, 5);
      const header = String.fromCharCode.apply(null, arr);
      if (header !== "%PDF-") {
        resolve({ valid: false, error: `${file.name} は有効なPDFファイルではありません。` });
      } else {
        resolve({ valid: true });
      }
    };
    reader.onerror = () => {
      resolve({ valid: false, error: `${file.name} は正しく読み取れませんでした。` });
    };
    reader.readAsArrayBuffer(file.slice(0, 5));
  });
}

function uploadFiles() {
  console.log('アップロード処理:', validFiles.value);
}

function formatFileSize(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}

const zoomLevel = ref(100);
const zoomStep = 20;

function zoomIn() {
  zoomLevel.value = Math.min(zoomLevel.value + zoomStep, 200);
}

function zoomOut() {
  zoomLevel.value = Math.max(zoomLevel.value - zoomStep, 20);
}
</script>


<style scoped>
.upload-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.drop-zone {
  border: 2px dashed #ccc;
  border-radius: 20px;
  width: 100%;
  font-family: sans-serif;
  margin: 20px 0;
  padding: 40px;
  text-align: center;
  font-size: 18px;
}

.drop-zone.drag-over {
  background-color: #f0f0f0;
  border-color: #999;
}

.preview-controls {
  margin: 20px 0;
  width: 100%;
  text-align: center;
}

.preview-controls button {
  margin: 0 10px;
  padding: 10px 20px;
  font-size: 16px;
}

.preview-area {
  margin: 20px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.preview-scroll {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 40px;
  padding: 20px;
}

.preview-item {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.preview {
  object-fit: contain;
  border: 1px solid #ddd;
  border-radius: 4px;
  transition: transform 0.3s ease;
}

.file-info {
  font-size: 14px;
  margin-top: 10px;
  word-break: break-all;
  max-width: 300px;
}

.button-group {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin: 20px 0;
}

.button-group button {
  padding: 10px 20px;
  font-size: 16px;
}

.error {
  color: red;
  margin-top: 10px;
  font-size: 16px;
}
</style>