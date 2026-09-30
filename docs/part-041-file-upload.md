# Part 41: File Upload ด้วย Vue.js และ Nuxt.js

## File Upload คืออะไร?

File Upload เป็นฟีเจอร์พื้นฐานที่แอปพลิเคชันส่วนใหญ่ต้องการ ตั้งแต่การอัปโหลดรูปภาพโปรไฟล์ ไปจนถึงการส่งเอกสาร PDF หรือวิดีโอ Vue.js และ Nuxt.js มีเครื่องมือที่ทรงพลังในการจัดการกับไฟล์อัปโหลด

## 1. File Upload ด้วย HTML5

HTML5 มี File API ที่ช่วยให้เราอ่านและจัดการไฟล์ได้โดยตรงในเบราว์เซอร์

### Input File พื้นฐาน

```vue
<!-- components/BasicFileUpload.vue -->
<template>
  <div class="file-upload-basic">
    <h2>Basic File Upload</h2>
    
    <!-- Input สำหรับเลือกไฟล์ -->
    <input
      type="file"
      ref="fileInput"
      @change="handleFileChange"
      accept="image/*,.pdf,.doc,.docx"
      multiple
      class="file-input"
    />
    
    <!-- แสดงรายการไฟล์ที่เลือก -->
    <div v-if="selectedFiles.length > 0" class="file-list">
      <h3>ไฟล์ที่เลือก:</h3>
      <ul>
        <li v-for="(file, index) in selectedFiles" :key="index">
          <span>{{ file.name }}</span>
          <span class="file-size">{{ formatFileSize(file.size) }}</span>
          <span class="file-type">{{ file.type }}</span>
          <button @click="removeFile(index)" class="btn-remove">ลบ</button>
        </li>
      </ul>
    </div>
    
    <!-- ปุ่มอัปโหลด -->
    <button
      @click="uploadFiles"
      :disabled="selectedFiles.length === 0 || isUploading"
      class="btn-upload"
    >
      {{ isUploading ? 'กำลังอัปโหลด...' : 'อัปโหลดไฟล์' }}
    </button>
    
    <!-- แสดงผลการอัปโหลด -->
    <div v-if="uploadResult" class="upload-result" :class="uploadResult.success ? 'success' : 'error'">
      {{ uploadResult.message }}
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const fileInput = ref(null)
const selectedFiles = ref([])
const isUploading = ref(false)
const uploadResult = ref(null)

// จัดการเมื่อเลือกไฟล์
const handleFileChange = (event) => {
  const files = Array.from(event.target.files)
  selectedFiles.value = files
  uploadResult.value = null
}

// ลบไฟล์ออกจากรายการ
const removeFile = (index) => {
  selectedFiles.value.splice(index, 1)
}

// แปลงขนาดไฟล์ให้อ่านง่าย
const formatFileSize = (bytes) => {
  if (bytes === 0) return '0 Bytes'
  const k = 1024
  const sizes = ['Bytes', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

// อัปโหลดไฟล์
const uploadFiles = async () => {
  if (selectedFiles.value.length === 0) return
  
  isUploading.value = true
  uploadResult.value = null
  
  try {
    const formData = new FormData()
    selectedFiles.value.forEach((file) => {
      formData.append('files', file)
    })
    
    const response = await $fetch('/api/upload', {
      method: 'POST',
      body: formData
    })
    
    uploadResult.value = {
      success: true,
      message: `อัปโหลดสำเร็จ! ${response.files.length} ไฟล์`
    }
    selectedFiles.value = []
  } catch (error) {
    uploadResult.value = {
      success: false,
      message: `เกิดข้อผิดพลาด: ${error.message}`
    }
  } finally {
    isUploading.value = false
  }
}
</script>

<style scoped>
.file-upload-basic {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}

.file-input {
  display: block;
  margin: 16px 0;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 100%;
}

.file-list {
  margin: 16px 0;
  padding: 12px;
  background: #f5f5f5;
  border-radius: 8px;
}

.file-list li {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 0;
  border-bottom: 1px solid #e0e0e0;
}

.file-size,
.file-type {
  color: #666;
  font-size: 0.85em;
}

.btn-remove {
  margin-left: auto;
  padding: 4px 8px;
  background: #ff4444;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-upload {
  padding: 10px 24px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1em;
}

.btn-upload:disabled {
  background: #aaa;
  cursor: not-allowed;
}

.upload-result {
  margin-top: 16px;
  padding: 12px;
  border-radius: 4px;
  font-weight: bold;
}

.upload-result.success {
  background: #d4edda;
  color: #155724;
}

.upload-result.error {
  background: #f8d7da;
  color: #721c24;
}
</style>
```

## 2. Drag & Drop Upload

การทำ Drag & Drop ช่วยให้ผู้ใช้สามารถลากไฟล์มาวางได้โดยตรง

