<script setup lang="ts">
import { ref, reactive, computed } from 'vue'
import type { ImageItem } from './ImageUploader.vue'

interface ElementState {
  x: number
  y: number
  width: number
  height: number
  rotation: number
}

interface CanvasElement {
  id: string
  type: 'image' | 'text'
  sourceId?: string
  url?: string
  text?: string
  x: number
  y: number
  width: number
  height: number
  rotation: number
  zIndex: number
  color?: string
  fontSize?: number
  originalState?: ElementState
}

const props = defineProps<{
  images: ImageItem[]
}>()

const STORAGE_KEYS = {
  CANVAS_SIZE: 'collage_canvas_size',
  BACKGROUND_MODE: 'collage_background_mode'
}

const loadCanvasSize = (): { width: number; height: number } => {
  try {
    const stored = localStorage.getItem(STORAGE_KEYS.CANVAS_SIZE)
    if (stored) {
      const parsed = JSON.parse(stored)
      if (
        typeof parsed === 'object' &&
        typeof parsed.width === 'number' &&
        typeof parsed.height === 'number' &&
        parsed.width >= 400 && parsed.width <= 1200 &&
        parsed.height >= 300 && parsed.height <= 900
      ) {
        return { width: parsed.width, height: parsed.height }
      }
    }
  } catch (e) {
    console.warn('Failed to load canvas size from localStorage:', e)
  }
  return { width: 800, height: 600 }
}

const saveCanvasSize = (width: number, height: number) => {
  try {
    localStorage.setItem(STORAGE_KEYS.CANVAS_SIZE, JSON.stringify({ width, height }))
  } catch (e) {
    console.warn('Failed to save canvas size to localStorage:', e)
  }
}

const loadBackgroundMode = (): boolean => {
  try {
    const stored = localStorage.getItem(STORAGE_KEYS.BACKGROUND_MODE)
    if (stored) {
      return stored === 'pattern'
    }
  } catch (e) {
    console.warn('Failed to load background mode from localStorage:', e)
  }
  return true
}

const saveBackgroundMode = (usePattern: boolean) => {
  try {
    localStorage.setItem(STORAGE_KEYS.BACKGROUND_MODE, usePattern ? 'pattern' : 'solid')
  } catch (e) {
    console.warn('Failed to save background mode to localStorage:', e)
  }
}

const canvasRef = ref<HTMLDivElement | null>(null)
const canvasElements = ref<CanvasElement[]>([])
const backgroundColor = ref('#ffffff')
const usePatternBackground = ref(loadBackgroundMode())
const savedSize = loadCanvasSize()
const canvasSize = reactive({ width: savedSize.width, height: savedSize.height })
const selectedElementId = ref<string | null>(null)
const isDraggingCanvas = ref(false)
const dragOffset = reactive({ x: 0, y: 0 })
const isResizing = ref(false)
const resizeDirection = ref('')
const isRotating = ref(false)
const rotationStartAngle = ref(0)
const rotationStartMouseAngle = ref(0)
const showDeleteConfirm = ref(false)
const deleteTargetId = ref<string | null>(null)
const showTextInput = ref(false)
const newText = ref('')
const zoom = ref(1)
const undoStack = ref<string[]>([])
const redoStack = ref<string[]>([])

const generateElementId = () => `canvas-el-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`

const updateCanvasSize = (width: number, height: number) => {
  canvasSize.width = Math.max(400, Math.min(1200, width))
  canvasSize.height = Math.max(300, Math.min(900, height))
  saveCanvasSize(canvasSize.width, canvasSize.height)
}

const zoomPercent = computed(() => Math.round(zoom.value * 100) + '%')

const saveCanvasState = () => {
  undoStack.value.push(JSON.stringify(canvasElements.value))
  redoStack.value = []
}

const undo = () => {
  if (undoStack.value.length > 0) {
    redoStack.value.push(JSON.stringify(canvasElements.value))
    canvasElements.value = JSON.parse(undoStack.value.pop()!)
  }
}

const redo = () => {
  if (redoStack.value.length > 0) {
    undoStack.value.push(JSON.stringify(canvasElements.value))
    canvasElements.value = JSON.parse(redoStack.value.pop()!)
  }
}

const zoomIn = () => {
  zoom.value = Math.min(zoom.value + 0.1, 2)
}

const zoomOut = () => {
  zoom.value = Math.max(zoom.value - 0.1, 0.5)
}

