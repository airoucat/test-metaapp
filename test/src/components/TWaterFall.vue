<template>
  <div 
    class="waterfall-container"
    ref="containerRef"
    @touchstart="handleTouchStart"
    @touchmove="handleTouchMove"
    @touchend="handleTouchEnd"
    :style="{ transform: `translateY(${pullOffset}px)` }"
  >
    <div class="refresh-indicator" v-show="isRefreshing || pullProgress > 0">
      <div class="loader" :class="{ active: isRefreshing }">
        {{ refreshStatusText }}
      </div>
    </div>
    <waterfall
      :list="processedList"
      :width="cardWidth"
      :breakpoints="responsiveBreakpoints"
      :gutter="15"
      row-key="id"
      img-selector="cover"
    >
      <template #item="{ item }">
        <div class="card-item" 
        @click="cardClickHandler(item)" >
          <!-- 媒体内容 -->
          <div class="media-wrapper" :style="{ paddingTop: `${(item.ratio || 0.5625) * 100}%` }">
            <!-- <video
              v-if="item.videoUrl"
              :src="item.videoUrl"
              :poster="item.cover"
              controls
              class="media-content"
            ></video> -->
            <img
              v-lazy="item.cover"
              class="media-content"
              :src="item.url"
              :alt="item.title"
            >
            <div v-if="item.videoUrl" class="video-play-hint">
              <svg class="play-icon" viewBox="0 0 24 24">
                <path d="M8 5v14l11-7z"/>
              </svg>
            </div>
          </div>

          <!-- 信息覆盖层 -->
          <div class="content-overlay">
            <h3 class="title">{{ item.title }}</h3>
            <div class="top-bar">
              <span class="game-tag">{{ item.tag }}</span>
            </div>
            
            <!-- 作者信息 -->
            <div class="author-info">
              <img :src="item.avatar" class="author-avatar" />
              <div class="meta">
                <p class="author">{{ item.author }}</p>
              </div>
            </div>
          </div>
        </div>
      </template>
    </waterfall>
    <div class="loadmore-indicator">
      <div v-if="isLoadingMore" class="loading-text">
        <span class="dot-flashing"></span>
        加载更多
      </div>
      <div v-else-if="!hasMore" class="no-more">
        没有更多内容了
      </div>
    </div>
  </div>
  <teleport to="body">
  <transition name="video-fade">
    <div v-if="showVideoModal" class="video-modal" @click.self="closeVideo">
      <div class="video-container">
        <video 
          ref="videoPlayer"
          :src="selectedVideoUrl"
          controls
          autoplay
        ></video>
      </div>
    </div>
  </transition>
</teleport>
</template>

<script setup>
import { ref, computed, watch, nextTick } from 'vue'
import { LazyImg, Waterfall } from 'vue-waterfall-plugin-next'
import 'vue-waterfall-plugin-next/dist/style.css'
import { useScroll, useThrottleFn } from '@vueuse/core'

const props = defineProps({
  items: {
    type: Array,
    required: true,
  },
  isLoading: Boolean,
  hasMore: Boolean
})

const emit = defineEmits(['load-more', 'refresh', 'toggle-like'])

// 响应式断点配置
const responsiveBreakpoints = {
  1280: { rowPerView: 4 },  // 桌面大屏
  1024: { rowPerView: 4 },  // 桌面常规
  768:  { rowPerView: 3 },  // 平板
  480:  { rowPerView: 2 },  // 手机竖屏
  0:    { rowPerView: 1 }   // 极小屏幕
}

// 基准卡片宽度计算
const cardWidth = computed(() => {
  const viewportWidth = window.innerWidth
  console.log(viewportWidth)
  return viewportWidth > 768 ? 300 : 280  // 响应式基础宽度
})

// 数据格式处理
const processedList = computed(() => props.items.map(item => ({
  ...item,
  // 确保字段与组件要求一致
  cover: item.videoUrl ? item.cover : item.url,
  // ratio: item.videoUrl ? 16/9 : 4/3  // 宽高比提示
})))

// 处理滚动触底
const handleScrollBottom = async () => {
  if (props.isLoading || !props.hasMore) return
  emit('load-more')
}

// 点赞处理
const toggleLike = item => {
  emit('toggle-like', {
    id: item.id,
    value: !item.liked
  })
}