```vue
<!-- components/DragDropUpload.vue -->
<template>
  <div class="drag-drop-container">
    <div
      class="drop-zone"
      :class="{
        'is-dragging': isDragging,
        'has-files': files.length > 0
      }"
      @dragenter.prevent="handleDragEnter"
      @dragleave.prevent="handleDragLeave"
      @dragover.prevent
      @drop.prevent="handleDrop"
      @click="triggerFileInput"
    >
      <div v-if="!isDragging" class="drop-zone-content">
        <div class="upload-icon">📁</div>
        <p class="drop-text">ลากและวางไฟล์ที่นี่</p>
        <p class="drop-subtext">หรือคลิกเพื่อเลือกไฟล์</p>
        <p class="drop-hint">รองรับ: JPG, PNG, PDF, DOCX (สูงสุด 10MB)</p>
      </div>
      
      <div v-else class="drag-active-content">
        <div class="upload-icon bounce">⬇️</div>
        <p>วางไฟล์ที่นี่!</p>
      </div>
      
      <!-- Hidden input -->
      <input
        ref="hiddenInput"
        type="file"
        multiple
        accept="image/*,.pdf,.doc,.docx"
        @change="handleInputChange"
        style="display: none"
      />
    </div>
    
    <!-- Preview Area -->
    <div v-if="files.length > 0" class="files-preview">
      <div class="preview-header">
        <h3>ไฟล์ที่เลือก ({{ files.length }})</h3>
        <button @click="clearAll" class="btn-clear">ล้างทั้งหมด</button>
      </div>
      
      <div class="preview-grid">
        <div
          v-for="(file, index) in files"
          :key="file.name"
          class="preview-item"
        >
          <!-- Preview รูปภาพ -->
          <div class="preview-thumb">
            <img
              v-if="file.preview"
              :src="file.preview"
              :alt="file.name"
            />
            <div v-else class="file-icon">
              {{ getFileIcon(file.type) }}
            </div>
          </div>
          
          <div class="preview-info">
            <p class="file-name" :title="file.name">{{ truncateFileName(file.name) }}</p>
            <p class="file-meta">{{ formatSize(file.size) }} · {{ file.type || 'unknown' }}</p>
          </div>
          
          <button
            class="btn-remove-preview"
            @click="removeFile(index)"
          >×</button>
        </div>
      </div>
      
      <button
        @click="uploadAll"
        :disabled="isUploading"
        class="btn-upload-all"
      >
        {{ isUploading ? `กำลังอัปโหลด... ${uploadProgress}%` : `อัปโหลด ${files.length} ไฟล์` }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue'

const isDragging = ref(false)
const files = ref([])
const isUploading = ref(false)
const uploadProgress = ref(0)
const hiddenInput = ref(null)
let dragCounter = 0

// จัดการ Drag Events
const handleDragEnter = () => {
  dragCounter++
  isDragging.value = true
}

const handleDragLeave = () => {
  dragCounter--
  if (dragCounter === 0) {
    isDragging.value = false
  }
}

const handleDrop = (event) => {
  isDragging.value = false
  dragCounter = 0
  const droppedFiles = Array.from(event.dataTransfer.files)
  addFiles(droppedFiles)
}

const handleInputChange = (event) => {
  const selectedFiles = Array.from(event.target.files)
  addFiles(selectedFiles)
  event.target.value = '' // reset input
}

const triggerFileInput = () => {
  hiddenInput.value?.click()
}

// เพิ่มไฟล์พร้อม validation
const addFiles = async (newFiles) => {
  const validFiles = newFiles.filter(file => {
    // ตรวจสอบขนาดไฟล์ (10MB)
    if (file.size > 10 * 1024 * 1024) {
      alert(`ไฟล์ ${file.name} มีขนาดเกิน 10MB`)
      return false
    }
    return true
  })
  
  // สร้าง preview สำหรับรูปภาพ
  const filesWithPreview = await Promise.all(
    validFiles.map(async (file) => {
      const preview = await createPreview(file)
      return Object.assign(file, { preview })
    })
  )
  
  files.value = [...files.value, ...filesWithPreview]
}

// สร้าง preview URL สำหรับรูปภาพ
const createPreview = (file) => {
  return new Promise((resolve) => {
    if (!file.type.startsWith('image/')) {
      resolve(null)
      return
    }
    const reader = new FileReader()
    reader.onload = (e) => resolve(e.target.result)
    reader.readAsDataURL(file)
  })
}

const removeFile = (index) => {
  files.value.splice(index, 1)
}

const clearAll = () => {
  files.value = []
}

const getFileIcon = (type) => {
  if (type.includes('pdf')) return '📄'
  if (type.includes('word') || type.includes('document')) return '📝'
  if (type.includes('video')) return '🎬'
  if (type.includes('audio')) return '🎵'
  return '📎'
}

const truncateFileName = (name, maxLength = 20) => {
  if (name.length <= maxLength) return name
  const ext = name.split('.').pop()
  return name.substring(0, maxLength - ext.length - 4) + '...' + ext
}

const formatSize = (bytes) => {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

const uploadAll = async () => {
  isUploading.value = true
  uploadProgress.value = 0
  
  try {
    const formData = new FormData()
    files.value.forEach(file => formData.append('files', file))
    
    // ใช้ XMLHttpRequest สำหรับ progress tracking
    await uploadWithProgress(formData)
    
    files.value = []
    alert('อัปโหลดสำเร็จ!')
  } catch (error) {
    alert('เกิดข้อผิดพลาดในการอัปโหลด: ' + error.message)
  } finally {
    isUploading.value = false
    uploadProgress.value = 0
  }
}

const uploadWithProgress = (formData) => {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest()
    
    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        uploadProgress.value = Math.round((event.loaded / event.total) * 100)
      }
    }
    
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText))
      } else {
        reject(new Error('Upload failed: ' + xhr.statusText))
      }
    }
    
    xhr.onerror = () => reject(new Error('Network error'))
    xhr.open('POST', '/api/upload')
    xhr.send(formData)
  })
}

onUnmounted(() => {
  // Cleanup object URLs
  files.value.forEach(file => {
    if (file.preview) URL.revokeObjectURL(file.preview)
  })
})
</script>

<style scoped>
.drag-drop-container {
  max-width: 700px;
  margin: 0 auto;
}

.drop-zone {
  border: 2px dashed #ccc;
  border-radius: 12px;
  padding: 40px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background: #fafafa;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.drop-zone:hover {
  border-color: #4CAF50;
  background: #f0f8f0;
}

.drop-zone.is-dragging {
  border-color: #2196F3;
  background: #e3f2fd;
  transform: scale(1.02);
}

.upload-icon {
  font-size: 48px;
  margin-bottom: 12px;
}

.bounce {
  animation: bounce 0.5s infinite alternate;
}

@keyframes bounce {
  from { transform: translateY(0); }
  to { transform: translateY(-10px); }
}

.drop-text {
  font-size: 1.2em;
  font-weight: bold;
  color: #333;
}

.drop-subtext {
  color: #666;
  margin: 4px 0;
}

.drop-hint {
  font-size: 0.8em;
  color: #999;
}

.files-preview {
  margin-top: 24px;
}

.preview-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.btn-clear {
  padding: 6px 16px;
  background: #ff5722;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.preview-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 16px;
  margin-bottom: 20px;
}

.preview-item {
  position: relative;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  overflow: hidden;
}

.preview-thumb {
  height: 100px;
  background: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
}

.preview-thumb img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.file-icon {
  font-size: 36px;
}

.preview-info {
  padding: 8px;
}

.file-name {
  font-size: 0.8em;
  font-weight: bold;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-meta {
  font-size: 0.7em;
  color: #999;
}

.btn-remove-preview {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 20px;
  height: 20px;
  background: rgba(0,0,0,0.6);
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  line-height: 1;
}

.btn-upload-all {
  width: 100%;
  padding: 12px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.1em;
}

.btn-upload-all:disabled {
  background: #aaa;
}
</style>
```