const handleDragOver = (e: DragEvent) => {
  e.preventDefault()
  if (e.dataTransfer) {
    e.dataTransfer.dropEffect = 'copy'
  }
}

const handleDrop = (e: DragEvent) => {
  e.preventDefault()
  
  const imageId = e.dataTransfer?.getData('imageId')
  const imageUrl = e.dataTransfer?.getData('imageUrl')
  
  if (!imageId || !imageUrl || !canvasRef.value) return
  
  const rect = canvasRef.value.getBoundingClientRect()
  const x = (e.clientX - rect.left) / zoom.value - 100
  const y = (e.clientY - rect.top) / zoom.value - 100
  
  const img = new Image()
  img.onload = () => {
    let width = 200
    let height = 200
    
    if (img.width > img.height) {
      height = (200 / img.width) * img.height
    } else {
      width = (200 / img.height) * img.width
    }
    
    const finalX = Math.max(0, Math.min(x, canvasSize.width - width))
    const finalY = Math.max(0, Math.min(y, canvasSize.height - height))
    
    saveCanvasState()
    canvasElements.value.push({
      id: generateElementId(),
      type: 'image',
      sourceId: imageId,
      url: imageUrl,
      x: finalX,
      y: finalY,
      width,
      height,
      rotation: 0,
      zIndex: canvasElements.value.length + 1,
      originalState: { x: finalX, y: finalY, width, height, rotation: 0 }
    })
  }
  img.src = imageUrl
}

const addImageToCenter = (image: ImageItem) => {
  const img = new Image()
  img.onload = () => {
    let width = 200
    let height = 200
    
    if (img.width > img.height) {
      height = (200 / img.width) * img.height
    } else {
      width = (200 / img.height) * img.width
    }
    
    const x = (canvasSize.width - width) / 2
    const y = (canvasSize.height - height) / 2
    
    const finalX = Math.max(0, Math.min(x, canvasSize.width - width))
    const finalY = Math.max(0, Math.min(y, canvasSize.height - height))
    
    saveCanvasState()
    const newElement: CanvasElement = {
      id: generateElementId(),
      type: 'image',
      sourceId: image.id,
      url: image.url,
      x: finalX,
      y: finalY,
      width,
      height,
      rotation: 0,
      zIndex: canvasElements.value.length + 1,
      originalState: { x: finalX, y: finalY, width, height, rotation: 0 }
    }
    
    canvasElements.value.push(newElement)
    selectedElementId.value = newElement.id
  }
  img.src = image.url
}

const addTextElement = () => {
  if (!newText.value.trim()) return
  
  const fontSize = 32
  const textWidth = newText.value.length * fontSize * 0.6
  const height = fontSize * 1.5
  
  const x = (canvasSize.width - textWidth) / 2
  const y = (canvasSize.height - height) / 2
  
  saveCanvasState()
  const newElement: CanvasElement = {
    id: generateElementId(),
    type: 'text',
    text: newText.value,
    x: Math.max(0, Math.min(x, canvasSize.width - textWidth)),
    y: Math.max(0, Math.min(y, canvasSize.height - height)),
    width: Math.max(100, textWidth),
    height,
    rotation: 0,
    zIndex: canvasElements.value.length + 1,
    color: '#333333',
    fontSize
  }
  
  canvasElements.value.push(newElement)
  selectedElementId.value = newElement.id
  newText.value = ''
  showTextInput.value = false
}

const handleElementMouseDown = (e: MouseEvent, element: CanvasElement) => {
  e.preventDefault()
  e.stopPropagation()
  selectedElementId.value = element.id
  isDraggingCanvas.value = true
  
  const rect = canvasRef.value?.getBoundingClientRect() || { left: 0, top: 0 }
  dragOffset.x = (e.clientX - rect.left) / zoom.value - element.x
  dragOffset.y = (e.clientY - rect.top) / zoom.value - element.y
  
  element.zIndex = canvasElements.value.length + 1
}

const handleElementDrag = (e: MouseEvent) => {
  if (!isDraggingCanvas.value || !selectedElementId.value || !canvasRef.value) return
  
  const rect = canvasRef.value.getBoundingClientRect()
  const element = canvasElements.value.find(el => el.id === selectedElementId.value)
  
  if (element) {
    element.x = Math.max(-element.width / 2, Math.min((e.clientX - rect.left) / zoom.value - dragOffset.x, canvasSize.width - element.width / 2))
    element.y = Math.max(-element.height / 2, Math.min((e.clientY - rect.top) / zoom.value - dragOffset.y, canvasSize.height - element.height / 2))
  }
}

