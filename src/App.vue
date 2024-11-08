<template>
  <div class="app-container">
    <!-- 添加左侧素材选择区域 -->
    <div class="resource-panel">
      <!-- 背景素材 -->
      <a-card title="背景素材" :bordered="false" size="small">
        <div class="resource-grid">
          <div
            v-for="bg in backgroundList"
            :key="bg.id"
            class="resource-item"
            :class="{ active: backgroundUrl === bg.path }"
            @click="selectBackground(bg)"
          >
            <img :src="bg.path" class="resource-preview" />
            <div class="resource-name">{{ bg.name }}</div>
          </div>
        </div>
      </a-card>

      <!-- 视频素材 -->
      <a-card title="视频素材" :bordered="false" size="small" style="margin-top: 16px;">
        <div class="resource-grid">
          <div
            v-for="video in videoList"
            :key="video.id"
            class="resource-item"
            :class="{ active: videoUrl === video.path }"
            @click="selectVideo(video)"
          >
            <div class="video-preview">
              <img v-if="video.thumbnail" :src="video.thumbnail" class="resource-preview" />
              <video-camera-outlined v-else />
            </div>
            <div class="resource-name">{{ video.name }}</div>
          </div>
        </div>
      </a-card>

      <!-- 贴片素材 -->
      <a-card title="贴片素材" :bordered="false" size="small" style="margin-top: 16px;">
        <div class="resource-grid">
          <div
            v-for="sticker in stickerList"
            :key="sticker.id"
            class="resource-item"
            :class="{ disabled: isStrickerUsed(sticker.id) }"
            @click="addSticker(sticker)"
          >
            <img :src="sticker.path" class="resource-preview" />
            <div class="resource-name">{{ sticker.name }}</div>
          </div>
        </div>
      </a-card>
    </div>

    <!-- 原有的画布区域 -->
    <div class="canvas-container">
      <div class="canvas" ref="canvasRef">
        <!-- 背景图层 -->
        <img
          v-if="backgroundUrl"
          :src="backgroundUrl"
          class="background-element"
          :style="{ zIndex: getLayerIndex('background') }"
        />
        
        <!-- 视频层 -->
        <video 
          v-if="videoUrl" 
          :src="videoUrl" 
          ref="videoRef"
          class="video-element"
          :style="{ zIndex: getLayerIndex('video') }"
          @loadeddata="handleVideoLoad"
        ></video>
        
        <!-- 贴层 -->
        <Vue3DraggableResizable
          v-for="(sticker, index) in stickers"
          :key="sticker.id"
          :w="sticker.width"
          :h="sticker.height"
          :x="sticker.x"
          :y="sticker.y"
          :parent="true"
          :prevent-deactivation="true"
          :active="activeStickerId === sticker.id"
          :draggable="true"
          :resizable="true"
          :min-width="20"
          :min-height="20"
          :snap="true"
          :snap-tolerance="5"
          :handles="['tl', 'tr', 'bl', 'br']"
          :lock-aspect-ratio="true"
          @activated="handleStickerActivated(sticker.id)"
          @deactivated="handleStickerDeactivated"
          @dragging="(pos) => handleDragResize(index, pos, 'drag')"
          @dragstop="(pos) => handleDragResize(index, pos, 'drag')"
          @resizing="(rect) => handleResize(index, rect)"
          @resizestop="(rect) => handleResize(index, rect)"
          class="sticker-container"
          :style="{ 
            zIndex: getLayerIndex(sticker.id),
            visibility: isLayerVisible(sticker.id) ? 'visible' : 'hidden'
          }"
        >
          <div class="sticker-wrapper">
            <img 
              :src="sticker.url" 
              class="sticker-image" 
              draggable="false"
            />
          </div>
        </Vue3DraggableResizable>
      </div>
    </div>

    <!-- 原有的控制面板 -->
    <div class="control-panel">
      <!-- 图层管理面板 -->
      <a-card title="图层管理" :bordered="false" size="small">
        <a-list
          :data-source="layers"
          class="layer-list"
        >
          <template #renderItem="{ item, index }">
            <a-list-item>
              <div class="layer-item">
                <a-space>
                  <div class="layer-preview">
                    <img v-if="item.type === 'sticker' || item.type === 'background'" :src="item.url" />
                    <video-camera-outlined v-else-if="item.type === 'video'" />
                  </div>
                  <span>{{ getLayerName(item) }}</span>
                </a-space>
                <a-space>
                  <a-button 
                    type="text" 
                    :disabled="index === 0"
                    @click="moveLayer(index, 'up')"
                  >
                    <up-outlined />
                  </a-button>
                  <a-button 
                    type="text" 
                    :disabled="index === layers.length - 1"
                    @click="moveLayer(index, 'down')"
                  >
                    <down-outlined />
                  </a-button>
                </a-space>
              </div>
            </a-list-item>
          </template>
        </a-list>
      </a-card>

      <!-- 导出控制面板 -->
      <a-card title="导出控制" :bordered="false" size="small" style="margin-top: 16px;">
        <div class="export-controls">
          <a-button 
            type="primary" 
            :disabled="!videoUrl || isExporting" 
            @click="startExport"
            block
          >
            {{ isExporting ? '导出中...' : '导出视频' }}
          </a-button>
        </div>
      </a-card>
    </div>

    <!-- 进度弹窗 -->
    <a-modal
      v-model:visible="showExportProgress"
      title="导出进度"
      :closable="false"
      :maskClosable="false"
      :footer="null"
      centered
    >
      <div class="export-progress">
        <a-progress :percent="exportProgress" status="active" />
        <div class="export-info">
          <p>预计剩余时间: {{ remainingTime }}</p>
          <p>资源加载进度:</p>
          <ul class="resource-loading-list">
            <li>
              <check-circle-outlined :style="{ color: backgroundUrl ? '#52c41a' : '#d9d9d9' }" />
              背景图片
            </li>
            <li>
              <check-circle-outlined :style="{ color: loadedStickers.size === stickers.length ? '#52c41a' : '#d9d9d9' }" />
              贴片素材 ({{ loadedStickers.size }}/{{ stickers.length }})
            </li>
          </ul>
        </div>
      </div>
    </a-modal>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue'