## 3. Progress Tracking

```vue
<!-- components/UploadWithProgress.vue -->
<template>
  <div class="upload-progress-demo">
    <h2>Upload พร้อม Progress Tracking</h2>
    
    <input
      type="file"
      multiple
      @change="handleFiles"
      ref="input"
    />
    
    <div v-if="uploadQueue.length > 0" class="upload-queue">
      <div
        v-for="item in uploadQueue"
        :key="item.id"
        class="upload-item"
      >
        <div class="item-header">
          <span class="item-name">{{ item.file.name }}</span>
          <span class="item-status" :class="item.status">
            {{ statusText(item.status) }}
          </span>
        </div>
        
        <!-- Progress Bar -->
        <div class="progress-bar-container">
          <div
            class="progress-bar"
            :style="{ width: item.progress + '%' }"
            :class="item.status"
          ></div>
        </div>
        
        <div class="item-footer">
          <span class="progress-text">{{ item.progress }}%</span>
          <span v-if="item.speed" class="upload-speed">{{ item.speed }}</span>
          <span v-if="item.eta" class="eta">ETA: {{ item.eta }}</span>
        </div>
        
        <!-- Error Message -->
        <div v-if="item.error" class="item-error">
          {{ item.error }}
        </div>
      </div>
    </div>
    
    <div class="actions">
      <button @click="startUpload" :disabled="isUploading || uploadQueue.length === 0">
        เริ่มอัปโหลด
      </button>
      <button @click="pauseAll" v-if="isUploading">
        หยุดชั่วคราว
      </button>
      <button @click="clearQueue" :disabled="isUploading">
        ล้างคิว
      </button>
    </div>
    
    <!-- Overall Progress -->
    <div v-if="uploadQueue.length > 0" class="overall-progress">
      <strong>ความคืบหน้าโดยรวม: {{ overallProgress }}%</strong>
      <div class="progress-bar-container">
        <div
          class="progress-bar overall"
          :style="{ width: overallProgress + '%' }"
        ></div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const input = ref(null)
const uploadQueue = ref([])
const isUploading = ref(false)
let idCounter = 0

const overallProgress = computed(() => {
  if (uploadQueue.value.length === 0) return 0
  const total = uploadQueue.value.reduce((sum, item) => sum + item.progress, 0)
  return Math.round(total / uploadQueue.value.length)
})

const handleFiles = (event) => {
  const files = Array.from(event.target.files)
  files.forEach(file => {
    uploadQueue.value.push({
      id: ++idCounter,
      file,
      progress: 0,
      status: 'pending',
      error: null,
      speed: null,
      eta: null,
      xhr: null
    })
  })
}

const statusText = (status) => {
  const texts = {
    pending: '⏳ รอ',
    uploading: '⬆️ กำลังอัปโหลด',
    success: '✅ สำเร็จ',
    error: '❌ ผิดพลาด',
    paused: '⏸️ หยุด'
  }
  return texts[status] || status
}

const startUpload = async () => {
  isUploading.value = true
  
  const pendingItems = uploadQueue.value.filter(item => item.status === 'pending')
  
  // อัปโหลดพร้อมกัน 3 ไฟล์
  const CONCURRENT_UPLOADS = 3
  const chunks = []
  
  for (let i = 0; i < pendingItems.length; i += CONCURRENT_UPLOADS) {
    chunks.push(pendingItems.slice(i, i + CONCURRENT_UPLOADS))
  }
  
  for (const chunk of chunks) {
    await Promise.all(chunk.map(item => uploadFile(item)))
  }
  
  isUploading.value = false
}

const uploadFile = (item) => {
  return new Promise((resolve) => {
    item.status = 'uploading'
    
    const formData = new FormData()
    formData.append('file', item.file)
    
    const xhr = new XMLHttpRequest()
    item.xhr = xhr
    
    let startTime = Date.now()
    let lastLoaded = 0
    
    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        const progress = Math.round((event.loaded / event.total) * 100)
        item.progress = progress
        
        // คำนวณความเร็ว
        const elapsed = (Date.now() - startTime) / 1000
        const loadedDiff = event.loaded - lastLoaded
        const speed = loadedDiff / elapsed
        lastLoaded = event.loaded
        startTime = Date.now()
        
        item.speed = formatSpeed(speed)
        
        // คำนวณ ETA
        const remaining = event.total - event.loaded
        const eta = remaining / speed
        item.eta = formatTime(eta)
      }
    }
    
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        item.status = 'success'
        item.progress = 100
        item.speed = null
        item.eta = null
      } else {
        item.status = 'error'
        item.error = 'HTTP ' + xhr.status
      }
      resolve()
    }
    
    xhr.onerror = () => {
      item.status = 'error'
      item.error = 'เกิดข้อผิดพลาดเครือข่าย'
      resolve()
    }
    
    xhr.open('POST', '/api/upload')
    xhr.send(formData)
  })
}

const pauseAll = () => {
  uploadQueue.value.forEach(item => {
    if (item.status === 'uploading' && item.xhr) {
      item.xhr.abort()
      item.status = 'paused'
    }
  })
  isUploading.value = false
}

const clearQueue = () => {
  uploadQueue.value = []
}

const formatSpeed = (bytesPerSec) => {
  if (bytesPerSec < 1024) return `${bytesPerSec.toFixed(0)} B/s`
  if (bytesPerSec < 1024 * 1024) return `${(bytesPerSec / 1024).toFixed(1)} KB/s`
  return `${(bytesPerSec / (1024 * 1024)).toFixed(1)} MB/s`
}

const formatTime = (seconds) => {
  if (seconds < 60) return `${Math.round(seconds)}s`
  return `${Math.floor(seconds / 60)}m ${Math.round(seconds % 60)}s`
}
</script>
```

