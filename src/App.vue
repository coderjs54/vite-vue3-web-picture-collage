<script setup lang="ts">
import { ref } from 'vue'
import { type ImageItem } from './components/ImageUploader.vue'
import CollageEditor from './components/CollageEditor.vue'

const images = ref<ImageItem[]>([])
const collageEditorRef = ref<InstanceType<typeof CollageEditor> | null>(null)
const uploadInputRef = ref<HTMLInputElement | null>(null)
const galleryScrollRef = ref<HTMLDivElement | null>(null)
const showClearConfirm = ref(false)


const handleImagesAdded = (newImages: ImageItem[]) => {
  images.value = [...images.value, ...newImages]
}

const handleGalleryMouseEnter = () => {
  const scrollEl = galleryScrollRef.value
  if (!scrollEl) return
  
  scrollEl.addEventListener('wheel', handleGalleryWheel, { passive: false })
}

const handleGalleryMouseLeave = () => {
  const scrollEl = galleryScrollRef.value
  if (!scrollEl) return
  
  scrollEl.removeEventListener('wheel', handleGalleryWheel)
}

const handleGalleryWheel = (e: WheelEvent) => {
  const scrollEl = galleryScrollRef.value
  if (!scrollEl) return
  
  const hasScrollbar = scrollEl.scrollWidth > scrollEl.clientWidth
  if (!hasScrollbar) return
  
  e.preventDefault()
  
  const delta = e.deltaY || (e as unknown as { detail: number }).detail || (-(e as unknown as { wheelDelta: number }).wheelDelta || 0)
  const scrollSpeed = 2
  
  const newScrollLeft = scrollEl.scrollLeft + delta * scrollSpeed
  
  scrollEl.scrollTo({
    left: newScrollLeft,
    behavior: 'smooth'
  })
}

const triggerUpload = () => {
  uploadInputRef.value?.click()
}

const generateId = () => `img-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`

const handleUploadChange = (e: Event) => {
  const target = e.target as HTMLInputElement
  const files = target.files
  if (!files) return
  
  const imageFiles = Array.from(files).filter(file => file.type.startsWith('image/'))
  const newImages: ImageItem[] = []
  
  imageFiles.forEach(file => {
    const reader = new FileReader()
    reader.onload = (event) => {
      newImages.push({
        id: generateId(),
        url: event.target?.result as string,
        name: file.name
      })
      
      if (newImages.length === imageFiles.length) {
        handleImagesAdded(newImages)
        target.value = ''
      }
    }
    reader.readAsDataURL(file)
  })
}

const removeImage = (id: string) => {
  images.value = images.value.filter(img => img.id !== id)
}

const handleClearCanvas = () => {
  showClearConfirm.value = true
}

const confirmClearCanvas = () => {
  collageEditorRef.value?.clearCanvas()
  showClearConfirm.value = false
}

const cancelClearCanvas = () => {
  showClearConfirm.value = false
}

const undo = () => {
  collageEditorRef.value?.undo()
}

const redo = () => {
  collageEditorRef.value?.redo()
}

const handleDragStart = (e: DragEvent, image: ImageItem) => {
  if (e.dataTransfer) {
    e.dataTransfer.setData('imageId', image.id)
    e.dataTransfer.setData('imageUrl', image.url)
    e.dataTransfer.effectAllowed = 'copy'
  }
}

const handleImageClick = (image: ImageItem) => {
  collageEditorRef.value?.addImageToCenter(image)
}

const openTextInput = () => {
  if (collageEditorRef.value) {
    collageEditorRef.value.showTextInput = true
  }
}

const updateCanvasWidth = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (collageEditorRef.value) {
    collageEditorRef.value.updateCanvasSize(Number(target.value), collageEditorRef.value.canvasSize.height)
  }
}

const updateCanvasHeight = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (collageEditorRef.value) {
    collageEditorRef.value.updateCanvasSize(collageEditorRef.value.canvasSize.width, Number(target.value))
  }
}

const updateBackgroundColor = (e: Event) => {
  const target = e.target as HTMLInputElement
  if (collageEditorRef.value) {
    collageEditorRef.value.backgroundColor = target.value
  }
}
</script>

