<template>
  <div 
    class="waterfall-container"
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
        <div class="card-item">
          <!-- 媒体内容 -->
          <div class="media-wrapper" :style="{ paddingTop: `${100 / (item.ratio || 1.7778)}%` }">
            <video
              v-if="item.videoUrl"
              :src="item.videoUrl"
              :poster="item.cover"
              controls
              class="media-content"
            ></video>
            <img
              v-else
              v-lazy="item.cover"
              class="media-content"
              :src="item.url"
              :alt="item.title"
            >
            
          </div>

          <!-- 信息覆盖层 -->
          <div class="content-overlay">
            <div class="top-bar">
              <span class="game-tag">{{ item.tag }}</span>
            </div>
            
            <!-- 作者信息 -->
            <div class="author-info">
              <img :src="item.avatar" class="author-avatar" />
              <div class="meta">
                <h3 class="title">{{ item.title }}</h3>
                <p class="author">@{{ item.author }}</p>
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

const emit = defineEmits(['load-more', 'toggle-like'])

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
const MIN_PULL_DISTANCE = 80 // 触发刷新的最小拖动距离
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
  const scrollTop = document.documentElement.scrollTop
  if (scrollTop === 0) { // 只有当在顶部时才开启下拉检测
    pullStartY.value = e.touches[0].clientY
    isPullActive.value = true
  }
}
const handleTouchMove = (e) => {
  if (!isPullActive.value) return
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
  }
}
const handleTouchEnd = () => {
  if (!isPullActive.value) return
  isPullActive.value = false
  // console.log(pullOffset.value)
  if (pullOffset.value > MIN_PULL_DISTANCE) {
    triggerRefresh()
  } else if(pullOffset.value < -MIN_PULL_DISTANCE) {
    handleReachBottom()
  }
  else {
    resetPull()
  }
}
// 触发刷新动作
const triggerRefresh = async () => {
  // isRefreshing.value = true
  pullOffset.value = MIN_PULL_DISTANCE // 保持刷新状态下的偏移量
  try {
    console.log('refresh')
    await emit('refresh') // 触发父组件刷新
  } finally {
    resetPull()
  }
}
// 重置下拉状态
const resetPull = () => {
  pullOffset.value = 0
  isRefreshing.value = false
  isLoadingMore.value = false
}
// 上拉加载处理（滚动到底部）
const handleReachBottom = useThrottleFn(async () => {
  if (!hasMore.value) return
  // isLoadingMore.value = true
  try {
    console.log('load-more')
    await emit('load-more')
  } finally {
    resetPull()
  }
}, 1000)
</script>

<style lang="scss" scoped>
.waterfall-container {
  width: 100%;
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

  .top-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;

    .game-tag {
      background: rgba(0, 0, 0, 0.75);
      color: #fff;
      padding: 4px 12px;
      border-radius: 20px;
      font-size: 12px;
      line-height: 1.4;
    }
  }
}

.author-info {
  display: flex;
  align-items: center;
  gap: 12px;

  .author-avatar {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .meta {
    flex: 1;
    min-width: 0;

    .title {
      font-size: 15px;
      margin: 0;
    }

    .author {
      font-size: 13px;
      color: #666;
      margin: 2px 0 0;
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
  top: -50px;
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
      transform: none;
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
</style>