## 4. Image Preview

```vue
<!-- components/ImagePreviewUpload.vue -->
<template>
  <div class="image-preview-upload">
    <h2>Image Upload พร้อม Preview</h2>
    
    <div class="upload-area" @click="$refs.fileInput.click()">
      <input
        type="file"
        ref="fileInput"
        accept="image/*"
        multiple
        @change="handleImageSelect"
        style="display: none"
      />
      <p>คลิกเพื่อเลือกรูปภาพ</p>
    </div>
    
    <!-- Image Gallery Preview -->
    <div v-if="images.length > 0" class="image-gallery">
      <div
        v-for="(img, index) in images"
        :key="img.id"
        class="gallery-item"
        @click="openLightbox(index)"
      >
        <img :src="img.preview" :alt="img.file.name" />
        
        <!-- Overlay Controls -->
        <div class="item-overlay">
          <button @click.stop="rotateImage(index)">🔄</button>
          <button @click.stop="cropImage(index)">✂️</button>
          <button @click.stop="removeImage(index)">🗑️</button>
        </div>
        
        <div class="item-info">
          <span>{{ img.file.name }}</span>
          <span>{{ formatSize(img.file.size) }}</span>
        </div>
      </div>
    </div>
    
    <!-- Lightbox -->
    <div v-if="lightboxOpen" class="lightbox" @click="closeLightbox">
      <div class="lightbox-content" @click.stop>
        <button class="lightbox-close" @click="closeLightbox">×</button>
        <button class="lightbox-prev" @click="prevImage">❮</button>
        <img :src="currentImage.preview" :alt="currentImage.file.name" />
        <button class="lightbox-next" @click="nextImage">❯</button>
        <p class="lightbox-caption">{{ currentImage.file.name }}</p>
      </div>
    </div>
    
    <div class="upload-actions" v-if="images.length > 0">
      <button @click="uploadImages" :disabled="isUploading">
        {{ isUploading ? 'กำลังอัปโหลด...' : `อัปโหลด ${images.length} รูป` }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const fileInput = ref(null)
const images = ref([])
const lightboxOpen = ref(false)
const currentIndex = ref(0)
const isUploading = ref(false)
let idCounter = 0

const currentImage = computed(() => images.value[currentIndex.value])

const handleImageSelect = async (event) => {
  const files = Array.from(event.target.files)
  
  for (const file of files) {
    const preview = await createPreview(file)
    images.value.push({
      id: ++idCounter,
      file,
      preview,
      rotation: 0
    })
  }
}

const createPreview = (file) => {
  return new Promise((resolve) => {
    const reader = new FileReader()
    reader.onload = (e) => resolve(e.target.result)
    reader.readAsDataURL(file)
  })
}

const removeImage = (index) => {
  images.value.splice(index, 1)
}

const rotateImage = (index) => {
  images.value[index].rotation = (images.value[index].rotation + 90) % 360
}

const cropImage = (index) => {
  // ในโปรเจกต์จริงอาจใช้ Cropper.js
  console.log('Crop image at index:', index)
}

const openLightbox = (index) => {
  currentIndex.value = index
  lightboxOpen.value = true
}

const closeLightbox = () => {
  lightboxOpen.value = false
}

const prevImage = () => {
  currentIndex.value = (currentIndex.value - 1 + images.value.length) % images.value.length
}

const nextImage = () => {
  currentIndex.value = (currentIndex.value + 1) % images.value.length
}

const formatSize = (bytes) => {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

const uploadImages = async () => {
  isUploading.value = true
  const formData = new FormData()
  images.value.forEach(img => formData.append('images', img.file))
  
  try {
    await $fetch('/api/upload/images', { method: 'POST', body: formData })
    images.value = []
    alert('อัปโหลดสำเร็จ!')
  } catch (err) {
    alert('เกิดข้อผิดพลาด: ' + err.message)
  } finally {
    isUploading.value = false
  }
}
</script>
```