const handleElementDragEnd = () => {
  isDraggingCanvas.value = false
  isResizing.value = false
  isRotating.value = false
}

const handleResizeStart = (e: MouseEvent, direction: string) => {
  e.preventDefault()
  e.stopPropagation()
  isResizing.value = true
  resizeDirection.value = direction
}

const handleResize = (e: MouseEvent) => {
  if (!isResizing.value || !selectedElementId.value || !canvasRef.value) return
  
  const rect = canvasRef.value.getBoundingClientRect()
  const element = canvasElements.value.find(el => el.id === selectedElementId.value)
  
  if (!element) return
  
  const mouseX = (e.clientX - rect.left) / zoom.value
  const mouseY = (e.clientY - rect.top) / zoom.value
  const aspectRatio = element.width / element.height
  
  switch (resizeDirection.value) {
    case 'se':
      if (e.shiftKey) {
        const delta = Math.max(mouseX - element.x, mouseY - element.y)
        element.width = Math.max(50, delta)
        element.height = Math.max(30, delta / aspectRatio)
      } else {
        element.width = Math.max(50, mouseX - element.x)
        element.height = Math.max(30, mouseY - element.y)
      }
      break
    case 'sw':
      if (e.shiftKey) {
        const delta = Math.max(element.x + element.width - mouseX, mouseY - element.y)
        element.width = Math.max(50, delta)
        element.height = Math.max(30, delta / aspectRatio)
        element.x = mouseX
      } else {
        element.width = Math.max(50, element.x + element.width - mouseX)
        element.x = mouseX
        element.height = Math.max(30, mouseY - element.y)
      }
      break
    case 'ne':
      if (e.shiftKey) {
        const delta = Math.max(mouseX - element.x, element.y + element.height - mouseY)
        element.width = Math.max(50, delta)
        element.height = Math.max(30, delta / aspectRatio)
        element.y = mouseY
      } else {
        element.width = Math.max(50, mouseX - element.x)
        element.height = Math.max(30, element.y + element.height - mouseY)
        element.y = mouseY
      }
      break
    case 'nw':
      if (e.shiftKey) {
        const delta = Math.max(element.x + element.width - mouseX, element.y + element.height - mouseY)
        element.width = Math.max(50, delta)
        element.height = Math.max(30, delta / aspectRatio)
        element.x = mouseX
        element.y = mouseY
      } else {
        element.width = Math.max(50, element.x + element.width - mouseX)
        element.height = Math.max(30, element.y + element.height - mouseY)
        element.x = mouseX
        element.y = mouseY
      }
      break
    case 'n':
      element.height = Math.max(30, element.y + element.height - mouseY)
      element.y = mouseY
      break
    case 's':
      element.height = Math.max(30, mouseY - element.y)
      break
    case 'e':
      element.width = Math.max(50, mouseX - element.x)
      break
    case 'w':
      element.width = Math.max(50, element.x + element.width - mouseX)
      element.x = mouseX
      break
  }
}

const handleRotateStart = (e: MouseEvent) => {
  e.preventDefault()
  e.stopPropagation()
  isRotating.value = true
  
  const element = canvasElements.value.find(el => el.id === selectedElementId.value)
  if (!element || !canvasRef.value) return
  
  const rect = canvasRef.value.getBoundingClientRect()
  const centerX = element.x + element.width / 2
  const centerY = element.y + element.height / 2
  const mouseX = (e.clientX - rect.left) / zoom.value
  const mouseY = (e.clientY - rect.top) / zoom.value
  
  rotationStartAngle.value = element.rotation
  rotationStartMouseAngle.value = Math.atan2(mouseY - centerY, mouseX - centerX) * (180 / Math.PI)
}

const handleRotate = (e: MouseEvent) => {
  if (!isRotating.value || !selectedElementId.value || !canvasRef.value) return
  
  const rect = canvasRef.value.getBoundingClientRect()
  const element = canvasElements.value.find(el => el.id === selectedElementId.value)
  
  if (!element) return
  
  const centerX = element.x + element.width / 2
  const centerY = element.y + element.height / 2
  const mouseX = (e.clientX - rect.left) / zoom.value
  const mouseY = (e.clientY - rect.top) / zoom.value
  
  const currentMouseAngle = Math.atan2(mouseY - centerY, mouseX - centerX) * (180 / Math.PI)
  const deltaAngle = currentMouseAngle - rotationStartMouseAngle.value
  element.rotation = ((rotationStartAngle.value + deltaAngle) % 360 + 360) % 360
}