<template>
  <div class="app">
    <header class="toolbar">
      <div class="toolbar-left">
        <div class="brand">
          <img src="./assets/favicon-32x32.png" alt="拼图工坊" class="brand-icon" />
          <span>拼图工坊</span>
        </div>
        <div class="divider"></div>
        <div class="btn-group">
          <button class="toolbar-btn" @click="undo" :disabled="!collageEditorRef?.undoStack?.length" title="撤销">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M3 7v6h6"></path>
              <path d="M21 17a9 9 0 0 0-9-9 9 9 0 0 0-6 2.3L3 13"></path>
            </svg>
          </button>
          <button class="toolbar-btn" @click="redo" :disabled="!collageEditorRef?.redoStack?.length" title="重做">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M21 7v6h-6"></path>
              <path d="M3 17a9 9 0 0 1 9-9 9 9 0 0 1 6 2.3L21 13"></path>
            </svg>
          </button>
        </div>
        <div class="divider"></div>
        <div class="btn-group">
          <button class="toolbar-btn" @click="openTextInput" title="添加文字">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="12" y1="5" x2="12" y2="19"></line>
              <line x1="5" y1="12" x2="19" y2="12"></line>
            </svg>
            <span>添加文字</span>
          </button>
          <button class="toolbar-btn" @click="triggerUpload" title="上传图片">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
              <polyline points="17 8 12 3 7 8"></polyline>
              <line x1="12" y1="3" x2="12" y2="15"></line>
            </svg>
            <span>上传图片</span>
          </button>
        </div>
      </div>
      <div class="toolbar-right">
        <div class="canvas-size">
          <span class="size-label">画布</span>
          <input
            type="number"
            :value="collageEditorRef?.canvasSize.width"
            @input="updateCanvasWidth($event)"
            min="400"
            max="1200"
            class="size-input"
          />
          <span class="size-sep">×</span>
          <input
            type="number"
            :value="collageEditorRef?.canvasSize.height"
            @input="updateCanvasHeight($event)"
            min="300"
            max="900"
            class="size-input"
          />
        </div>
        <div class="divider"></div>
        <button class="toolbar-btn danger" @click="handleClearCanvas" title="清空画布">
          <span>清空</span>
        </button>
        <button class="toolbar-btn primary" @click="collageEditorRef?.downloadCollage()" title="下载拼图">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
            <polyline points="7 10 12 15 17 10"></polyline>
            <line x1="12" y1="15" x2="12" y2="3"></line>
          </svg>
          <span>下载拼图</span>
        </button>
      </div>
    </header>

    <main class="canvas-area">
      <div class="canvas-wrapper">
        <CollageEditor ref="collageEditorRef" :images="images" />
      </div>
      <div class="float-panel">
        <div class="panel-item">
          <input
            type="color"
            :value="collageEditorRef?.backgroundColor"
            @input="updateBackgroundColor($event)"
            class="color-picker"
            title="画布背景色"
          />
        </div>
        <div class="divider-vertical"></div>
        <div class="panel-item">
          <button 
            class="background-toggle-btn"
            @click="collageEditorRef?.togglePatternBackground()"
            :class="{ 'active': collageEditorRef?.usePatternBackground }"
            :title="collageEditorRef?.usePatternBackground ? '切换到纯色背景' : '切换到点阵背景'"
          >
            <svg v-if="collageEditorRef?.usePatternBackground" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="7" height="7"></rect>
              <rect x="14" y="3" width="7" height="7"></rect>
              <rect x="14" y="14" width="7" height="7"></rect>
              <rect x="3" y="14" width="7" height="7"></rect>
            </svg>
            <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="18" height="18"></rect>
            </svg>
          </button>
        </div>
        <div class="divider-vertical"></div>
        <div class="zoom-control">
          <button class="zoom-btn" @click="collageEditorRef?.zoomOut()" title="缩小">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="5" y1="12" x2="19" y2="12"></line>
            </svg>
          </button>
          <span class="zoom-value">{{ collageEditorRef?.zoomPercent || '100%' }}</span>
          <button class="zoom-btn" @click="collageEditorRef?.zoomIn()" title="放大">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <line x1="5" y1="12" x2="19" y2="12"></line>
              <line x1="12" y1="5" x2="12" y2="19"></line>
            </svg>
          </button>
        </div>
      </div>
    </main>

    <input
      ref="uploadInputRef"
      type="file"
      accept="image/*"
      multiple
      class="hidden-upload-input"
      @change="handleUploadChange"
    />
    
    <footer class="gallery-bar">
      <div class="gallery-header">
        <span class="gallery-title">图片库</span>
        <span class="gallery-count">共 {{ images.length }} 张</span>
      </div>
      <div 
        class="gallery-scroll" 
        ref="galleryScrollRef"
        @mouseenter="handleGalleryMouseEnter"
        @mouseleave="handleGalleryMouseLeave"
      >
        <div class="gallery-list">
          <div
            v-for="image in images"
            :key="image.id"
            class="gallery-item"
            draggable="true"
            @dragstart="handleDragStart($event, image)"
            @click="handleImageClick(image)"
          >
            <img :src="image.url" :alt="image.name" />
            <button class="gallery-remove" @click.stop="removeImage(image.id)" title="删除图片">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <line x1="18" y1="6" x2="6" y2="18"></line>
                <line x1="6" y1="6" x2="18" y2="18"></line>
              </svg>
            </button>
          </div>
          <div v-if="images.length === 0" class="gallery-empty">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="7" height="7"></rect>
              <rect x="14" y="3" width="7" height="7"></rect>
              <rect x="14" y="14" width="7" height="7"></rect>
              <rect x="3" y="14" width="7" height="7"></rect>
            </svg>
            <span>暂无图片，点击上方「上传图片」按钮添加</span>
          </div>
        </div>
      </div>
    </footer>

    <div v-if="showClearConfirm" class="clear-confirm-modal" @click.self="cancelClearCanvas">
      <div class="modal-content">
        <h3>确认清空画布</h3>
        <p>确定要清空画布吗？此操作将删除所有画布内容且无法撤销。</p>
        <div class="modal-buttons">
          <button class="btn-danger" @click="confirmClearCanvas">确认清空</button>
          <button class="btn-secondary" @click="cancelClearCanvas">取消</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style lang="scss" scoped>