## 5. File Type Validation

```typescript
// utils/fileValidation.ts
export interface FileValidationOptions {
  maxSize?: number        // bytes
  allowedTypes?: string[] // MIME types
  allowedExtensions?: string[]
  minWidth?: number       // for images
  minHeight?: number      // for images
  maxWidth?: number
  maxHeight?: number
}

export interface ValidationResult {
  valid: boolean
  errors: string[]
}

export const validateFile = async (
  file: File,
  options: FileValidationOptions = {}
): Promise<ValidationResult> => {
  const errors: string[] = []

  // ตรวจสอบขนาดไฟล์
  if (options.maxSize && file.size > options.maxSize) {
    const maxMB = (options.maxSize / (1024 * 1024)).toFixed(2)
    errors.push(`ไฟล์มีขนาดเกิน ${maxMB} MB`)
  }

  // ตรวจสอบ MIME type
  if (options.allowedTypes && options.allowedTypes.length > 0) {
    const isTypeAllowed = options.allowedTypes.some(type => {
      if (type.endsWith('/*')) {
        return file.type.startsWith(type.slice(0, -2))
      }
      return file.type === type
    })
    if (!isTypeAllowed) {
      errors.push(`ประเภทไฟล์ ${file.type} ไม่ได้รับอนุญาต`)
    }
  }

  // ตรวจสอบนามสกุลไฟล์
  if (options.allowedExtensions && options.allowedExtensions.length > 0) {
    const ext = file.name.split('.').pop()?.toLowerCase() || ''
    if (!options.allowedExtensions.includes(ext)) {
      errors.push(`นามสกุลไฟล์ .${ext} ไม่ได้รับอนุญาต`)
    }
  }

  // ตรวจสอบขนาดรูปภาพ
  if (file.type.startsWith('image/') && (
    options.minWidth || options.minHeight ||
    options.maxWidth || options.maxHeight
  )) {
    const dimensions = await getImageDimensions(file)
    
    if (options.minWidth && dimensions.width < options.minWidth) {
      errors.push(`รูปภาพต้องมีความกว้างอย่างน้อย ${options.minWidth}px`)
    }
    if (options.minHeight && dimensions.height < options.minHeight) {
      errors.push(`รูปภาพต้องมีความสูงอย่างน้อย ${options.minHeight}px`)
    }
    if (options.maxWidth && dimensions.width > options.maxWidth) {
      errors.push(`รูปภาพต้องมีความกว้างไม่เกิน ${options.maxWidth}px`)
    }
    if (options.maxHeight && dimensions.height > options.maxHeight) {
      errors.push(`รูปภาพต้องมีความสูงไม่เกิน ${options.maxHeight}px`)
    }
  }

  return {
    valid: errors.length === 0,
    errors
  }
}

const getImageDimensions = (file: File): Promise<{ width: number; height: number }> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve({ width: img.width, height: img.height })
    img.onerror = reject
    img.src = URL.createObjectURL(file)
  })
}
```