const confirmDelete = (id: string) => {
  deleteTargetId.value = id
  showDeleteConfirm.value = true
}

const cancelDelete = () => {
  showDeleteConfirm.value = false
  deleteTargetId.value = null
}

const deleteElement = () => {
  if (!deleteTargetId.value) return
  
  saveCanvasState()
  canvasElements.value = canvasElements.value.filter(el => el.id !== deleteTargetId.value)
  if (selectedElementId.value === deleteTargetId.value) {
    selectedElementId.value = null
  }
  showDeleteConfirm.value = false
  deleteTargetId.value = null
}

const resetElement = (id: string) => {
  const element = canvasElements.value.find(el => el.id === id)
  if (!element || !element.originalState) return
  
  element.x = element.originalState.x
  element.y = element.originalState.y
  element.width = element.originalState.width
  element.height = element.originalState.height
  element.rotation = element.originalState.rotation
}

const clearCanvas = () => {
  saveCanvasState()
  canvasElements.value = []
  selectedElementId.value = null
}

const downloadCollage = async () => {
  const canvas = document.createElement('canvas')
  canvas.width = canvasSize.width
  canvas.height = canvasSize.height
  
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  
  ctx.fillStyle = backgroundColor.value
  ctx.fillRect(0, 0, canvas.width, canvas.height)
  
  const sortedElements = [...canvasElements.value].sort((a, b) => a.zIndex - b.zIndex)
  
  for (const element of sortedElements) {
    ctx.save()
    ctx.translate(element.x + element.width / 2, element.y + element.height / 2)
    ctx.rotate((element.rotation * Math.PI) / 180)
    
    if (element.type === 'image' && element.url) {
      const imgObj = new Image()
      imgObj.src = element.url
      await new Promise(resolve => { imgObj.onload = resolve })
      ctx.drawImage(imgObj, -element.width / 2, -element.height / 2, element.width, element.height)
    } else if (element.type === 'text' && element.text) {
      ctx.font = `${element.fontSize || 32}px Arial, sans-serif`
      ctx.fillStyle = element.color || '#333333'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText(element.text, 0, 0)
    }
    
    ctx.restore()
  }
  
  const link = document.createElement('a')
  link.download = `collage-${Date.now()}.png`
  link.href = canvas.toDataURL('image/png')
  link.click()
}

defineExpose({
  addImageToCenter,
  clearCanvas,
  downloadCollage,
  showTextInput,
  canvasSize,
  backgroundColor,
  usePatternBackground,
  updateCanvasSize,
  togglePatternBackground: () => { 
    usePatternBackground.value = !usePatternBackground.value 
    saveBackgroundMode(usePatternBackground.value)
  },
  zoomIn,
  zoomOut,
  zoomPercent,
  undo,
  redo,
  undoStack,
  redoStack
})
</script>