@use '/src/style.scss' as *;

.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: $bg;
}

.toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 42px;
  padding: 0 16px;
  background: $bg-white;
  border-bottom: 0.5px solid $border-light;
  flex-shrink: 0;
}

.toolbar-left,
.toolbar-right {
  display: flex;
  align-items: center;
  gap: 8px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  font-weight: 500;
  color: $accent;

  .brand-icon {
    width: 16px;
    height: 16px;
    border-radius: 3px;
  }
}

.divider {
  width: 1px;
  height: 18px;
  background: rgba(0, 0, 0, 0.08);
}

.btn-group {
  display: flex;
  align-items: center;
  border-radius: 4px;
  overflow: hidden;

  .toolbar-btn {
    border-radius: 0;
    border-right: none;

    &:last-child {
      border-right: 0.5px solid $border-light;
    }
  }
}

.toolbar-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 28px;
  padding: 0 12px;
  font-size: 12px;
  color: $text;
  background: transparent;
  border: 0.5px solid $border-light;
  border-radius: 4px;
  transition: all 0.15s ease;

  &:hover:not(:disabled) {
    background: $accent-medium;
    border-color: $accent-border;
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  &.primary {
    background: $accent;
    color: white;
    border-color: $accent;
    font-weight: 500;

    &:hover:not(:disabled) {
      background: $accent-hover;
      border-color: $accent-hover;
    }
  }

  &.danger {
    color: $danger;

    &:hover:not(:disabled) {
      background: $danger-light;
      border-color: $danger-border;
    }
  }

  svg {
    width: 14px;
    height: 14px;
  }
}

.canvas-size {
  display: flex;
  align-items: center;
  gap: 2px;
}

.size-label {
  font-size: 11px;
  color: $text-light;
  margin-right: 4px;
}

.size-input {
  width: 52px;
  height: 26px;
  text-align: center;
  font-size: 11px;
  border: 0.5px solid $border-light;
  border-radius: 4px;
  background: transparent;
  color: $text;

  &:focus {
    border-color: $accent;
    outline: none;
  }

  &::-webkit-outer-spin-button,
  &::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }

  &[type=number] {
    -moz-appearance: textfield;
  }
}

