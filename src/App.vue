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
            :class="{ 
              disabled: isStrickerUsed(sticker.id),
              'video-sticker': isVideoSticker(sticker)
            }"
            @click="handleStickerClick(sticker)"
          >
            <img 
              :src="sticker.path.match(/\.(mp4|webm|mov)$/i) ? stickerImagesCache.get(sticker.path) : sticker.path" 
              class="resource-preview" 
            />
            <div class="resource-name">
              {{ sticker.name }}
              <span v-if="isVideoSticker(sticker)" class="video-badge">视频</span>
            </div>
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
      <!-- 图层理面板 -->
      <a-card title="图层管理" :bordered="false" size="small">
        <a-list
          :data-source="layers"
          class="layer-list"
        >
          <template #renderItem="{ item, index }">
            <a-list-item>
              <div class="layer-item">
                <div class="layer-content">
                  <div class="layer-preview">
                    <template v-if="item.type === 'sticker'">
                      <img 
                        v-if="item.mediaType === 'video'"
                        :src="stickerImagesCache.get(item.url)" 
                        class="preview-image"
                      />
                      <img 
                        v-else 
                        :src="item.url" 
                        class="preview-image"
                      />
                    </template>
                    <template v-else-if="item.type === 'background'">
                      <img :src="item.url" class="preview-image" />
                    </template>
                    <template v-else-if="item.type === 'video'">
                      <video-camera-outlined />
                    </template>
                  </div>
                  <div class="layer-info">
                    <div class="layer-main">
                      <div class="layer-name">{{ getLayerName(item) }}</div>
                      <div class="layer-actions">
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
                        <a-button
                          type="text"
                          danger
                          :disabled="item.type === 'background' || item.type === 'video'"
                          @click="removeLayer(index)"
                        >
                          <delete-outlined />
                        </a-button>
                      </div>
                    </div>
                    <!-- 视频贴片时间信息放在第二行 -->
                    <div v-if="item.type === 'sticker' && item.mediaType === 'video'" class="layer-time">
                      播放时间: {{ formatTime(item.startAt) }} - {{ formatTime(item.startAt + item.duration) }}
                    </div>
                  </div>
                </div>
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
      :keyboard="false"
      :footer="null"
      centered
    >
      <div class="export-progress">
        <a-progress :percent="exportProgress" status="active" />
        <div class="export-info">
          <p>正在导出视频，请稍候...</p>
          <a-button 
            type="primary" 
            danger
            :loading="isStoppingExport"
            @click="handleStopExport"
          >
            停止导出
          </a-button>
        </div>
      </div>
    </a-modal>

    <VideoStartTimeModal
      v-model:visible="showTimeModal"
      :main-video="{ url: videoUrl, duration: videoRef?.duration }"
      :sticker-video="pendingVideoSticker"
      @confirm="handleTimeSelected"
    />
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, watch, defineComponent } from 'vue'
import Vue3DraggableResizable from 'vue3-draggable-resizable'
import 'vue3-draggable-resizable/dist/Vue3DraggableResizable.css'
import { 
  VideoCameraOutlined,
  UpOutlined,
  DownOutlined,
  CheckCircleOutlined,
  DeleteOutlined 
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
        // 图片贴片，加载图片
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
    console.log('频时间值无效，等待...')
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
    
    // 预加载所资源
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
    
    // 重置所有视频贴片到始位置并确保播放
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
      
      // 设置制完成处理
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
    
    // 获取主视频当前时间
    const mainVideoTime = videoRef.value.currentTime
    
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
          break
          
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
          break
          
        case 'sticker':
          const sticker = stickers.find(s => s.id === layer.id)
          if (!sticker) continue
          
          // 第一个日志：在计算缩放比例之前，记录原始信息
          console.log('贴片渲染信息:', {
            编辑时位置: {
              x: sticker.x,
              y: sticker.y,
              width: sticker.width,
              height: sticker.height
            },
            画布信息: {
              canvasWidth: canvasWidth.value,
              canvasHeight: canvasHeight.value,
              exportWidth: exportCanvas.value.width,
              exportHeight: exportCanvas.value.height
            }
          })
          
          // 计算缩放比例
          const scaleX = exportCanvas.value.width / canvasWidth.value
          const scaleY = exportCanvas.value.height / canvasHeight.value
          
          // 计算渲染位置和尺寸（保持精确值，不要过早取整）
          const renderX = sticker.x * scaleX
          const renderY = sticker.y * scaleY
          const renderWidth = sticker.width * scaleX
          const renderHeight = sticker.height * scaleY
          
          // 第二个日志：在计算完转换后的位置后，记录转换结果
          console.log('贴片导出转换:', {
            id: sticker.id,
            缩放比例: { scaleX, scaleY },
            导出时位置: {
              x: renderX,
              y: renderY,
              width: renderWidth,
              height: renderHeight
            }
          })
          
          if (isVideoSticker(sticker)) {
            // 处理视频贴片
            const videoElement = stickerVideoRefs.value.get(sticker.id)
            
            if (videoElement && videoElement.readyState >= 2) {
              const startTime = sticker.videoProps?.startAt || 0
              const duration = sticker.videoProps?.duration || 0
              
              // 只有在主视频时间在贴片的播放时间范围内才渲染
              if (mainVideoTime >= startTime && mainVideoTime <= (startTime + duration)) {
                // 计算贴片应该显示的时间点
                const clipTime = mainVideoTime - startTime
                
                // 只有当时间差异较大时才更新时间
                const currentClipTime = videoElement.currentTime
                const timeDiff = Math.abs(currentClipTime - clipTime)
                if (timeDiff > 0.1) { // 添加阈值，避免频繁更新
                  if (sticker.videoProps?.loop) {
                    videoElement.currentTime = clipTime % duration
                  } else {
                    videoElement.currentTime = Math.min(clipTime, duration)
                  }
                }
                
                // 计算渲染位置和尺寸
                const scaleX = exportCanvas.value.width / canvasWidth.value
                const scaleY = exportCanvas.value.height / canvasHeight.value
                
                const renderX = Math.round(sticker.x * scaleX)
                const renderY = Math.round(sticker.y * scaleY)
                const renderWidth = Math.round(sticker.width * scaleX)
                const renderHeight = Math.round(sticker.height * scaleY)
                
                // 渲染视频贴片
                ctx.drawImage(
                  videoElement,
                  renderX,
                  renderY,
                  renderWidth,
                  renderHeight
                )
              }
            }
          } else {
            // 处理图片贴片
            const img = stickerImagesCache.get(sticker.id)
            if (!img?.complete) continue
            
            // 使精确值进行染
            ctx.drawImage(
              img,
              renderX,
              renderY,
              renderWidth,
              renderHeight
            )
          }
          break
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
    willReadFrequently: false  // 不需要频繁读像素数据
  })
  
  // 设置图像滑
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
    // 设置不保存视频的标志
    shouldSaveVideo.value = videoRef.value?.ended || false
    
    // 暂停主视频 - 修改这部分代码
    if (videoRef.value) {
      try {
        videoRef.value.pause()
      } catch (err) {
        console.warn('暂停主视频失败:', err)
      }
    }
    
    // 停止所有视频贴片的播放
    for (const videoElement of stickerVideoRefs.value.values()) {
      if (videoElement) {
        try {
          videoElement.pause()
        } catch (err) {
          console.warn('暂停视频贴片失败:', err)
        }
      }
    }
    
    // 停止媒体录制
    if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
      // 移除旧的 onstop 处理器
      mediaRecorder.value.onstop = null
      
      // 添加新的 onstop 处理器
      mediaRecorder.value.onstop = () => {
        if (shouldSaveVideo.value) {
          // 只有在正常完成时才保存视频
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
            message.success('视频导出成功')
          } catch (err) {
            console.error('保存导出文件失败:', err)
            message.error('保存导出文件失败')
          }
        } else {
          console.log('导出已取消，不保存视频')
        }
        
        // 重置状态
        isExporting.value = false
        showExportProgress.value = false
        exportProgress.value = 0
        recordedChunks.value = []
        shouldSaveVideo.value = true // 重置标志
      }
      
      try {
        mediaRecorder.value.stop()
      } catch (err) {
        console.error('停止媒体录制失败:', err)
      }
    }
    
    console.log('导出停止完成')
    
  } catch (error) {
    console.error('停止导出时出错:', error)
    message.error('停止导出时出错')
    // 确保状态被重置
    isExporting.value = false
    showExportProgress.value = false
    exportProgress.value = 0
    recordedChunks.value = []
    shouldSaveVideo.value = true // 重置标志
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
  
  console.warn('未找到支持的编码格式，使用默格式');
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
    
    // 只在有宽数据时更新
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
    
    console.log('本地素材加载完，缓存状态:', {
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
    // 检查是否有主视频
    if (!videoUrl.value || !videoRef.value) {
      message.error('请先选择主视频后再添加视频贴片')
      return
    }
    
    console.log('正在添加视频贴片:', sticker)
    
    const video = document.createElement('video')
    video.muted = true
    video.playsInline = true
    video.loop = true
    
    video.addEventListener('loadedmetadata', () => {
      // 保存待处理的视频贴片信息
      pendingVideoSticker.value = {
        video,
        sticker,
        duration: video.duration,
        width: video.videoWidth,
        height: video.videoHeight
      }
      
      // 显示时间选择弹窗
      showTimeModal.value = true
    })
    
    video.addEventListener('error', (e) => {
      console.error('视频加载失败:', e.target.error)
      message.error(`视频加载失败：${e.target.error.message}`)
    })
    
    video.src = sticker.path
    video.load()
  } else {
    // 原有的图片贴片处理逻辑保持不变
    const img = new Image()
    img.onload = () => {
      const newSticker = {
        id: stickerId,
        originalId: sticker.id,
        url: sticker.path,
        type: 'image',
        x: 20,
        y: 20,
        width: 200,
        height: 200,
        originalWidth: img.naturalWidth,
        originalHeight: img.naturalHeight
      }
      
      stickers.push(newSticker)
      stickerImagesCache.set(stickerId, img)
      loadedStickers.value.add(stickerId)
      
      layers.unshift({
        type: 'sticker',
        id: stickerId,
        url: sticker.path,
        mediaType: 'image',
        visible: true
      })
      
      message.success(`已添加贴片：${sticker.name}`)
    }
    
    img.onerror = () => {
      message.error(`加载贴片失败：${sticker.name}`)
    }
    
    img.src = sticker.path
  }
}

// 修改 isStrickerUsed 方法
const isStrickerUsed = (stickerId) => {
  // 检查是否在当前贴片列表中
  return stickers.some(s => s.originalId === stickerId)
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
  // 检查缓存
  if (stickerImagesCache.get(videoPath)) {
    return stickerImagesCache.get(videoPath)
  }

  return new Promise((resolve, reject) => {
    const video = document.createElement('video')
    video.crossOrigin = 'anonymous'
    video.preload = 'metadata'
    
    video.onerror = (error) => {
      console.error('视频加载失败:', error)
      reject(error)
    }
    
    video.onloadedmetadata = () => {
      video.currentTime = video.duration / 2
    }
    
    video.onseeked = () => {
      try {
        const canvas = document.createElement('canvas')
        const maxSize = 200
        const ratio = Math.min(maxSize / video.videoWidth, maxSize / video.videoHeight)
        canvas.width = video.videoWidth * ratio
        canvas.height = video.videoHeight * ratio
        
        const ctx = canvas.getContext('2d')
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
        
        const thumbnail = canvas.toDataURL('image/jpeg', 0.7)
        
        // 缓存生成的预览图
        stickerImagesCache.set(videoPath, thumbnail)
        
        video.remove()
        canvas.remove()
        
        resolve(thumbnail)
      } catch (error) {
        console.error('生成缩略图失败:', error)
        reject(error)
      }
    }
    
    video.src = videoPath
  }).catch(error => {
    console.warn('获取视频缩略图失败:', error)
    return null
  })
}

// 修添加视频的方法,让视频可以插入到任意位置
const selectVideo = (video) => {
  if (videoUrl.value === video.path) {
    // 如果是当前选中的视频，则取消选择
    videoUrl.value = ''
    // 从图层中移除
    const videoLayerIndex = layers.findIndex(layer => layer.type === 'video')
    if (videoLayerIndex !== -1) {
      layers.splice(videoLayerIndex, 1)
    }
    message.success(`移除视频：${video.name}`)
  } else {
    // 选择新视频
    videoUrl.value = video.path
    // 检查是否已存在频图层
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
  console.log('resize 事件始数据:', { index, rect })

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
  
  // 更新片尺寸以匹配图片比例
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

// 添加判断���数
const isVideoSticker = (sticker) => {
  return sticker?.url?.match(/\.(mp4|webm|mov)$/i)
}

// 1. 首先添加一个新的 ref 来存所有视频贴片的引用
const stickerVideoRefs = ref(new Map())

// 1. 定义视频贴片时间选择弹窗组件
const VideoStartTimeModal = defineComponent({
  props: {
    visible: Boolean,
    mainVideo: Object,
    stickerVideo: Object,
  },
  emits: ['update:visible', 'confirm', 'cancel'],
  setup(props, { emit }) {
    const currentTime = ref(0)
    const videoRef = ref(null)
    
    const formatTime = (seconds) => {
      if (!seconds) return '0:00'
      const mins = Math.floor(seconds / 60)
      const secs = Math.floor(seconds % 60)
      return `${mins}:${secs.toString().padStart(2, '0')}`
    }
    
    const handleConfirm = () => {
      if (!props.mainVideo?.duration || !props.stickerVideo?.duration) {
        message.error('视频信息不完整')
        return
      }

      // 检查剩余时间是否足够
      const remainingTime = props.mainVideo.duration - currentTime.value
      if (remainingTime < props.stickerVideo.duration) {
        const shortfall = props.stickerVideo.duration - remainingTime
        message.error(`当前时间点无法完整放视频贴片：
          - 贴片时长：${formatTime(props.stickerVideo.duration)}
          - 剩余时长：${formatTime(remainingTime)}
          - 缺少时间：${formatTime(shortfall)}
          请选择更早的时间点`)
        return
      }
      
      emit('confirm', currentTime.value)
      emit('update:visible', false)
    }

    return {
      currentTime,
      videoRef,
      formatTime,
      handleConfirm
    }
  },
  template: `
    <a-modal
      :visible="visible"
      title="选择视频贴片起始时间"
      @cancel="$emit('update:visible', false)"
      @ok="handleConfirm"
      :width="600"
      :centered="true"
      :destroyOnClose="true"
    >
      <div class="time-selector">
        <div class="video-container" style="width: 70%;">
          <video
            style="width: 70%;"
            ref="videoRef"
            :src="mainVideo?.url"
            controls
            @timeupdate="(e) => currentTime = e.target.currentTime"
          />
        </div>
        <div class="time-info">
          <div class="time-row">
            <span class="time-label">当前选择时间点：</span>
            <span class="time-value">{{ formatTime(currentTime) }}</span>
          </div>
          <div class="time-row">
            <span class="time-label">视频贴片时长：</span>
            <span class="time-value">{{ formatTime(stickerVideo?.duration) }}</span>
          </div>
          <div class="time-row">
            <span class="time-label">主视频总时长：</span>
            <span class="time-value">{{ formatTime(mainVideo?.duration) }}</span>
          </div>
          <div class="time-row">
            <span class="time-label">剩余可用时长：</span>
            <span class="time-value" :class="{ 'time-warning': (mainVideo?.duration - currentTime) < (stickerVideo?.duration || 0) }">
              {{ formatTime(mainVideo?.duration - currentTime) }}
            </span>
          </div>
        </div>
      </div>
    </a-modal>
  `
})

// 2. 主要逻辑代码
const showTimeModal = ref(false)
const pendingVideoSticker = ref(null)

// 监听主视频变化
watch(() => videoUrl.value, (newUrl) => {
  if (newUrl) {
    // 先获取需要清理的视频贴片
    const videoStickers = stickers.filter(s => s.type === 'video')
    
    // 先停止所有视频播放
    videoStickers.forEach(sticker => {
      const video = stickerVideoRefs.value.get(sticker.id)
      if (video) {
        // 先暂停视频播放
        video.pause()
        // 移除视频引用前先清空src
        video.removeAttribute('src')
        video.load()
      }
    })

    // 然后清理其他资源
    videoStickers.forEach(sticker => {
      const index = stickers.findIndex(s => s.id === sticker.id)
      if (index !== -1) {
        // 移除贴片
        stickers.splice(index, 1)
        // 清理引用
        stickerVideoRefs.value.delete(sticker.id)
        loadedStickers.value.delete(sticker.id)
        // 移除对应图层
        const layerIndex = layers.findIndex(l => l.id === sticker.id)
        if (layerIndex !== -1) {
          layers.splice(layerIndex, 1)
        }
      }
    })

    if (videoStickers.length > 0) {
      message.warning('主视频已更换，所有视频贴片已被清除')
    }
  }
})

// 处理时间选择确认
const handleTimeSelected = (startTime) => {
  if (!pendingVideoSticker.value) return
  
  const { video, sticker, duration, width, height } = pendingVideoSticker.value
  const stickerId = `sticker-${Date.now()}`
  
  // 计算合的初始显示尺寸
  const MAX_INITIAL_SIZE = 200
  const aspectRatio = width / height
  let initialWidth, initialHeight
  
  if (aspectRatio >= 1) {
    initialWidth = MAX_INITIAL_SIZE
    initialHeight = MAX_INITIAL_SIZE / aspectRatio
  } else {
    initialHeight = MAX_INITIAL_SIZE
    initialWidth = MAX_INITIAL_SIZE * aspectRatio
  }
  
  // 确保视频贴片的预览图已经被缓存
  if (!stickerImagesCache.get(sticker.path)) {
    console.warn('视频贴片预览图未找到，尝试重新生成')
    getVideoThumbnail(sticker.path).then(thumbnail => {
      if (thumbnail) {
        stickerImagesCache.set(sticker.path, thumbnail)
        console.log('视频贴片预览图已重新生成并缓存')
      }
    })
  }
  
  const newSticker = {
    id: stickerId,
    originalId: sticker.id,
    url: sticker.path,  // 使用原始路径，这样可以通过它找到缓存的预览图
    type: 'video',
    x: 20,
    y: 20,
    width: Math.round(initialWidth),
    height: Math.round(initialHeight),
    originalWidth: width,
    originalHeight: height,
    videoProps: {
      muted: true,
      loop: false,
      currentTime: 0,
      startAt: startTime,
      duration: duration
    }
  }
  
  stickers.push(newSticker)
  stickerVideoRefs.value.set(stickerId, video)
  loadedStickers.value.add(stickerId)
  
  // 更新图层信息
  layers.unshift({
    type: 'sticker',
    id: stickerId,
    url: sticker.path,  // 保持使用原始路径
    mediaType: 'video',
    visible: true,
    startAt: startTime,
    duration: duration
  })
  
  video.play().catch(err => {
    console.error('视频播放失败:', err)
  })
  
  message.success(`已添加视频贴片：${sticker.name}`)
  pendingVideoSticker.value = null
}

// 添加新的方法和状态
const formatTime = (seconds) => {
  if (!seconds) return '0:00'
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

// 修改处理贴片点击的方法
const handleStickerClick = (sticker) => {
  // 检查贴片是否已被使用
  if (isStrickerUsed(sticker.id)) {
    message.warning('该贴片已被使用')
    return
  }
  
  // 如果是视频贴片，需要先检查是否有主视频
  if (isVideoSticker(sticker)) {
    if (!videoUrl.value || !videoRef.value) {
      message.error('请先选择主视频后再添加视频贴片')
      return
    }
  }
  
  addSticker(sticker)
}

// 添加删除图层的方法
const removeLayer = (index) => {
  const layer = layers[index]
  
  // 如果是贴片,需要清理相关资源
  if (layer.type === 'sticker') {
    // 从stickers数组中移除
    const stickerIndex = stickers.findIndex(s => s.id === layer.id)
    if (stickerIndex !== -1) {
      const sticker = stickers[stickerIndex]
      
      // 如果是视频贴片,清理视频资源
      if (isVideoSticker(sticker)) {
        const video = stickerVideoRefs.value.get(sticker.id)
        if (video) {
          video.pause()
          video.src = ''
          video.load()
          stickerVideoRefs.value.delete(sticker.id)
        }
      }
      
      stickers.splice(stickerIndex, 1)
      loadedStickers.value.delete(sticker.id)
    }
  }
  
  // 从图层列表中移除
  layers.splice(index, 1)
  
  message.success('已移除图层')
  
  // 打印调试信息
  console.log('图层移除后的状态:', {
    剩余图层: layers.length,
    剩余贴片: stickers.length,
    视频引用: Array.from(stickerVideoRefs.value.keys()),
    已加载贴片: Array.from(loadedStickers.value)
  })
}

// 添加调试日志
watch(stickerImagesCache, (newCache) => {
  console.log('预览图缓存更新:', {
    缓存数量: newCache.size,
    缓存键列表: Array.from(newCache.keys())
  })
}, { deep: true })

// 1. 在 script setup 顶部添加画布尺寸变量
const canvasWidth = ref(0)
const canvasHeight = ref(0)

// 在mounted时获取真实尺寸
onMounted(() => {
  // 获取画布的实际显示尺寸
  const updateCanvasSize = () => {
    if (canvasRef.value) {
      canvasWidth.value = canvasRef.value.clientWidth
      canvasHeight.value = canvasRef.value.clientHeight
      
      console.log('更新画布尺寸:', {
        实际尺寸: {
          width: canvasWidth.value,
          height: canvasHeight.value
        },
        导出尺寸: {
          width: exportCanvas.value?.width || 1080,
          height: exportCanvas.value?.height || 1920
        }
      })
    }
  }
  
  // 初始更新
  updateCanvasSize()
  
  // 监听窗口大小变化
  window.addEventListener('resize', updateCanvasSize)
  
  // 组件卸载时移除监听
  onUnmounted(() => {
    window.removeEventListener('resize', updateCanvasSize)
  })
})

// 添加新的响应式变量
const shouldSaveVideo = ref(true)
const isStoppingExport = ref(false)

// 添加停止导出的处理函数
const handleStopExport = async () => {
  if (isStoppingExport.value) return
  
  try {
    isStoppingExport.value = true
    await stopExport()
    message.success('已停止导出')
  } catch (error) {
    console.error('停止导出失败:', error)
    message.error('停止导出失败')
  } finally {
    isStoppingExport.value = false
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
  width: 100%;
  padding: 4px 0;
}

.layer-content {
  display: flex;
  gap: 8px;
  width: 100%;
}

.layer-preview {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: 4px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f0f0;
  border: 1px solid #e8e8e8;
}

.preview-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

/* 视频图标样式 */
.layer-preview .anticon {
  font-size: 18px;
  color: #666;
}

.layer-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.layer-main {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.layer-name {
  font-size: 14px;
  color: #333;
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-right: 8px;
}

.layer-time {
  font-size: 12px;
  color: #666;
  display: flex;
  align-items: center;
  gap: 4px;
}

.layer-actions {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.layer-actions .ant-btn {
  min-width: 24px;
  width: 24px;
  height: 24px;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.layer-actions .ant-btn-text.ant-btn-dangerous {
  opacity: 0.8;
}

.layer-actions .ant-btn-text.ant-btn-dangerous:hover {
  opacity: 1;
  background: rgba(255, 77, 79, 0.1);
}

.layer-actions .ant-btn-text.ant-btn-dangerous[disabled] {
  opacity: 0.3;
  cursor: not-allowed;
}

/* 优化列表项样式 */
:deep(.ant-list-item) {
  padding: 8px !important;
  border-radius: 4px;
}

:deep(.ant-list-item:hover) {
  background: #f5f5f5;
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

/* 确保最后一个控项没有底部边距 */
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

.time-selector {
  padding: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.video-container {
  width: 100%;
  height: 280px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #000;
  border-radius: 4px;
  overflow: hidden;
  text-align: center;
}

.video-container video {
  max-width: 100%;
  max-height: 280px;
  width: auto;
  height: auto;
  object-fit: contain;
  margin: 0 auto;
}

.time-info {
  background: #f5f5f5;
  padding: 15px;
  border-radius: 4px;
  width: 100%;
  max-width: 400px;
  margin: 0 auto;
}

.time-row {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 8px 0;
  font-size: 14px;
  text-align: center;
}

.time-label {
  color: #666;
  flex: 0 0 120px;
  text-align: right;
  padding-right: 10px;
}

.time-value {
  color: #333;
  font-weight: 500;
  flex: 0 0 100px;
  text-align: left;
}

.time-warning {
  color: #ff4d4f;
}

/* 添加新的样式 */
.layer-info {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.layer-name {
  font-size: 14px;
}

.layer-time {
  font-size: 12px;
  color: #666;
}

.video-badge {
  background: #1890ff;
  color: #fff;
  padding: 0 4px;
  border-radius: 2px;
  font-size: 12px;
  margin-left: 4px;
}

.resource-item.video-sticker .resource-preview {
  background: #000;
}

.resource-item.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background: #f5f5f5;
}

.resource-item.disabled:hover {
  border-color: #d9d9d9;
}

/* 添加删除按钮的悬停样式 */
.layer-item .ant-btn-text.ant-btn-dangerous {
  opacity: 0.8;
}

.layer-item .ant-btn-text.ant-btn-dangerous:hover {
  opacity: 1;
  background: rgba(255, 77, 79, 0.1);
}

.layer-item .ant-btn-text.ant-btn-dangerous[disabled] {
  opacity: 0.3;
  cursor: not-allowed;
}
</style> 