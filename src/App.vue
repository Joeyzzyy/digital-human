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
            <img 
              :src="sticker.path.match(/\.(mp4|webm|mov)$/i) ? stickerImagesCache.get(sticker.path) : sticker.path" 
              class="resource-preview" 
            />
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
            visibility: isLayerVisible(sticker.id) ? 'open' : 'hidden'
          }"
        >
          <div class="sticker-wrapper">
            <!-- 根据缓存的元素类型显示不同内容 -->
            <template v-if="isVideoSticker(sticker)">
              <video
                :src="sticker.url"
                class="sticker-video"
                :style="{
                  width: '100%',
                  height: '100%',
                  objectFit: 'contain'
                }"
                muted
                playsinline
                loop
                autoplay
                ref="stickerVideoRef"
              ></video>
            </template>
            <template v-else>
              <img 
                :src="sticker.url" 
                class="sticker-image"
                draggable="false"
              />
            </template>
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
      v-model:open="showExportProgress"
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
const remainingTime = ref('计算...')
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
      // 如果是视频贴片，直接返回缓存的预览图
      if (url.match(/\.(mp4|webm|mov)$/i)) {
        const cachedPreview = stickerImagesCache.get(url)
        if (cachedPreview) {
          loadedStickers.value.add(id)
          resolve(cachedPreview)
          return
        }
      }

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
      img.onerror = () => reject(new Error(`图片加载失败: ${url}`))
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
    for (const sticker of stickers) {
      if (isVideoSticker(sticker)) {
        // 对于视频贴片，确保视频元素已准备就绪
        const videoElement = stickerVideoRefs.value.get(sticker.id)
        if (videoElement) {
          loadPromises.push(
            new Promise((resolve) => {
              if (videoElement.readyState >= 2) { // HAVE_CURRENT_DATA
                resolve()
              } else {
                videoElement.addEventListener('loadeddata', () => resolve(), { once: true })
              }
            })
          )
        }
      } else {
        // 对于图片贴片，加载图片
        loadPromises.push(loadImage(sticker.url, sticker.id))
      }
    }
    
    await Promise.all(loadPromises)
    
    console.log('所有资源加载完成', {
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
    console.log('视频加载完成:', {
      duration: videoRef.value.duration,
      readyState: videoRef.value.readyState,
      currentTime: videoRef.value.currentTime
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

// 修改更新导出进度的函数
const updateExportProgress = () => {
  if (!videoRef.value || !isExporting.value) return
  
  const currentTime = videoRef.value.currentTime
  const videoDuration = videoRef.value.duration
  
  // 添加调试日志
  console.log('进度计算数据:', {
    currentTime,
    videoDuration,
    startTime: startTime.value,
    elapsed: (Date.now() - startTime.value) / 1000,
    isPlaying: !videoRef.value.paused,
    readyState: videoRef.value.readyState
  })
  
  // 确保视频已经开始播放且时间值有效
  if (!isFinite(currentTime) || !isFinite(videoDuration) || videoDuration === 0) {
    console.log('视频时间值无效，等待...')
    remainingTime.value = '计算中...'
    requestAnimationFrame(updateExportProgress)
    return
  }
  
  const progress = (currentTime / videoDuration) * 100
  exportProgress.value = Math.round(progress)
  
  // 计算剩余时间
  const elapsed = (Date.now() - startTime.value) / 1000 // 已经过的时间（秒）
  
  // 确保已经过了足够的时间来计算速率
  if (elapsed >= 0.5 && currentTime > 0) { // 等待至少0.5秒以获得更准确的计算
    const rate = currentTime / elapsed // 处理速率（秒/秒）
    console.log('处理速率计算:', { rate, elapsed, currentTime })
    
    if (rate > 0) {
      const remaining = (videoDuration - currentTime) / rate
      if (isFinite(remaining) && remaining > 0) {
        remainingTime.value = `${Math.round(remaining)}秒`
      }
    }
  }
  
  // 继续更新进度
  if (progress < 100 && isExporting.value) {
    requestAnimationFrame(updateExportProgress)
  }
}

// 修改开始导出函数
const startExport = async () => {
  if (!videoUrl.value || isExporting.value) return
  
  try {
    console.log('开始导出')
    isExporting.value = true
    showExportProgress.value = true
    exportProgress.value = 0
    remainingTime.value = '计算中...'
    
    // 初始化导出画布
    initExportCanvas()
    
    // 预加载所有资源
    await preloadResources()
    
    // 确保视频准备就绪
    if (videoRef.value) {
      videoRef.value.currentTime = 0
      try {
        await new Promise((resolve, reject) => {
          const timeupdate = () => {
            if (videoRef.value.currentTime > 0) {
              videoRef.value.removeEventListener('timeupdate', timeupdate)
              resolve()
            }
          }
          videoRef.value.addEventListener('timeupdate', timeupdate)
          
          videoRef.value.play().catch(reject)
        })
        
        // 只有在视频真正开始播放后才设置开始时间
        startTime.value = Date.now()
        console.log('视频开始播放，设置开始时间:', startTime.value)
        
      } catch (err) {
        console.error('视频播放失败:', err)
        throw new Error('无法播放视频')
      }
    }
    
    // 重置所有视频贴片到起始位置并确保播放
    const videoPlayPromises = []
    for (const videoElement of stickerVideoRefs.value.values()) {
      if (videoElement && typeof videoElement.play === 'function') {
        try {
          videoElement.currentTime = 0
          videoPlayPromises.push(
            videoElement.play()
              .catch(err => console.warn('播放视频贴片失败:', err))
          )
        } catch (err) {
          console.warn('设置视频贴片播放失败:', err)
        }
      }
    }
    
    try {
      await Promise.all(videoPlayPromises)
    } catch (err) {
      console.warn('等待视频播放时出错:', err)
    }
    
    // 创建媒体流
    const stream = exportCanvas.value.captureStream(30)
    
    // 添加音频轨道（如果有）
    if (videoRef.value) {
      try {
        const audioTracks = videoRef.value.captureStream().getAudioTracks()
        if (audioTracks.length > 0) {
          stream.addTrack(audioTracks[0])
        }
      } catch (err) {
        console.warn('添加音频轨道失败:', err)
      }
    }
    
    // 配置媒体录制器
    try {
      mediaRecorder.value = new MediaRecorder(stream, {
        mimeType: getSupportedMimeType(),
        videoBitsPerSecond: 8000000
      })
      
      // 设置数据处理
      mediaRecorder.value.ondataavailable = (event) => {
        if (event.data && event.data.size > 0) {
          recordedChunks.value.push(event.data)
        }
      }
      
      // 设置���制完成处理
      mediaRecorder.value.onstop = () => {
        try {
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
          message.success('视频导出成功')
        } catch (err) {
          console.error('保存导出文件失败:', err)
          message.error('保存导出文件失败')
        }
      }
      
      // 开始录制
      mediaRecorder.value.start(1000)
      
      // 开始渲染循环
      renderFrame()
      updateExportProgress()
      
    } catch (err) {
      console.error('创建媒体录制器失败:', err)
      throw new Error('创建媒体录制器失败')
    }
    
  } catch (error) {
    console.error('导出失败:', error)
    message.error('导出失败：' + error.message)
    await stopExport()
  }
}

// 1. 首先添加画布尺寸的计算函数
const getCanvasDimensions = () => {
  const canvas = canvasRef.value
  return {
    width: canvas?.clientWidth || 0,
    height: canvas?.clientHeight || 0
  }
}

// 2. 修改 renderFrame 函数
const renderFrame = async () => {
  if (!isExporting.value || !videoRef.value) return
  
  try {
    const ctx = exportContext.value
    ctx.clearRect(0, 0, exportCanvas.value.width, exportCanvas.value.height)
    
    // 获取画布尺寸
    const { width: canvasWidth, height: canvasHeight } = getCanvasDimensions()
    
    // 按照图层顺序渲染
    const renderLayers = [...layers].reverse()

    for (const layer of renderLayers) {
      if (!layer.visible) continue

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

          if (isVideoSticker(sticker)) {
            // 处理视频贴片
            const videoElement = stickerVideoRefs.value.get(sticker.id)
            if (videoElement && !videoElement.ended) {
              // 计算导出尺寸和位置
              const renderX = Math.round((sticker.x / canvasWidth) * exportCanvas.value.width)
              const renderY = Math.round((sticker.y / canvasHeight) * exportCanvas.value.height)
              const exportWidth = Math.round(sticker.width * (exportCanvas.value.width / canvasWidth))
              const exportHeight = Math.round(sticker.height * (exportCanvas.value.height / canvasHeight))

              ctx.drawImage(
                videoElement,
                renderX,
                renderY,
                exportWidth,
                exportHeight
              )
            }
          } else {
            // 处理图片贴片
            const img = stickerImagesCache.get(sticker.id)
            if (!img?.complete) continue
            
            // 计算导出尺寸和位置
            const renderX = Math.round((sticker.x / canvasWidth) * exportCanvas.value.width)
            const renderY = Math.round((sticker.y / canvasHeight) * exportCanvas.value.height)
            const exportWidth = Math.round(sticker.width * (exportCanvas.value.width / canvasWidth))
            const exportHeight = Math.round(sticker.height * (exportCanvas.value.height / canvasHeight))

            ctx.drawImage(
              img,
              renderX,
              renderY,
              exportWidth,
              exportHeight
            )
          }
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
    willReadFrequently: false  // 不需要频繁读���像素数据
  })
  
  // 设置图像平滑
  exportContext.value.imageSmoothingEnabled = true
  exportContext.value.imageSmoothingQuality = 'high'
  
  // 设置背景为白色
  exportContext.value.fillStyle = '#FFFFFF'
  exportContext.value.fillRect(0, 0, exportCanvas.value.width, exportCanvas.value.height)
}

// 3. 修改 stopExport 函数中的错误处理
const stopExport = async () => {
  console.log('停止导出')
  if (!isExporting.value) return
  
  try {
    // 暂停主视频
    if (videoRef.value) {
      try {
        await videoRef.value.pause()
      } catch (err) {
        console.warn('暂停主视频失败:', err)
      }
    }
    
    // 停止所有视频贴片的播放
    const pausePromises = []
    for (const videoElement of stickerVideoRefs.value.values()) {
      if (videoElement && !videoElement.paused && typeof videoElement.pause === 'function') {
        try {
          pausePromises.push(
            new Promise((resolve) => {
              videoElement.pause()
                .then(() => resolve())
                .catch((err) => {
                  console.warn('暂停视频贴片失败:', err)
                  resolve() // 即使失败也resolve
                })
            })
          )
        } catch (err) {
          console.warn('创建暂停Promise失败:', err)
        }
      }
    }
    
    // 等待所有暂停操作完
    if (pausePromises.length > 0) {
      try {
        await Promise.all(pausePromises)
      } catch (err) {
        console.warn('等待视频暂停时出错:', err)
      }
    }
    
    // 停止媒体录制
    if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
      try {
        mediaRecorder.value.stop()
      } catch (err) {
        console.error('停止媒体录制失败:', err)
      }
    }
    
    // 重置状态
    isExporting.value = false
    showExportProgress.value = false
    exportProgress.value = 0
    
    console.log('导出停止完成')
    
  } catch (error) {
    console.error('停止导出时出错:', error)
    message.error('停止导出时出错')
    // 确保状态被重置
    isExporting.value = false
    showExportProgress.value = false
    exportProgress.value = 0
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

// 1. 添加新的应式变量
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
    ]
    
    // 为每个视频获取缩略图
    const videosWithThumbnails = await Promise.all(
      videos.map(async (video) => {
        const thumbnail = await getVideoThumbnail(video.path)
        return { ...video, thumbnail }
      })
    )
    
    videoList.value = videosWithThumbnails
    
    // 加载背景图列表
    backgroundList.value = [
      { id: 'bg1', name: '赫莲娜背景', path: '/assets/backgrounds/helena-bg.png' },
      { id: 'bg2', name: '背景2', path: '/assets/backgrounds/lamer-bg.png' },
    ]
    
    // 加载贴片列表并预加载所有视频贴片的预览图
    const stickers = [
      { id: 'sticker1', name: '赫莲娜机制片1', path: '/assets/stickers/helena-sticker.png' },
      { id: 'sticker2', name: '海蓝之谜机制贴片1', path: '/assets/stickers/lamer-sticker.png' },
      { id: 'sticker3', name: '测试视频贴片', path: '/assets/stickers/video-sticker.mp4' },
    ]

    // 预加载所有贴片的预览图
    const stickersWithPreviews = await Promise.all(
      stickers.map(async (sticker) => {
        if (sticker.path.match(/\.(mp4|webm|mov)$/i)) {
          try {
            const thumbnail = await getVideoThumbnail(sticker.path)
            if (thumbnail) {
              stickerImagesCache.set(sticker.path, thumbnail)
              console.log(`视频贴片 ${sticker.name} 的预览图已预加载`)
            }
          } catch (error) {
            console.error(`预加载视频贴片 ${sticker.name} 的预览图失败:`, error)
          }
        }
        return sticker
      })
    )

    stickerList.value = stickersWithPreviews
    
    console.log('本地素材加载完成，缓存状态:', {
      视频缩略图: videoList.value.filter(v => v.thumbnail).length,
      贴片预览图: Array.from(stickerImagesCache.keys()),
      总缓存数量: stickerImagesCache.size
    })
  } catch (error) {
    console.error('加载本地素材失败:', error)
    message.error('加载素材失败')
  }
}

// 3. 添加素材选择相关方法
const addSticker = (sticker) => {
  const stickerId = `sticker-${Date.now()}`
  
  if (sticker.path.match(/\.(mp4|webm|mov)$/i)) {
    console.log('正在添加视频贴片:', sticker)
    
    const video = document.createElement('video')
    video.muted = true
    video.playsInline = true
    video.loop = true
    
    video.addEventListener('loadedmetadata', () => {
      // 计算合适的初始显示尺寸
      const MAX_INITIAL_SIZE = 200 // 与图片贴片保持一致
      const aspectRatio = video.videoWidth / video.videoHeight
      let initialWidth, initialHeight
      
      if (aspectRatio >= 1) {
        // 横向视频
        initialWidth = MAX_INITIAL_SIZE
        initialHeight = MAX_INITIAL_SIZE / aspectRatio
      } else {
        // 纵向视频
        initialHeight = MAX_INITIAL_SIZE
        initialWidth = MAX_INITIAL_SIZE * aspectRatio
      }
      
      const newSticker = {
        id: stickerId,
        originalId: sticker.id,
        url: sticker.path,
        type: 'video',
        x: 20,
        y: 20,
        width: Math.round(initialWidth),     // 使用计算后的初始宽度
        height: Math.round(initialHeight),    // 使用计算后的初始高度
        originalWidth: video.videoWidth,      // 保存原始尺寸以供参考
        originalHeight: video.videoHeight,
        videoProps: {
          muted: true,
          loop: true,
          currentTime: 0
        }
      }
      
      console.log('视频贴片初始化尺寸:', {
        原始尺寸: `${video.videoWidth}x${video.videoHeight}`,
        显示尺寸: `${newSticker.width}x${newSticker.height}`,
        纵横比: aspectRatio.toFixed(2)
      })
      
      stickers.push(newSticker)
      stickerVideoRefs.value.set(stickerId, video)
      loadedStickers.value.add(stickerId)
      
      layers.unshift({
        type: 'sticker',
        id: stickerId,
        url: sticker.path,
        mediaType: 'video',
        visible: true
      })
      
      video.play().catch(err => {
        console.error('视频播放失败:', err)
      })
      
      message.success(`已添加视频贴片：${sticker.name}`)
    })
    
    video.addEventListener('error', (e) => {
      console.error('视频加载失败:', e.target.error)
      message.error(`视频加载失败：${e.target.error.message}`)
    })
    
    video.src = sticker.path
    video.load()
  } else {
    // 原有的图片贴片处理逻辑
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
        originalHeight: initialHeight,
      }
      
      stickers.push(newSticker)
      
      // 缓存图片
      stickerImagesCache.set(stickerId, img)
      loadedStickers.value.add(stickerId)
      
      // 始终将新贴片添图层最顶部（数组开头）
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
  return new Promise((resolve, reject) => {
    const video = document.createElement('video')
    video.crossOrigin = 'anonymous'
    video.preload = 'metadata' // 添加预加载设置
    
    // 添加错误处理
    video.onerror = (error) => {
      console.error('视频加载失败:', error)
      reject(error)
    }
    
    // 监听元数据加载完成
    video.onloadedmetadata = () => {
      // 设置到视频的中间位置以获取更有代表性的帧
      video.currentTime = video.duration / 2
    }
    
    // 监听特定帧加载完成
    video.onseeked = () => {
      try {
        // 创建canvas并设置合适的尺寸
        const canvas = document.createElement('canvas')
        const maxSize = 200 // 限制缩略图最大尺寸
        const ratio = Math.min(maxSize / video.videoWidth, maxSize / video.videoHeight)
        canvas.width = video.videoWidth * ratio
        canvas.height = video.videoHeight * ratio
        
        // 绘制视频帧
        const ctx = canvas.getContext('2d')
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
        
        // 转换为base64并返回
        const thumbnail = canvas.toDataURL('image/jpeg', 0.7) // 使用较低质量以减小大小
        
        // 清理资源
        video.remove()
        canvas.remove()
        
        resolve(thumbnail)
      } catch (error) {
        console.error('生成缩略图失败:', error)
        reject(error)
      }
    }
    
    // 设置视频源并开始加载
    video.src = videoPath
  }).catch(error => {
    console.warn('获取视频缩略图失败:', error)
    return null // 失败时返回 null
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
  
  // 计算宽比
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

// 添加判断函数
const isVideoSticker = (sticker) => {
  return sticker?.url?.match(/\.(mp4|webm|mov)$/i)
}

// 1. 首先添加一个新的 ref 来存储所有视频贴片的引用
const stickerVideoRefs = ref(new Map())
</script>

<style scoped>
.app-container {
  display: grid;
  grid-template-columns: 250px 1fr 300px;
  gap: 20px;
  padding: 20px;
  height: 100vh;
  overflow: hidden;
  min-width: 1024px;
}

.canvas-container {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #f0f0f0;
  border-radius: 8px;
  overflow: hidden;
  min-height: 400px;
}

.canvas {
  aspect-ratio: 9/16;
  height: 95%;
  min-height: 400px;
  max-height: 90vh;
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

.time-control-panel {
  padding: 16px;
}

.timeline {
  height: 60px;
  position: relative;
  margin-bottom: 16px;
}

.timeline-ruler {
  height: 20px;
  background: #f0f0f0;
  position: relative;
}

.timeline-track {
  height: 40px;
  background: #fafafa;
  position: relative;
}

.video-block {
  position: absolute;
  height: 30px;
  top: 5px;
  background: #1890ff;
  border-radius: 4px;
  cursor: move;
}

.video-block:hover {
  background: #40a9ff;
}

.time-inputs {
  margin-bottom: 16px;
}

.unit {
  margin-left: 8px;
  color: rgba(0, 0, 0, 0.45);
}

.preview-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style> 