.size-sep {
  font-size: 11px;
  color: $text-light;
}

.canvas-area {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 28px;
  overflow: auto;
  position: relative;
}

.canvas-wrapper {
  position: relative;
}

.float-panel {
  position: fixed;
  top: 56px;
  right: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 12px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 8px;
  box-shadow: $shadow-float;
  backdrop-filter: blur(8px);
}

.divider-vertical {
  width: 1px;
  height: 20px;
  background: $border-light;
}

.color-picker {
  width: 28px;
  height: 28px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  padding: 2px;
}

.background-toggle-btn {
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 0.5px solid $border-light;
  border-radius: 4px;
  color: $text;
  cursor: pointer;
  transition: all 0.15s ease;

  &:hover {
    background: $accent-medium;
    border-color: $accent-border;
  }

  &.active {
    background: $accent-light;
    border-color: $accent-border;
    color: $accent;
  }

  svg {
    width: 16px;
    height: 16px;
  }
}

.zoom-control {
  display: flex;
  align-items: center;
  gap: 8px;
}

.zoom-btn {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 0.5px solid $border-light;
  border-radius: 4px;
  color: $text;
  transition: all 0.15s ease;

  &:hover {
    background: $accent-medium;
    border-color: $accent-border;
  }

  svg {
    width: 12px;
    height: 12px;
  }
}

.zoom-value {
  font-size: 12px;
  font-weight: 500;
  color: $text;
  min-width: 40px;
  text-align: center;
}

.gallery-bar {
  background: $bg-white;
  border-top: 0.5px solid $border-light;
  padding: 10px 16px;
  flex-shrink: 0;
  height: 120px;
}

.gallery-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.gallery-title {
  font-size: 12px;
  font-weight: 500;
  color: $text;
}

.gallery-count {
  font-size: 11px;
  color: $text-light;
}

.gallery-scroll {
  overflow-x: auto;
  overflow-y: visible;
  padding: 4px 0;
}

.gallery-list {
  display: flex;
  gap: 10px;
  min-width: 100%;
  justify-content: center;
}

.gallery-item {
  position: relative;
  width: 56px;
  height: 56px;
  border-radius: 6px;
  overflow: hidden;
  border: 0.5px solid $border-light;
  cursor: grab;
  transition: all 0.15s ease;
  flex-shrink: 0;

  &:hover {
    border-color: $accent;
    transform: translateY(-1px);
  }

  &:active {
    cursor: grabbing;
  }

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
}

.gallery-remove {
  position: absolute;
  top: 2px;
  right: 2px;
  width: 16px;
  height: 16px;
  display: none;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.6);
  border-radius: 50%;
  color: white;
  opacity: 0;
  transition: all 0.15s ease;

  .gallery-item:hover & {
    display: flex;
    opacity: 1;
  }

  &:hover {
    background: $danger;
  }

  svg {
    width: 10px;
    height: 10px;
  }
}

.gallery-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: $text-gray;
  gap: 8px;
  width: 100%;
  flex-shrink: 0;

  svg {
    width: 32px;
    height: 32px;
    opacity: 0.7;
  }

  span {
    font-size: 12px;
    text-align: center;
  }
}

.hidden-upload-input {
  display: none;
}

.clear-confirm-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;

  .modal-content {
    background: #fff;
    border-radius: 8px;
    padding: 24px;
    min-width: 320px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);

    h3 {
      font-size: 16px;
      font-weight: 600;
      color: $text;
      margin-bottom: 12px;
    }

    p {
      font-size: 14px;
      color: $text-light;
      margin-bottom: 20px;
      line-height: 1.5;
    }

    .modal-buttons {
      display: flex;
      gap: 12px;
      justify-content: flex-end;

      button {
        padding: 8px 16px;
        border-radius: 4px;
        font-size: 14px;
        cursor: pointer;
        transition: all 0.15s ease;
        border: none;
        outline: none;

        &.btn-danger {
          background: $danger;
          color: white;

          &:hover {
            background: #dc2626;
          }
        }

        &.btn-secondary {
          background: transparent;
          color: $text;
          border: 0.5px solid $border-light;

          &:hover {
            background: $accent-medium;
            border-color: $accent-border;
          }
        }
      }
    }
  }
}
</style>