import Vue3DraggableResizable from 'vue3-draggable-resizable'
import 'vue3-draggable-resizable/dist/Vue3DraggableResizable.css'
import { 
  VideoCameraOutlined,
  UpOutlined,
  DownOutlined,
  CheckCircleOutlined 
} from '@ant-design/icons-vue'
import { message } from 'ant-design-vue'

// 添加 stickerStates 的声明
const videoUrl = ref('')
const videoRef = ref(null)
const canvasRef = ref(null)
const stickers = reactive([])

// 新增导出相关的变量
const isExporting = ref(false)
const mediaRecorder = ref(null)
const recordedChunks = ref([])
const exportCanvas = ref(null)
const exportContext = ref(null)

const layers = reactive([])
const exportProgress = ref(0)
const remainingTime = ref('计算��...')
const startTime = ref(0)

// 添加新的响应式变量来跟踪激活的
const activeStickerId = ref(null)

// 添加一个用于缓存片的 Map
const stickerImagesCache = new Map()

// 添加新的响应式变量
const showExportProgress = ref(false) // 控制进示
const loadedStickers = ref(new Set()) // 跟踪已加载的贴片

// 的变量
const backgroundUrl = ref('')

// 资源预加载函数
const preloadResources = async () => {
  console.log('开始预加载资源')
  
  const loadImage = (url, id) => {
    return new Promise((resolve, reject) => {
      const img = new Image()
      img.onload = () => {
        stickerImagesCache.set(id, img)
        if (id !== 'background') {
          loadedStickers.value.add(id)
        }
        console.log(`图片加载成功: ${id}`, {
          尺寸: { w: img.naturalWidth, h: img.naturalHeight }
        })
        resolve(img)
      }
      img.onerror = () => reject(new Error(`图片加载失败: ${id}`))
      img.src = url
    })
  }
  
  try {
    const loadPromises = []
    
    // 加载背景图（如果有）
    if (backgroundUrl.value) {
      loadPromises.push(loadImage(backgroundUrl.value, 'background'))
    }
    
    // 加载所有贴片
    loadPromises.push(...stickers.map(sticker => 
      loadImage(sticker.url, sticker.id)
    ))
    
    await Promise.all(loadPromises)
    
    console.log('所有资源加载成', {
      背景图: !!backgroundUrl.value,
      贴片数量: stickers.length,
      已加载数量: loadedStickers.value.size,
      缓存数量: stickerImagesCache.size
    })
  } catch (error) {
    console.error('资源加载失败:', error)
    throw error
  }
}