## 6. Upload to Cloudinary

```typescript
// server/api/upload/cloudinary.post.ts
import { v2 as cloudinary } from 'cloudinary'
import { readMultipartFormData } from 'h3'

export default defineEventHandler(async (event) => {
  // ตั้งค่า Cloudinary
  cloudinary.config({
    cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
    api_key: process.env.CLOUDINARY_API_KEY,
    api_secret: process.env.CLOUDINARY_API_SECRET,
  })

  // อ่านข้อมูลจาก multipart form
  const formData = await readMultipartFormData(event)
  if (!formData) {
    throw createError({ statusCode: 400, message: 'ไม่มีข้อมูลไฟล์' })
  }

  const uploadedFiles = []
  
  for (const field of formData) {
    if (!field.filename) continue // ข้ามถ้าไม่ใช่ไฟล์
    
    // แปลงเป็น base64 สำหรับ Cloudinary
    const base64 = `data:${field.type};base64,${field.data.toString('base64')}`
    
    const result = await cloudinary.uploader.upload(base64, {
      folder: 'vue-course-uploads',
      transformation: [
        { width: 1200, height: 1200, crop: 'limit' },
        { quality: 'auto:good' },
        { fetch_format: 'auto' }
      ]
    })
    
    uploadedFiles.push({
      publicId: result.public_id,
      url: result.secure_url,
      thumbnailUrl: cloudinary.url(result.public_id, {
        width: 200,
        height: 200,
        crop: 'fill',
        quality: 'auto',
        fetch_format: 'auto'
      }),
      width: result.width,
      height: result.height,
      format: result.format,
      size: result.bytes
    })
  }
  
  return { files: uploadedFiles }
})
```

## 7. ตัวอย่าง: Complete File Upload Component

