<template>
  <div>
    <input type="file" @change="handleFileChange" ref="fileInput" multiple accept=".jpg,.jpeg,.png,.pdf">
    <button @click="uploadFiles" :disabled="!isValid">アップロード</button>
    <div>最大ファイルサイズ: {{ formatFileSize(MAX_FILE_SIZE) }}</div>
    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
    <ul v-if="validFiles.length">
      <li v-for="file in validFiles" :key="file.name">
        {{ file.name }} - {{ formatFileSize(file.size) }}
        <img v-if="file.type.startsWith('image/')" :src="file.preview" class="preview" />
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

const fileInput = ref(null);
const validFiles = ref([]);
const errorMessage = ref('');

const MAX_FILE_SIZE = 1 * 1024 * 1024; // 1MB
// const MAX_FILE_SIZE = 20 * 1024 * 1024; // 20MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];
const MIN_IMAGE_WIDTH = 100;
const MAX_IMAGE_WIDTH = 4000;
const MIN_IMAGE_HEIGHT = 100;
const MAX_IMAGE_HEIGHT = 4000;

const isValid = computed(() => validFiles.value.length > 0 && !errorMessage.value);

async function handleFileChange(event) {
  const files = Array.from(event.target.files);
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

  // 他の検証ロジック（変更なし）
  // ...

  return true;
}

function formatFileSize(bytes) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}

// その他の関数（変更なし）
// ...

</script>

<style scoped>
.error {
  color: red;
  margin-top: 10px;
}
.preview {
  max-width: 100px;
  max-height: 100px;
  margin-left: 10px;
}
</style>