// 修改视频加载处理
const handleVideoLoad = () => {
  if (videoRef.value) {
    const video = videoRef.value
    video.width = exportCanvas.value.width
    video.height = exportCanvas.value.height
    
    // 确保视频保持比例
    video.style.objectFit = 'contain'
    
    console.log('视频加载完成:', {
      videoSize: {
        width: video.videoWidth,
        height: video.videoHeight
      },
      canvasSize: {
        width: exportCanvas.value.width,
        height: exportCanvas.value.height
      }
    })
  }
}

// 图层管理相关方法
const getLayerIndex = (id) => {
  const index = layers.findIndex(layer => layer.id === id)
  // 返回基于图层置的 z-index，数值越大越靠前
  return index === -1 ? 1 : (layers.length - index) * 100
}

const isLayerVisible = (id) => {
  const layer = layers.find(layer => layer.id === id)
  return layer?.visible ?? true // 默认为可见
}

const moveLayer = (index, direction) => {
  const layer = layers[index]
  
  // 只限制背景图层
  if (layer.type === 'background') {
    message.warning('背景图层固定在底部，不能移动')
    return
  }
  
  // 计算新的位置
  const newIndex = direction === 'up' ? index - 1 : index + 1
  
  // 检查边界条件
  if (newIndex < 0 || newIndex >= layers.length) return
  
  // 检查是否会移动到背景图层的位置
  const targetLayer = layers[newIndex]
  if (targetLayer.type === 'background') {
    message.warning('无法移动到背景图层的位置')
    return
  }
  
  // 执行图层移动
  [layers[index], layers[newIndex]] = [layers[newIndex], layers[index]]
  
  console.log('图层顺序已更新:', layers.map(l => ({
    id: l.id,
    type: l.type,
    zIndex: getLayerIndex(l.id)
  })))
}

// 更导出
const updateExportProgress = () => {
  if (!videoRef.value || !isExporting.value) return
  
  const currentTime = videoRef.value.currentTime
  const videoDuration = videoRef.value.duration
  const progress = (currentTime / videoDuration) * 100
  exportProgress.value = Math.round(progress)
  
  // 计算剩余时间
  const elapsed = (Date.now() - startTime.value) / 1000
  const rate = currentTime / elapsed
  const remaining = (videoDuration - currentTime) / rate
  remainingTime.value = `${Math.round(remaining)}秒`
  
  if (progress < 100) {
    requestAnimationFrame(updateExportProgress)
  }
}