// 下拉刷新相关
const containerRef = ref(null)
// 在原有的响应式变量后添加这两个控制变量
const enablePullDown = ref(false)  // 是否允许下拉刷新
const enablePullUp = ref(false)    // 是否允许上拉加载
const MIN_PULL_DISTANCE = 80 // 触发刷新的最小拖动距离
const PULL_DIRECTION = {       // 拉动方向枚举
  UP: 'up',
  DOWN: 'down'
}
const DAMPING_FACTOR = 0.6     // 滚动阻尼系数
const pullStartY = ref(0)
const pullOffset = ref(0)
const isPullActive = ref(false)
const isRefreshing = ref(false)
// 上拉加载相关
const isLoadingMore = ref(false)
const hasMore = ref(true)
const { y: scrollY } = useScroll(window)
// 组合式状态计算
const pullProgress = computed(() => 
  Math.min(pullOffset.value / MIN_PULL_DISTANCE, 1)
)
const refreshStatusText = computed(() => {
  if (isRefreshing.value) return '换一批'
  return pullOffset.value > MIN_PULL_DISTANCE ? '释放刷新' : '下拉刷新'
})
// 触摸事件处理
const handleTouchStart = (e) => {
  /* console.log('touchstart')
  const scrollTop = document.documentElement.scrollTop
  if (scrollTop === 0) { // 只有当在顶部时才开启下拉检测
    pullStartY.value = e.touches[0].clientY
    isPullActive.value = true
  } */
  const container = containerRef.value
  if (!container) return
  // 获取当前滚动位置参数
  const { scrollTop, scrollHeight, clientHeight } = container
  const currentPosition = scrollTop + clientHeight
  console.log(scrollTop, currentPosition, scrollHeight)
  
  // 设置可操作方向（顶部5px内可下拉，底部5px内可上拉）
  enablePullDown.value = scrollTop <= 5
  enablePullUp.value = (scrollHeight - currentPosition) <= 5
  pullStartY.value = e.touches[0].clientY
  isPullActive.value = true
}
const handleTouchMove = (e) => {
  /* if (!isPullActive.value) return
  const currentY = e.touches[0].clientY
  const deltaY = currentY - pullStartY.value
  // 向下拖动且未达到最大距离（带阻尼效果）
  if (deltaY !== 0) {
    if(deltaY>MIN_PULL_DISTANCE) {
      isRefreshing.value = true
    } else if(deltaY<-MIN_PULL_DISTANCE) {
      isLoadingMore.value = true
    }
    const dampedY = deltaY * 0.6 // 阻尼系数
    pullOffset.value = Math.min(dampedY, MIN_PULL_DISTANCE * 1.5)
    e.preventDefault() // 阻止默认滚动行为
  } */
  if (!isPullActive.value) return
  
  const currentY = e.touches[0].clientY
  const deltaY = (currentY - pullStartY.value) * DAMPING_FACTOR
  const direction = deltaY > 0 ? PULL_DIRECTION.DOWN : PULL_DIRECTION.UP
  
  // 动态计算允许的操作
  let allowAction = false
  
  if (direction === PULL_DIRECTION.DOWN && enablePullDown.value) {
    pullOffset.value = Math.min(deltaY, MIN_PULL_DISTANCE * 1.5)
    isRefreshing.value = pullOffset.value > MIN_PULL_DISTANCE
    allowAction = true
  }
  console.log(isRefreshing.value, pullOffset.value);
  
  if (direction === PULL_DIRECTION.UP && enablePullUp.value) {
    pullOffset.value = Math.max(deltaY, -MIN_PULL_DISTANCE * 1.5)
    isLoadingMore.value = Math.abs(pullOffset.value) > MIN_PULL_DISTANCE
    allowAction = true
  }
  // console.log(deltaY, pullOffset.value, allowAction, enablePullDown.value, enablePullUp.value)
  // 只有需要拉动反馈时才阻止默认滚动
  if (allowAction && Math.abs(deltaY) > 5) {
    e.preventDefault()
  }
}
const handleTouchEnd = () => {
  /* if (!isPullActive.value) return
  isPullActive.value = false
  // console.log(pullOffset.value)
  if (pullOffset.value > MIN_PULL_DISTANCE) {
    triggerRefresh()
  } else if(pullOffset.value < -MIN_PULL_DISTANCE) {
    handleReachBottom()
  }
  else {
    resetPull()
  } */
  if (!isPullActive.value) return
  isPullActive.value = false
  // 通过最终偏移量触发操作
  if (pullOffset.value > MIN_PULL_DISTANCE) {
    console.log('refresh')
    triggerRefresh()
  } else if (pullOffset.value < -MIN_PULL_DISTANCE) {
    console.log('load-more')
    handleReachBottom()
  }
  
  // 重置状态
  resetPull()
}
/* // 触发刷新动作
const triggerRefresh = async () => {
  // isRefreshing.value = true
  pullOffset.value = MIN_PULL_DISTANCE // 保持刷新状态下的偏移量
  try {
    console.log('refresh')
    await emit('refresh') // 触发父组件刷新
  } finally {
    resetPull()
  }
} */
// 增强的刷新方法
const triggerRefresh = useThrottleFn(async () => {
  console.log(isRefreshing.value)
  if (!isRefreshing.value) return
  
  isRefreshing.value = true
  try {
    await emit('refresh')
  } finally {
    setTimeout(resetPull, 300)  // 保持提示显示时间
  }
}, 1000)