<template>
  <div 
    ref="canvasRef"
    class="canvas"
    :class="{ 'pattern-background': usePatternBackground }"
    :style="{
      width: canvasSize.width + 'px',
      height: canvasSize.height + 'px',
      backgroundColor,
      transform: `scale(${zoom})`,
      transformOrigin: 'center center'
    }"
    @dragover="handleDragOver"
    @drop="handleDrop"
    @mousemove="(e) => { handleElementDrag(e); handleResize(e); handleRotate(e); }"
    @mouseup="handleElementDragEnd"
    @mouseleave="handleElementDragEnd"
    @click="selectedElementId = null"
  >
    <div
      v-for="element in canvasElements"
      :key="element.id"
      class="canvas-element"
      :class="{ 'is-selected': selectedElementId === element.id }"
      :style="{
        left: element.x + 'px',
        top: element.y + 'px',
        width: element.width + 'px',
        height: element.height + 'px',
        zIndex: element.zIndex
      }"
      @mousedown.stop="handleElementMouseDown($event, element)"
      @click.stop="selectedElementId = element.id"
    >
      <div class="element-inner" :style="{ transform: `rotate(${element.rotation}deg)` }">
        <img v-if="element.type === 'image' && element.url" :src="element.url" draggable="false" />
        <span v-else-if="element.type === 'text' && element.text" class="canvas-text" :style="{ color: element.color, fontSize: element.fontSize + 'px' }">
          {{ element.text }}
        </span>
      </div>
      
      <div v-if="selectedElementId === element.id" class="selection-controls">
        <div class="resize-handle nw" @mousedown.stop="handleResizeStart($event, 'nw')"></div>
        <div class="resize-handle ne" @mousedown.stop="handleResizeStart($event, 'ne')"></div>
        <div class="resize-handle sw" @mousedown.stop="handleResizeStart($event, 'sw')"></div>
        <div class="resize-handle se" @mousedown.stop="handleResizeStart($event, 'se')"></div>
        
        <div class="resize-handle edge n" @mousedown.stop="handleResizeStart($event, 'n')"></div>
        <div class="resize-handle edge s" @mousedown.stop="handleResizeStart($event, 's')"></div>
        <div class="resize-handle edge w" @mousedown.stop="handleResizeStart($event, 'w')"></div>
        <div class="resize-handle edge e" @mousedown.stop="handleResizeStart($event, 'e')"></div>
        
        <div class="rotate-handle" @mousedown.stop="handleRotateStart($event)" title="拖动旋转">
          <div class="rotate-dot"></div>
          <div class="rotate-line"></div>
        </div>
        
        <button class="delete-btn" @click.stop="confirmDelete(element.id)" title="删除">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3">
            <line x1="18" y1="6" x2="6" y2="18"></line>
            <line x1="6" y1="6" x2="18" y2="18"></line>
          </svg>
        </button>
        
        <button class="reset-btn" @click.stop="resetElement(element.id)" title="重置为原始状态">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"></path>
            <path d="M3 3v5h5"></path>
          </svg>
        </button>
      </div>
    </div>

    <div v-if="canvasElements.length === 0" class="canvas-empty">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
        <rect x="3" y="3" width="7" height="7"></rect>
        <rect x="14" y="3" width="7" height="7"></rect>
        <rect x="14" y="14" width="7" height="7"></rect>
        <rect x="3" y="14" width="7" height="7"></rect>
      </svg>
      <p>从下方图片库拖拽图片到这里</p>
      <p class="hint">点击图片可快速添加到画布中心</p>
    </div>
  </div>

  <div v-if="showTextInput" class="text-input-modal" @click.self="showTextInput = false">
    <div class="text-input-content">
      <input
        type="text"
        v-model="newText"
        placeholder="输入文字..."
        class="text-input-field"
        @keyup.enter="addTextElement"
        autofocus
      />
      <div class="text-input-buttons">
        <button class="btn-primary" @click="addTextElement">添加</button>
        <button class="btn-secondary" @click="showTextInput = false">取消</button>
      </div>
    </div>
  </div>

  <div v-if="showDeleteConfirm" class="delete-confirm-modal" @click.self="cancelDelete">
    <div class="modal-content">
      <h3>确认删除</h3>
      <p>确定要删除这个元素吗？此操作无法撤销。</p>
      <div class="modal-buttons">
        <button class="btn-danger" @click="deleteElement">确认删除</button>
        <button class="btn-secondary" @click="cancelDelete">取消</button>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
@use '../style.scss' as *;
$accent: #534AB7;
$danger: #ef4444;
$rotate-color: #3B82F6;