// 修改开始导出法
const startExport = async () => {
  if (!videoUrl.value || isExporting.value) return
  
  try {
    console.log('开始导出')
    isExporting.value = true
    showExportProgress.value = true
    exportProgress.value = 0
    startTime.value = Date.now()
    recordedChunks.value = []
    
    // 初始化导出画布
    initExportCanvas()
    
    // 预加载所有资源
    await preloadResources()
    
    // 创建媒体流
    const stream = exportCanvas.value.captureStream(30) // 设置帧率为30fps
    
    // 添加音频轨道（如果有）
    if (videoRef.value) {
      const audioTracks = videoRef.value.captureStream().getAudioTracks()
      if (audioTracks.length > 0) {
        stream.addTrack(audioTracks[0])
      }
    }
    
    // 配置媒体录制器
    mediaRecorder.value = new MediaRecorder(stream, {
      mimeType: getSupportedMimeType(),
      videoBitsPerSecond: 8000000 // 设置视频比特率
    })
    
    // 设置数据处理
    mediaRecorder.value.ondataavailable = (event) => {
      if (event.data.size > 0) {
        recordedChunks.value.push(event.data)
      }
    }
    
    // 设置录制完成处理
    mediaRecorder.value.onstop = () => {
      const blob = new Blob(recordedChunks.value, { type: 'video/webm' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'output.webm'
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      URL.revokeObjectURL(url)
      
      isExporting.value = false
      showExportProgress.value = false
      exportProgress.value = 0
      message.success('视频导出功')
    }
    
    // 开始录制前确保视频准备就绪
    videoRef.value.currentTime = 0
    await videoRef.value.play()
    
    // 开始录制
    mediaRecorder.value.start(1000) // 每秒生成一个数据块
    
    // 开始渲染循环
    renderFrame()
    updateExportProgress()
    
  } catch (error) {
    console.error('导出失败:', error)
    message.error('导出失败：' + error.message)
    await stopExport()
  }
}

// 修改渲染帧函数，添加更多的错误处理和日志
const renderFrame = async () => {
  if (!isExporting.value || !videoRef.value) return
  
  try {
    const ctx = exportContext.value
    ctx.clearRect(0, 0, exportCanvas.value.width, exportCanvas.value.height)
    
    // 计算缩放比例
    const displayCanvas = canvasRef.value
    const ratio = {
      x: exportCanvas.value.width / displayCanvas.clientWidth,
      y: exportCanvas.value.height / displayCanvas.clientHeight
    }

    // 重要：反转图层数组以确保正确的渲染顺序
    // 因为 layers 数组中，索引越小表示图层越靠前
    const renderLayers = [...layers].reverse()

    // 按照图层顺序渲染
    for (const layer of renderLayers) {
      // 检查图层是否可见
      if (!layer.visible) {
        console.log(`跳过不可见图层: ${layer.id}`)
        continue
      }

      console.log(`渲染图层: ${layer.id}, 类型: ${layer.type}, z-index: ${getLayerIndex(layer.id)}`)

      switch (layer.type) {
        case 'background':
          const bgImg = stickerImagesCache.get('background')
          if (bgImg?.complete) {
            ctx.drawImage(bgImg, 0, 0, exportCanvas.value.width, exportCanvas.value.height)
          }
          break;

        case 'video':
          if (videoRef.value) {
            ctx.drawImage(
              videoRef.value,
              0,
              0,
              exportCanvas.value.width,
              exportCanvas.value.height
            )
          }
          break;

        case 'sticker':
          const sticker = stickers.find(s => s.id === layer.id)
          if (!sticker) continue

          const img = stickerImagesCache.get(sticker.id)
          if (!img?.complete) continue

          // 获取画布尺寸
          const canvasWidth = canvasRef.value.clientWidth
          const canvasHeight = canvasRef.value.clientHeight
          
          // 使用保存的缩放比例（如果没有则使用当前尺寸计算���
          const scale = sticker.scale || (sticker.width / sticker.originalWidth)
          
          // 计算导出尺寸（应用缩放比例）
          const exportWidth = Math.round(sticker.originalWidth * scale * (exportCanvas.value.width / canvasWidth))
          const exportHeight = Math.round(sticker.originalHeight * scale * (exportCanvas.value.height / canvasHeight))
          
          // 计算位置
          const renderX = Math.round((sticker.x / canvasWidth) * exportCanvas.value.width)
          const renderY = Math.round((sticker.y / canvasHeight) * exportCanvas.value.height)

          console.log('贴片最终渲染:', {
            贴片ID: sticker.id,
            缩放信息: {
              保存的比例: sticker.scale,
              计算的比例: sticker.width / sticker.originalWidth,
              使用的比例: scale
            },
            尺寸信息: {
              原始: {
                width: sticker.originalWidth,
                height: sticker.originalHeight
              },
              当前: {
                width: sticker.width,
                height: sticker.height
              },
              导出: {
                width: exportWidth,
                height: exportHeight
              }
            },
            位置: {
              当前: { x: sticker.x, y: sticker.y },
              渲染: { x: renderX, y: renderY }
            }
          })

          ctx.drawImage(
            img,
            renderX,
            renderY,
            exportWidth,
            exportHeight
          )
          break;
      }
    }

    // 继续渲染循环
    if (!videoRef.value.ended) {
      requestAnimationFrame(renderFrame)
    } else {
      console.log('视频播放结束，停止渲染')
      await stopExport()
    }
    
  } catch (error) {
    console.error('渲染帧失败:', error)
    await stopExport()
  }
}

// 修改初始化导出画布的方法
const initExportCanvas = () => {
  exportCanvas.value = document.createElement('canvas')
  exportCanvas.value.width = 1080  // 设置固定的导出尺寸
  exportCanvas.value.height = 1920
  
  // 获取 2D 上下文并设置合适的选项
  exportContext.value = exportCanvas.value.getContext('2d', {
    alpha: false,  // 禁用 alpha 道以提高性能
    willReadFrequently: false  // 不需要频繁读取像素数据
  })
  
  // 设置图像平滑
  exportContext.value.imageSmoothingEnabled = true
  exportContext.value.imageSmoothingQuality = 'high'
  
  // 设置背景为白色
  exportContext.value.fillStyle = '#FFFFFF'
  exportContext.value.fillRect(0, 0, exportCanvas.value.width, exportCanvas.value.height)
}

// 优化的停止导出方法
const stopExport = async () => {
  console.log('停止导出')
  if (!isExporting.value) return
  
  try {
    await videoRef.value?.pause()
    if (mediaRecorder.value?.state !== 'inactive') {
      mediaRecorder.value.stop()
    }
    isExporting.value = false
    showExportProgress.value = false
  } catch (error) {
    console.error('停止导出时出错:', error)
    message.error('停止导出时出错')
  }
}

// 添加获取支持的媒体格式函数
const getSupportedMimeType = () => {
  const types = [
    'video/webm;codecs=h264',
    'video/webm;codecs=vp9',
    'video/webm;codecs=vp8',
    'video/webm',
  ];
  
  for (const type of types) {
    if (MediaRecorder.isTypeSupported(type)) {
      console.log('使用编码格式:', type);
      return type;
    }
  }
  
  console.warn('未找到支持的编码格式，使用默认格式');
  return 'video/webm';
}

// 添加这新的方法
const handleStickerActivated = (stickerId) => {
  console.log('激活贴片:', stickerId)
  activeStickerId.value = stickerId
}

const handleStickerDeactivated = () => {
  console.log('取消激活贴片')
  activeStickerId.value = null
}

// 生命周期钩子
onMounted(() => {
  initExportCanvas()
  initLocalResources()
})

onUnmounted(() => {
  if (mediaRecorder.value) {
    try {
      if (mediaRecorder.value.state !== 'inactive') {
        mediaRecorder.value.stop()
      }
    } catch (error) {
      console.warn('停止录制时出错:', error)
    }
  }
  
  // 清理视频资源
  if (videoRef.value) {
    videoRef.value.pause()
    videoRef.value.src = ''
    videoRef.value.load()
  }
  
  // 清理其他资源
  stickerImagesCache.clear()
  loadedStickers.value.clear()
  
  // 重状态
  isExporting.value = false
  showExportProgress.value = false
  exportProgress.value = 0
})

// 修改拖拽处理函数，使用更可靠的位置计算
const onDrag = (index, { x, y }) => {
  if (index >= 0 && index < stickers.length) {
    const sticker = stickers[index]
    
    // 确保所有值都是有效的数字
    const newX = parseFloat(x)
    const newY = parseFloat(y)
    
    if (!isNaN(newX) && !isNaN(newY)) {
      // 更新贴片位置和transform属性
      sticker.x = newX
      sticker.y = newY
      sticker.transform = {
        ...sticker.transform,
        x: newX,
        y: newY
      }
      
      console.log('贴片位置已更新:', {
        id: sticker.id,
        newPosition: { x: newX, y: newY },
        transform: sticker.transform
      })
    }
  }
}

// 修改缩放处理函数，确保位置和尺寸同步更新
const onResize = (index, { x, y, width, height }) => {
  if (index >= 0 && index < stickers.length) {
    const sticker = stickers[index]
    
    // 只在有宽高数据时更新
    if (width !== undefined && height !== undefined) {
      sticker.width = Math.max(20, width)
      sticker.height = Math.max(20, height)
    }
    
    // 位置总是更新
    if (x !== undefined && y !== undefined) {
      sticker.x = x
      sticker.y = y
    }
    
    console.log('贴片更新:', {
      事件: width !== undefined ? 'resize' : 'drag',
      更新后数据: {
        x: sticker.x,
        y: sticker.y,
        width: sticker.width,
        height: sticker.height
      }
    })
  }
}

// 1. 添加新的响应式变量
const videoList = ref([])      // 视频列表
const backgroundList = ref([]) // 背景图列表
const stickerList = ref([])    // 贴片列表

// 2. 添加初始化函数来加载本素材
const initLocalResources = async () => {
  try {
    // 加载视频列表
    const videos = [
      { id: 'video1', name: '赫莲娜口播', path: '/assets/videos/helena.mp4' },
      { id: 'video2', name: '海蓝之谜口播', path: '/assets/videos/lamer.mp4' },
      // ... 更多视频
    ]
    
    // 为每个视频获取缩略图
    const videosWithThumbnails = await Promise.all(
      videos.map(async (video) => {
        const thumbnail = await getVideoThumbnail(video.path)
        return { ...video, thumbnail }
      })
    )
    
    videoList.value = videosWithThumbnails
    console.log('视频资源加载完成，包含缩略图')
    
    // 加载背景图列表
    backgroundList.value = [
      { id: 'bg1', name: '赫莲娜背景', path: '/assets/backgrounds/helena-bg.png' },
      { id: 'bg2', name: '背景2', path: '/assets/backgrounds/lamer-bg.png' },
      // ... 更多背景
    ]
    
    // 加载贴片列表
    stickerList.value = [
      { id: 'sticker1', name: '赫莲娜机制片1', path: '/assets/stickers/helena-sticker.png' },
      { id: 'sticker2', name: '贴片2', path: '/assets/stickers/lamer-sticker.png' },
      // ... 更多贴片
    ]
    
    console.log('本地素材加载完成')
  } catch (error) {
    console.error('加载本地素材失败:', error)
    message.error('加载素材失败')
  }
}

// 3. 添加素材选择相关方法
const addSticker = (sticker) => {
  const stickerId = `sticker-${Date.now()}`
  const img = new Image()
  img.src = sticker.path
  
  img.onload = () => {
    const initialWidth = img.naturalWidth
    const initialHeight = img.naturalHeight
    
    const newSticker = {
      id: stickerId,
      originalId: sticker.id,
      url: sticker.path,
      x: 20,
      y: 20,
      width: initialWidth,
      height: initialHeight,
      // 保存原始尺寸
      originalWidth: initialWidth,
      originalHeight: initialHeight
    }
    
    stickers.push(newSticker)
    
    // 缓存图片
    stickerImagesCache.set(stickerId, img)
    loadedStickers.value.add(stickerId)
    
    // 始终将新贴片添加到图层最顶部（数组开头）
    layers.unshift({
      type: 'sticker',
      id: stickerId,
      url: sticker.path,
      visible: true
    })
    
    ensureBackgroundAtBottom() // 确保背景在底部
    console.log('贴片添加成功:', newSticker)
    message.success(`已添加贴片：${sticker.name}`)
  }
  
  img.src = sticker.path
}

// 修改 isStrickerUsed 方法
const isStrickerUsed = (stickerId) => {
  const isUsed = stickers.some(s => s.originalId === stickerId)
  return isUsed // 返回布尔值，用于控制样式
}

// 添加获取图层名称的方法
const getLayerName = (layer) => {
  switch (layer.type) {
    case 'video':
      return videoList.value.find(v => v.path === videoUrl.value)?.name || '视频';
    case 'background':
      return backgroundList.value.find(b => b.path === backgroundUrl.value)?.name || '背景';
    case 'sticker':
      return stickerList.value.find(s => s.path === layer.url)?.name || '贴片';
    default:
      return layer.type;
  }
}

// 修改获取视频缩略图的方法
const getVideoThumbnail = async (videoPath) => {
  return new Promise((resolve) => {
    const video = document.createElement('video')
    video.crossOrigin = 'anonymous'
    video.src = videoPath
    
    // 监听可以寻找时间点时
    video.addEventListener('loadedmetadata', () => {
      // 设置到 0.5 秒或者视频总时长的 10%，取较小值
      video.currentTime = Math.min(0.5, video.duration * 0.1)
    })
    
    // 当时间更新后获取帧
    video.addEventListener('timeupdate', () => {
      // 创建临时canvas
      const canvas = document.createElement('canvas')
      canvas.width = video.videoWidth
      canvas.height = video.videoHeight
      
      // 绘制视频帧
      const ctx = canvas.getContext('2d')
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
      
      // 转换为base64
      const thumbnail = canvas.toDataURL('image/jpeg')
      resolve(thumbnail)
      
      // 清理资源
      video.remove()
      canvas.remove()
    }, { once: true }) // 只执行一次
    
    // 添加错误处理
    video.addEventListener('error', () => {
      console.error('获取视频缩略图失败:', videoPath)
      resolve(null)
    })
  })
}

// 修改添加视频的方法,让视频可以插入到任意位置
const selectVideo = (video) => {
  if (videoUrl.value === video.path) {
    // 如果是当前选中的视频，则取消选择
    videoUrl.value = ''
    // 从图层中移除
    const videoLayerIndex = layers.findIndex(layer => layer.type === 'video')
    if (videoLayerIndex !== -1) {
      layers.splice(videoLayerIndex, 1)
    }
    message.success(`已移除视频：${video.name}`)
  } else {
    // 选择新视频
    videoUrl.value = video.path
    // 检查是否已存在视频图层
    const existingVideoLayer = layers.find(layer => layer.type === 'video')
    if (!existingVideoLayer) {
      // 添加到图层管理顶部(而不是固定位置)
      layers.unshift({
        type: 'video',
        id: 'video',
        visible: true
      })
    }
    message.success(`已选择视频：${video.name}`)
  }
}

// 添加背选择方法
const selectBackground = (bg) => {
  if (backgroundUrl.value === bg.path) {
    backgroundUrl.value = ''
    // 从图层中移除背景
    const bgLayerIndex = layers.findIndex(layer => layer.type === 'background')
    if (bgLayerIndex !== -1) {
      layers.splice(bgLayerIndex, 1)
    }
    message.success(`已移除背景：${bg.name}`)
  } else {
    backgroundUrl.value = bg.path
    const img = new Image()
    img.src = bg.path
    img.onload = () => {
      stickerImagesCache.set('background', img)
      
      // 检查是否已存在背景图层
      const existingBgLayer = layers.find(layer => layer.type === 'background')
      if (!existingBgLayer) {
        // 将背景图层添加到数组末尾（最底部）
        layers.push({
          type: 'background',
          id: 'background',
          url: bg.path,
          visible: true,
          locked: true // 添加锁定标志
        })
      }
    }
    message.success(`已选择背景：${bg.name}`)
  }
}

// 添加新的方法来强制背景图层保持在底部
const ensureBackgroundAtBottom = () => {
  const bgLayerIndex = layers.findIndex(layer => layer.type === 'background')
  if (bgLayerIndex !== -1 && bgLayerIndex !== layers.length - 1) {
    const bgLayer = layers.splice(bgLayerIndex, 1)[0]
    layers.push(bgLayer)
  }
}

const handleDragResize = (index, pos, type) => {
  if (index >= 0 && index < stickers.length && type === 'drag') {
    const sticker = stickers[index]
    if (pos.x !== undefined && pos.y !== undefined) {
      sticker.x = pos.x
      sticker.y = pos.y
    }
  }
}

const handleResize = (index, rect) => {
  console.log('resize 事件原始数据:', { index, rect })

  if (index >= 0 && index < stickers.length) {
    const sticker = stickers[index]
    
    console.log('resize 前的贴片数据:', {
      贴片ID: sticker.id,
      当前尺寸: {
        width: sticker.width,
        height: sticker.height
      },
      原始尺寸: {
        width: sticker.originalWidth,
        height: sticker.originalHeight
      }
    })
    
    // 确保 rect 中有有效数据
    const newWidth = rect?.width || rect?.w
    const newHeight = rect?.height || rect?.h
    
    if (newWidth && newHeight) {
      // 计算新的缩放比例
      const scale = Math.max(
        newWidth / sticker.originalWidth,
        newHeight / sticker.originalHeight
      )
      
      // 更新贴片数据
      sticker.width = newWidth
      sticker.height = newHeight
      sticker.scale = scale
      
      console.log('resize 后的贴片数据:', {
        贴片ID: sticker.id,
        新尺寸: {
          width: sticker.width,
          height: sticker.height
        },
        新缩放比例: scale
      })
    }
  }
}

const handleStickerImageLoad = (event, sticker) => {
  const img = event.target
  const naturalWidth = img.naturalWidth
  const naturalHeight = img.naturalHeight
  
  // 计算宽高比
  const aspectRatio = naturalWidth / naturalHeight
  
  // 更新贴片尺寸以匹配图片比例
  if (sticker.width / sticker.height !== aspectRatio) {
    // 保持当前宽度，调整高度
    sticker.height = Math.round(sticker.width / aspectRatio)
    
    console.log('调整贴片尺寸:', {
      贴片ID: sticker.id,
      原始尺寸: {
        width: naturalWidth,
        height: naturalHeight,
        ratio: aspectRatio
      },
      调整后: {
        width: sticker.width,
        height: sticker.height,
        ratio: sticker.width / sticker.height
      }
    })
  }
}
</script>

<style scoped>
.app-container {
  display: grid;
  grid-template-columns: 250px 1fr 300px;
  gap: 20px;
  padding: 20px;
  height: 100vh;
}

.canvas-container {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #f0f0f0;
  border-radius: 8px;
  overflow: hidden;
}

.canvas {
  width: 540px;
  height: 960px;
  background: #fff;
  position: relative;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.control-panel {
  width: 300px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.layer-list {
  max-height: 400px;
  overflow-y: auto;
}

.layer-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.layer-preview {
  width: 24px;
  height: 24px;
  border-radius: 2px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f0f0;
}

.layer-preview img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.sticker-container {
  position: absolute;
  padding: 0 !important;
}

.sticker-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.sticker-image {
  width: 100%;
  height: 100%;
  object-fit: contain;
  pointer-events: none;
  user-select: none;
}

/* 自定义handle样式 */
:deep(.vdr-handle) {
  border: 2px solid #fff;
  background-color: #2196f3;
  width: 10px;
  height: 10px;
}

:deep(.vdr-container.active) {
  border: 2px solid #2196f3;
  z-index: 100;
}

:deep(.vdr-container) {
  border: 1px solid transparent;
}

.video-element {
  width: 100%;
  height: 100%;
  object-fit: contain;
  position: absolute;
}

/* 修复上传按钮样式 */
:deep(.ant-upload) {
  display: block;
  width: 100%;
}

:deep(.ant-upload-select) {
  display: block;
  width: 100%;
}

:deep(.ant-btn) {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 100%;
}

:deep(.ant-btn .anticon) {
  margin-right: 8px;
}

.export-progress {
  padding: 20px;
}

.export-info {
  margin-top: 16px;
  text-align: center;
}

.export-info p {
  margin: 8px 0;
}

/* 添加背景图样式 */
.background-element {
  position: absolute;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

/* 添加新的样式 */
.resource-section {
  margin-bottom: 16px;
}

.section-title {
  font-weight: bold;
  margin-bottom: 8px;
}

.resource-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}

.resource-item {
  width: 100px;
  height: 100px;
  border: 2px solid transparent;
  border-radius: 4px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.resource-item.active {
  border-color: #2196f3;
}

.resource-name {
  text-align: center;
  margin-top: 8px;
}

/* 添加新样式 */
.resource-panel {
  width: 250px;
  height: 100%;
  overflow-y: auto;
  padding-right: 16px;
}

.resource-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
}

.resource-item {
  aspect-ratio: 1;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  padding: 4px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
}

.resource-item:hover {
  border-color: #1890ff;
}

.resource-item.active {
  border-color: #1890ff;
  background: rgba(24, 144, 255, 0.1);
}

.resource-item.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.resource-preview {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.resource-name {
  font-size: 12px;
  margin-top: 4px;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 在控制面板中添加以下内容 */
.control-item {
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.control-label {
  margin-left: 8px;
  color: rgba(0, 0, 0, 0.85);
}

.export-controls {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* 确保最后一个控制项没有底部边距 */
.control-item:last-child {
  margin-bottom: 0;
}

.resource-loading-list {
  list-style: none;
  padding: 0;
  margin: 8px 0;
}

.resource-loading-list li {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.export-info {
  margin-top: 16px;
  text-align: left;
}

.export-info p {
  margin: 8px 0;
}

.video-preview {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f0f0;
  border-radius: 4px;
  overflow: hidden;
}

.video-preview img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.video-preview .anticon {
  font-size: 24px;
  color: #999;
}
</style> 