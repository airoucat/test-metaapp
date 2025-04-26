<template>
  <div class="waterfall-container">
    <div 
      v-for="column in columns"
      :key="column.id"
      class="waterfall-column"
    >
      <div 
        v-for="item in column.items"
        :key="item.id"
        class="waterfall-item"
        ref="itemRefs"
      >
        <!-- 媒体内容 -->
        <div class="media-container">
          <img
            v-if="item.type === 'image'"
            :src="item.url"
            :alt="item.title"
            @load="handleMediaLoad"
          />
          <video
            v-else
            :poster="item.poster"
            preload="metadata"
            @loadedmetadata="handleMediaLoad"
          >
            <source :src="item.url">
          </video>
        </div>

        <!-- 文本内容 -->
        <div class="content-info">
          <h3 class="title">{{ item.title }}</h3>
          <p class="author">@{{ item.author }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useDebounceFn } from '@vueuse/core'

const props = defineProps<{
  items: WaterfallItem[]
  gap?: number // 间距，默认8px
}>()

// 响应式数据
const itemHeights = ref(new Map<string | number, number>())
const columns = ref<Array<{ id: number; items: WaterfallItem[]; height: number }>>([
  { id: 1, items: [], height: 0 },
  { id: 2, items: [], height: 0 }
])

// 布局参数
const gap = computed(() => props.gap || 8)
const containerWidth = ref(0)

// 防抖更新布局
const updateLayout = useDebounceFn(() => {
  const newColumns = [
    { id: 1, items: [], height: 0 },
    { id: 2, items: [], height: 0 }
  ]

  props.items.forEach(item => {
    // 获取实际高度（优先使用已计算高度）
    const height = itemHeights.value.get(item.id) || estimateHeight(item) || 300
    
    // 选择较矮的列
    const targetColumn = newColumns[0].height <= newColumns[1].height ? 0 : 1
    
    newColumns[targetColumn].items.push(item)
    newColumns[targetColumn].height += height + gap.value
  })

  columns.value = newColumns
}, 100)

// 估算高度（用于未加载时的占位）
const estimateHeight = (item: WaterfallItem) => {
  if (item.height && item.width) {
    const aspectRatio = item.height / item.width
    return (containerWidth.value / 2 - gap.value) * aspectRatio
  }
  return null
}

// 媒体加载处理
const handleMediaLoad = (event: Event) => {
  const element = event.target as HTMLElement
  const itemEl = element.closest('.waterfall-item')
  if (!itemEl) return

  const height = itemEl.getBoundingClientRect().height
  const itemId = itemEl.dataset.itemId
  if (itemId) {
    itemHeights.value.set(itemId, height)
    updateLayout()
  }
}

// 响应式布局
const handleResize = () => {
  containerWidth.value = document.documentElement.clientWidth
  updateLayout()
}

// 生命周期
onMounted(() => {
  handleResize()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped lang="scss">
.waterfall-container {
  display: flex;
  justify-content: space-between;
  padding: 0 v-bind('gap + "px"');
  box-sizing: border-box;

  .waterfall-column {
    width: calc(50% - #{v-bind('gap / 2 + "px"')});
    
    .waterfall-item {
      margin-bottom: v-bind('gap + "px"');
      background: #fff;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
      
      .media-container {
        position: relative;
        width: 100%;
        
        img, video {
          width: 100%;
          height: auto;
          display: block;
          background: #f5f5f5;
        }
        
        video {
          aspect-ratio: 9/16;
          object-fit: cover;
        }
      }

      .content-info {
        padding: 12px;
        
        .title {
          margin: 0 0 6px;
          font-size: 14px;
          line-height: 1.4;
        }
        
        .author {
          margin: 0;
          font-size: 12px;
          color: #666;
        }
      }
    }
  }
}

// 手机横屏适配
@media (orientation: landscape) {
  .waterfall-container {
    padding: 0 16px;
  }
}
</style>