.canvas {
  position: relative;
  background-color: #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  -webkit-user-select: none;
  user-select: none;

  &.pattern-background {
    background-image: 
      linear-gradient(45deg, #e8e8e8 25%, transparent 25%),
      linear-gradient(-45deg, #e8e8e8 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, #e8e8e8 75%),
      linear-gradient(-45deg, transparent 75%, #e8e8e8 75%);
    background-size: 16px 16px;
    background-position: 0 0, 0 8px, 8px -8px, -8px 0px;
  }
}

.canvas-empty {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  color: $text-gray;

  svg {
    width: 48px;
    height: 48px;
    margin-bottom: 12px;
    opacity: 0.7;
  }

  p {
    font-size: 13px;
    margin-bottom: 6px;
  }

  .hint {
    font-size: 11px;
    opacity: 0.85;
  }
}

.canvas-element {
  position: absolute;
  cursor: move;
  transition: box-shadow 0.15s ease;
  -webkit-user-select: none;
  user-select: none;

  &.is-selected {
    outline: 2px solid $accent;
  }

  .element-inner {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    pointer-events: none;
  }

  .canvas-text {
    font-family: Arial, sans-serif;
    font-weight: bold;
    text-align: center;
    line-height: 1.2;
    padding: 8px;
    word-break: break-word;
    max-width: 100%;
  }
}

.selection-controls {
  position: absolute;
  inset: -6px;
  pointer-events: none;
}

.resize-handle {
  position: absolute;
  width: 12px;
  height: 12px;
  background: #fff;
  border: 1.5px solid $accent;
  border-radius: 50%;
  cursor: grab;
  pointer-events: auto;
  transition: all 0.15s ease;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.15);

  &:hover {
    transform: scale(1.3);
    background: $accent;
    border-color: $accent;
  }

  &.nw { top: -6px; left: -6px; cursor: nw-resize; }
  &.ne { top: -6px; right: -6px; cursor: ne-resize; }
  &.sw { bottom: -6px; left: -6px; cursor: sw-resize; }
  &.se { bottom: -6px; right: -6px; cursor: se-resize; }
  
  &.edge {
    width: 16px;
    height: 16px;
    
    &.n { 
      top: -8px; 
      left: 50%; 
      transform: translateX(-50%); 
      cursor: n-resize; 
    }
    &.s { 
      bottom: -8px; 
      left: 50%; 
      transform: translateX(-50%); 
      cursor: s-resize; 
    }
    &.w { 
      left: -8px; 
      top: 50%; 
      transform: translateY(-50%); 
      cursor: w-resize; 
    }
    &.e { 
      right: -8px; 
      top: 50%; 
      transform: translateY(-50%); 
      cursor: e-resize; 
    }
    
    &:hover {
      transform: translateX(-50%) scale(1.2);
      background: rgba(83, 74, 183, 0.1);
    }
    
    &.w:hover, &.e:hover {
      transform: translateY(-50%) scale(1.2);
    }
  }
}

.rotate-handle {
  position: absolute;
  top: -28px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  pointer-events: auto;
  cursor: grab;

  &:active {
    cursor: grabbing;
  }

  .rotate-dot {
    width: 12px;
    height: 12px;
    background: $rotate-color;
    border: 2px solid #fff;
    border-radius: 50%;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
    transition: all 0.15s ease;

    &:hover {
      transform: scale(1.3);
    }
  }

  .rotate-line {
    width: 2px;
    height: 10px;
    background: $rotate-color;
    margin-top: 2px;
  }
}

.delete-btn {
  position: absolute;
  top: -27px;
  right: -9px;
  width: 18px;
  height: 18px;
  background: $danger;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  pointer-events: auto;
  border: none;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: all 0.15s ease;

  &:hover {
    transform: scale(1.2);
  }

  svg {
    width: 10px;
    height: 10px;
  }
}

.reset-btn {
  position: absolute;
  top: -27px;
  right: 20px;
  width: 18px;
  height: 18px;
  background: #6B7280;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  pointer-events: auto;
  border: none;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: all 0.15s ease;

  &:hover {
    transform: scale(1.2);
    background: #4B5563;
  }

  svg {
    width: 10px;
    height: 10px;
  }
}

.text-input-modal {
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
}

.text-input-content {
  background: #fff;
  border-radius: 8px;
  padding: 20px;
  width: 90%;
  max-width: 360px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.text-input-field {
  padding: 10px 12px;
  border: 0.5px solid rgba(0, 0, 0, 0.1);
  border-radius: 6px;
  font-size: 14px;
  width: 100%;

  &:focus {
    outline: none;
    border-color: $accent;
  }
}

.text-input-buttons {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}

.btn-primary {
  padding: 8px 16px;
  background: $accent;
  color: #fff;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.15s ease;

  &:hover {
    background: #4338ca;
  }
}

.btn-secondary {
  padding: 8px 16px;
  background: transparent;
  color: #666;
  border-radius: 6px;
  font-size: 12px;
  border: 0.5px solid rgba(0, 0, 0, 0.1);
  transition: all 0.15s ease;

  &:hover {
    background: rgba(0, 0, 0, 0.05);
  }
}

.btn-danger {
  padding: 8px 16px;
  background: $danger;
  color: #fff;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.15s ease;

  &:hover {
    background: #dc2626;
  }
}

.delete-confirm-modal {
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
