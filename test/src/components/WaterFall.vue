<template>
  <div class="waterfall-container" ref="container">
    <div 
      v-for="(column, colIndex) in columns" 
      :key="colIndex" 
      class="waterfall-column"
    >
      <div 
        v-for="item in column.items" 
        :key="item.id" 
        class="waterfall-item"
        :style="{ width: `${itemWidth}px` }"
        ref="itemsRef"
      >
        <slot :item="item">
          <!-- 默认内容 -->
          <div class="content-wrapper">
            <img 
              v-if="item.type === 'image'"
              :src="item.url" 
              @load="handleItemLoad($event, item)"
            />
            <video 
              v-else-if="item.type === 'video'"
              controls 
              @loadedmetadata="handleItemLoad($event, item)"
            >
              <source :src="item.url" :type="item.videoType || 'video/mp4'">
            </video>
            <div v-if="item.title" class="item-title">{{ item.title }}</div>
            <div v-if="item.desc" class="item-desc">{{ item.desc }}</div>
          </div>
        </slot>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { useElementSize } from '@vueuse/core'

const props = defineProps({
  items: { type: Array, required: true },
  gap: { type: Number, default: 20 },
  itemWidth: { type: Number, default: 300 },  // 单个项目固定宽度
  minColumns: { type: Number, default: 1 },   // 最小列数
  maxColumns: { type: Number, default: Infinity } // 最大列数
})

const container = ref(null)
const itemsRef = ref([])
const { width: containerWidth } = useElementSize(container)
const itemHeights = ref({})
const columns = ref([])
const resizeObserver = ref(null)

// 计算列数
const columnCount = computed(() => {
  if (!containerWidth.value) return props.minColumns
  
  const maxPossibleColumns = Math.floor(
    (containerWidth.value + props.gap) / (props.itemWidth + props.gap)
  )
  
  return Math.min(
    props.maxColumns,
    Math.max(
      props.minColumns,
      maxPossibleColumns
    )
  )
})

// 更新布局函数
function updateLayout() {
  const newColumns = Array.from({ length: columnCount.value }, () => ({
    items: [],
    height: 0
  }))
  
  props.items.forEach(item => {
    const height = itemHeights.value[item.id] || 0
    const shortestColumn = newColumns.reduce((prev, curr) => 
      curr.height < prev.height ? curr : prev, 
      newColumns[0]
    )
    
    shortestColumn.items.push(item)
    shortestColumn.height += height + props.gap
  })
  
  columns.value = newColumns
  console.log(columns)
}

// 防抖版本
let debounceTimer = null
function debouncedUpdateLayout() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    updateLayout()
  }, 100)
}

// 处理内容加载
function handleItemLoad(event, item) {
  nextTick(() => {
    const itemElement = event.target.closest('.waterfall-item')
    if (itemElement) {
      itemHeights.value[item.id] = itemElement.offsetHeight
      debouncedUpdateLayout()
    }
  })
}

// 初始化ResizeObserver
function initResizeObserver() {
  if (resizeObserver.value) {
    resizeObserver.value.disconnect()
  }
  
  resizeObserver.value = new ResizeObserver((entries) => {
    entries.forEach(entry => {
      const itemId = entry.target.__itemId
      if (itemId) {
        itemHeights.value[itemId] = entry.target.offsetHeight
      }
    })
    debouncedUpdateLayout()
  })
  
  itemsRef.value.forEach(el => {
    if (el && el.__itemId) {
      resizeObserver.value.observe(el)
    }
  })
}

// 监视必要的变化
watch(() => props.items, () => {
  nextTick(() => {
    // 更新item refs
    itemsRef.value.forEach((el, index) => {
      if (el && props.items[index]) {
        el.__itemId = props.items[index].id
      }
    })
    initResizeObserver()
    updateLayout() // 立即更新布局
  })
}, { immediate: true })

watch(() => [containerWidth.value, props.gap, props.itemWidth], () => {
  debouncedUpdateLayout()
})

onMounted(() => {
  window.addEventListener('resize', debouncedUpdateLayout)
})

onUnmounted(() => {
  if (resizeObserver.value) {
    resizeObserver.value.disconnect()
  }
  window.removeEventListener('resize', debouncedUpdateLayout)
  clearTimeout(debounceTimer)
})
</script>

<style scoped>
.waterfall-container {
  display: flex;
  gap: v-bind('gap + "px"');
  justify-content: center;
}

.waterfall-column {
  display: flex;
  flex-direction: column;
  gap: v-bind('gap + "px"');
}

.waterfall-item {
  background: #fff;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.waterfall-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
}

.content-wrapper {
  padding: 12px;
}

.waterfall-item img {
  width: 100%;
  height: auto;
  display: block;
  border-radius: 4px;
  margin-bottom: 8px;
}

.waterfall-item video {
  width: 100%;
  display: block;
  border-radius: 4px;
  margin-bottom: 8px;
}

.item-title {
  font-weight: bold;
  margin-bottom: 4px;
  font-size: 16px;
}

.item-desc {
  font-size: 14px;
  color: #666;
  margin-bottom: 8px;
}

/* @media (max-width: 768px) {
  .waterfall-container {
    flex-direction: column;
  }
  
  .waterfall-item {
    width: 100% !important;
  }
} */
</style>