// 重置下拉状态
const resetPull = () => {
  pullOffset.value = 0
  enablePullDown.value = false
  enablePullUp.value = false
  isRefreshing.value = false
  isLoadingMore.value = false
}
// 上拉加载处理（滚动到底部）
const handleReachBottom = useThrottleFn(async () => {
  // console.log(isLoadingMore.value)
  console.log(props.hasMore, isLoadingMore.value)
  // if (!props.hasMore || isLoadingMore.value) return
  
  isLoadingMore.value = true
  try {
    await emit('load-more')
  } finally {
    setTimeout(() => (isLoadingMore.value = false), 300)
  }
}, 1000)

// 视频播放处理
const showVideoModal = ref(false)
const videoPlayer = ref(null)
const selectedVideoUrl = ref('')
const cardClickHandler = (item) => {
  console.log('click', item)
  if (item.videoUrl) {
    // selectedVideoUrl.value = item.videoUrl
    selectedVideoUrl.value = "http://vjs.zencdn.net/v/oceans.mp4"
    showVideoModal.value = true
  }
}
const closeVideo = () => {
  showVideoModal.value = false
  const video = videoPlayer.value
  if (video) {
    video.pause()
    video.currentTime = 0
  }
}
</script>

<style lang="scss" scoped>
.waterfall-container {
  width: 100%;
  overflow: scroll;
  height: calc(100vh - 2rem);
  // padding: 0 12px;
  max-width: 1440px;
  margin: 0 auto;
}

.card-item {
  background: #fff;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.08);

}

.media-wrapper {
  position: relative;
  padding-top: 56.25%;  // 按比例撑开容器

  .media-content {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #f5f5f5;
    border-radius: 8px 8px 0 0;
  }
  .media-content-img {
    object-fit: cover;
    width:100%;
    // height: 100%;
  }
}

.content-overlay {
  padding: 12px 16px;

  .title {
      font-size: 14px;
      margin: 0;
      color: #000;
      text-align: left;
    }

  .top-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 6px;

    .game-tag {
      background: rgba(198, 198, 198, 0.75);
      color:rgba(59, 58, 58, 0.75);
      padding: 4px;
      border-radius: 5px;
      font-size: 10px;
      line-height: 1.4;
    }
  }
}

.author-info {
  display: flex;
  align-items: center;
  gap: 6px;

  .author-avatar {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .meta {
    flex: 1;
    min-width: 0;

    .author {
      font-size: 13px;
      color: #666;
      margin: 2px 0 0;
      text-align: left;
    }
  }
}

.status-loading,
.status-no-more {
  text-align: center;
  padding: 24px;
  color: #666;
  font-size: 14px;
}
.refresh-indicator {
  position: absolute;
  z-index: 1000;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
  text-align: center;
  .loader {
    display: inline-flex;
    align-items: center;
    padding: 8px 20px;
    background: rgba(0, 0, 0, 0.8);
    color: white;
    border-radius: 25px;
    transition: all 0.3s ease;
    transform: scale(0.8);
    opacity: 0;
    &.active {
      // transform: none;
      opacity: 1;
    }
  }
}
.loadmore-indicator {
  padding: 20px 0;
  text-align: center;
  color: #000;
  .dot-flashing {
    display: inline-block;
    width: 8px;
    height: 8px;
    margin-right: 8px;
    border-radius: 50%;
    background-color: #333;
    animation: dotFlashing 1s infinite linear;
  }
}
@keyframes dotFlashing {
  0%, 80%, 100% { opacity: 0; }
  40% { opacity: 1; }
}
/* 视频提示样式 */
.video-play-hint {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 48px;
    height: 48px;
    background: rgba(0,0,0,0.6);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: opacity 0.3s;

    .play-icon {
      fill: white;
      width: 24px;
      height: 24px;
      margin-left: 3px;
    }
  }

  .video-play-hint:hover {
    opacity: 0.9;
  }

/* 视频模态框样式 */
.video-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.8);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;

  .video-container {
    width: 80vw;
    max-width: 900px;
    video {
      width: 100%;
      border-radius: 8px;
      overflow: hidden;
    }
  }
}

.video-fade-enter-active,
.video-fade-leave-active {
  transition: opacity 0.3s;
}

.video-fade-enter-from,
.video-fade-leave-to {
  opacity: 0;
}

</style>