```vue
<!-- components/CompleteFileUpload.vue -->
<template>
  <div class="complete-upload">
    <h1>Complete File Upload System</h1>
    
    <!-- Upload Zone -->
    <div
      class="upload-zone"
      :class="{ active: isDragging, disabled: isUploading }"
      @dragover.prevent="isDragging = true"
      @dragleave.prevent="isDragging = false"
      @drop.prevent="handleDrop"
      @click="!isUploading && $refs.fileInput.click()"
    >
      <input
        ref="fileInput"
        type="file"
        multiple
        :accept="acceptedTypes"
        @change="handleFileSelect"
        style="display: none"
      />
      
      <div class="zone-content">
        <div class="zone-icon">{{ isDragging ? '⬇️' : '☁️' }}</div>
        <h3>{{ isDragging ? 'วางไฟล์ที่นี่' : 'ลากหรือคลิกเพื่อเลือกไฟล์' }}</h3>
        <p class="zone-hint">
          รองรับ: {{ acceptedExtensions.join(', ') }} · 
          สูงสุด {{ maxFileSizeMB }}MB ต่อไฟล์
        </p>
      </div>
    </div>
    
    <!-- Validation Errors -->
    <div v-if="validationErrors.length > 0" class="validation-errors">
      <div v-for="error in validationErrors" :key="error" class="error-item">
        ⚠️ {{ error }}
      </div>
    </div>
    
    <!-- File Queue -->
    <div v-if="fileQueue.length > 0" class="file-queue">
      <div class="queue-header">
        <h3>คิวการอัปโหลด ({{ fileQueue.length }} ไฟล์)</h3>
        <div class="queue-actions">
          <button
            v-if="hasCompletedFiles"
            @click="clearCompleted"
            class="btn-secondary"
          >
            ล้างที่เสร็จแล้ว
          </button>
          <button
            @click="uploadAll"
            :disabled="isUploading || !hasPendingFiles"
            class="btn-primary"
          >
            อัปโหลดทั้งหมด
          </button>
        </div>
      </div>
      
      <div class="queue-list">
        <div
          v-for="item in fileQueue"
          :key="item.id"
          class="queue-item"
          :class="item.status"
        >
          <!-- Preview -->
          <div class="item-preview">
            <img v-if="item.preview" :src="item.preview" :alt="item.file.name" />
            <span v-else class="file-type-icon">{{ getIcon(item.file.type) }}</span>
          </div>
          
          <!-- Info -->
          <div class="item-details">
            <div class="item-name">{{ item.file.name }}</div>
            <div class="item-meta">{{ formatSize(item.file.size) }} · {{ item.file.type }}</div>
            
            <!-- Progress -->
            <div v-if="item.status === 'uploading'" class="progress-container">
              <div class="progress-track">
                <div
                  class="progress-fill"
                  :style="{ width: item.progress + '%' }"
                ></div>
              </div>
              <span class="progress-label">{{ item.progress }}%</span>
            </div>
            
            <!-- Result URL -->
            <div v-if="item.status === 'success' && item.resultUrl" class="result-url">
              <a :href="item.resultUrl" target="_blank">ดูไฟล์ที่อัปโหลด</a>
            </div>
            
            <!-- Error -->
            <div v-if="item.status === 'error'" class="item-error">
              {{ item.errorMessage }}
              <button @click="retryUpload(item)">ลองใหม่</button>
            </div>
          </div>
          
          <!-- Status Icon -->
          <div class="item-status-icon">
            <span v-if="item.status === 'pending'">⏳</span>
            <span v-else-if="item.status === 'uploading'">⬆️</span>
            <span v-else-if="item.status === 'success'">✅</span>
            <span v-else-if="item.status === 'error'">❌</span>
          </div>
          
          <!-- Remove Button -->
          <button
            v-if="item.status !== 'uploading'"
            @click="removeFromQueue(item.id)"
            class="btn-remove"
          >×</button>
        </div>
      </div>
      
      <!-- Summary -->
      <div class="queue-summary">
        <span class="summary-item pending">รอ: {{ pendingCount }}</span>
        <span class="summary-item uploading">กำลังอัปโหลด: {{ uploadingCount }}</span>
        <span class="summary-item success">สำเร็จ: {{ successCount }}</span>
        <span class="summary-item error">ผิดพลาด: {{ errorCount }}</span>
      </div>
    </div>
    
    <!-- Uploaded Gallery -->
    <div v-if="uploadedFiles.length > 0" class="uploaded-gallery">
      <h3>ไฟล์ที่อัปโหลดแล้ว</h3>
      <div class="gallery-grid">
        <div
          v-for="file in uploadedFiles"
          :key="file.publicId"
          class="gallery-item"
        >
          <img :src="file.thumbnailUrl" :alt="file.publicId" />
          <div class="gallery-item-overlay">
            <a :href="file.url" target="_blank" class="btn-view">ดู</a>
            <button @click="copyUrl(file.url)" class="btn-copy">คัดลอก URL</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from 'vue'

// Config
const acceptedExtensions = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'pdf']
const acceptedTypes = 'image/*,.pdf'
const maxFileSizeMB = 10
const maxFileSize = maxFileSizeMB * 1024 * 1024

// State
const fileInput = ref(null)
const isDragging = ref(false)
const isUploading = ref(false)
const fileQueue = ref([])
const uploadedFiles = ref([])
const validationErrors = ref([])
let idCounter = 0

// Computed
const pendingCount = computed(() => fileQueue.value.filter(i => i.status === 'pending').length)
const uploadingCount = computed(() => fileQueue.value.filter(i => i.status === 'uploading').length)
const successCount = computed(() => fileQueue.value.filter(i => i.status === 'success').length)
const errorCount = computed(() => fileQueue.value.filter(i => i.status === 'error').length)
const hasPendingFiles = computed(() => pendingCount.value > 0)
const hasCompletedFiles = computed(() => successCount.value > 0 || errorCount.value > 0)

// Handlers
const handleDrop = (event) => {
  isDragging.value = false
  addFiles(Array.from(event.dataTransfer.files))
}

const handleFileSelect = (event) => {
  addFiles(Array.from(event.target.files))
}

const addFiles = async (files) => {
  validationErrors.value = []
  
  for (const file of files) {
    // Validate
    const errors = validateFile(file)
    if (errors.length > 0) {
      validationErrors.value.push(...errors.map(e => `${file.name}: ${e}`))
      continue
    }
    
    const preview = file.type.startsWith('image/') ? await createPreview(file) : null
    
    fileQueue.value.push({
      id: ++idCounter,
      file,
      preview,
      status: 'pending',
      progress: 0,
      resultUrl: null,
      errorMessage: null
    })
  }
}

const validateFile = (file) => {
  const errors = []
  const ext = file.name.split('.').pop()?.toLowerCase()
  
  if (!acceptedExtensions.includes(ext)) {
    errors.push(`นามสกุล .${ext} ไม่รองรับ`)
  }
  if (file.size > maxFileSize) {
    errors.push(`ขนาดเกิน ${maxFileSizeMB}MB`)
  }
  return errors
}

const createPreview = (file) => {
  return new Promise((resolve) => {
    const reader = new FileReader()
    reader.onload = e => resolve(e.target.result)
    reader.readAsDataURL(file)
  })
}

const removeFromQueue = (id) => {
  fileQueue.value = fileQueue.value.filter(item => item.id !== id)
}

const clearCompleted = () => {
  fileQueue.value = fileQueue.value.filter(
    item => item.status !== 'success' && item.status !== 'error'
  )
}

const uploadAll = async () => {
  isUploading.value = true
  const pending = fileQueue.value.filter(item => item.status === 'pending')
  
  await Promise.all(pending.map(item => uploadFile(item)))
  isUploading.value = false
}

const uploadFile = async (item) => {
  item.status = 'uploading'
  item.progress = 0
  
  const formData = new FormData()
  formData.append('file', item.file)
  
  try {
    const result = await uploadWithProgress(formData, (progress) => {
      item.progress = progress
    })
    
    item.status = 'success'
    item.progress = 100
    item.resultUrl = result.url
    
    uploadedFiles.value.push(result)
  } catch (error) {
    item.status = 'error'
    item.errorMessage = error.message
  }
}

const uploadWithProgress = (formData, onProgress) => {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest()
    
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        onProgress(Math.round((e.loaded / e.total) * 100))
      }
    }
    
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText))
      } else {
        reject(new Error('Upload failed'))
      }
    }
    
    xhr.onerror = () => reject(new Error('Network error'))
    xhr.open('POST', '/api/upload/cloudinary')
    xhr.send(formData)
  })
}

const retryUpload = (item) => {
  item.status = 'pending'
  item.errorMessage = null
  uploadFile(item)
}

const getIcon = (type) => {
  if (type.includes('pdf')) return '📄'
  if (type.includes('image')) return '🖼️'
  return '📎'
}

const formatSize = (bytes) => {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

const copyUrl = async (url) => {
  await navigator.clipboard.writeText(url)
  alert('คัดลอก URL แล้ว!')
}

onUnmounted(() => {
  fileQueue.value.forEach(item => {
    if (item.preview) URL.revokeObjectURL(item.preview)
  })
})
</script>

<style scoped>
.complete-upload { max-width: 800px; margin: 0 auto; padding: 24px; }
.upload-zone {
  border: 2px dashed #4CAF50;
  border-radius: 12px;
  padding: 48px;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
}
.upload-zone.active { background: #e8f5e9; border-style: solid; }
.upload-zone.disabled { opacity: 0.5; cursor: not-allowed; }
.zone-icon { font-size: 48px; margin-bottom: 12px; }
.zone-hint { color: #666; font-size: 0.85em; }
.validation-errors {
  background: #fff3cd;
  border: 1px solid #ffc107;
  border-radius: 8px;
  padding: 12px;
  margin: 16px 0;
}
.error-item { color: #856404; padding: 4px 0; }
.file-queue { margin-top: 24px; }
.queue-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}
.queue-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  margin-bottom: 8px;
}
.queue-item.success { border-color: #4CAF50; background: #f1f8f1; }
.queue-item.error { border-color: #f44336; background: #fdf1f0; }
.item-preview {
  width: 60px;
  height: 60px;
  border-radius: 4px;
  overflow: hidden;
  flex-shrink: 0;
}
.item-preview img { width: 100%; height: 100%; object-fit: cover; }
.file-type-icon { font-size: 32px; }
.item-details { flex: 1; }
.item-name { font-weight: bold; margin-bottom: 4px; }
.item-meta { font-size: 0.8em; color: #666; }
.progress-container { display: flex; align-items: center; gap: 8px; margin-top: 8px; }
.progress-track {
  flex: 1;
  height: 6px;
  background: #e0e0e0;
  border-radius: 3px;
  overflow: hidden;
}
.progress-fill {
  height: 100%;
  background: #2196F3;
  border-radius: 3px;
  transition: width 0.3s;
}
.progress-label { font-size: 0.8em; color: #666; white-space: nowrap; }
.result-url { margin-top: 4px; }
.result-url a { color: #4CAF50; font-size: 0.85em; }
.item-error { color: #f44336; font-size: 0.85em; }
.item-status-icon { font-size: 20px; }
.btn-remove {
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
  color: #999;
}
.queue-summary {
  display: flex;
  gap: 16px;
  padding: 12px;
  background: #f5f5f5;
  border-radius: 8px;
  font-size: 0.85em;
}
.summary-item.pending { color: #ff9800; }
.summary-item.uploading { color: #2196F3; }
.summary-item.success { color: #4CAF50; }
.summary-item.error { color: #f44336; }
.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 12px;
  margin-top: 16px;
}
.gallery-item {
  position: relative;
  aspect-ratio: 1;
  border-radius: 8px;
  overflow: hidden;
  cursor: pointer;
}
.gallery-item img { width: 100%; height: 100%; object-fit: cover; }
.gallery-item-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  opacity: 0;
  transition: opacity 0.2s;
}
.gallery-item:hover .gallery-item-overlay { opacity: 1; }
.btn-view, .btn-copy {
  padding: 4px 10px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-size: 0.8em;
}
.btn-view { background: white; color: #333; text-decoration: none; }
.btn-copy { background: #4CAF50; color: white; }
.btn-primary {
  padding: 8px 20px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.btn-secondary {
  padding: 8px 20px;
  background: white;
  color: #666;
  border: 1px solid #ccc;
  border-radius: 6px;
  cursor: pointer;
}
</style>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **File Upload พื้นฐาน** - ใช้ input type="file" กับ HTML5 File API
2. **Drag & Drop** - จัดการ dragenter, dragleave, drop events
3. **Progress Tracking** - ใช้ XMLHttpRequest.upload.onprogress
4. **Image Preview** - ใช้ FileReader API อ่านไฟล์เป็น Data URL
5. **File Validation** - ตรวจสอบประเภท ขนาด และขนาดรูปภาพ
6. **Cloudinary Upload** - อัปโหลดขึ้น Cloud Storage พร้อม transformation
7. **Complete Component** - รวมทุกฟีเจอร์เข้าด้วยกัน
