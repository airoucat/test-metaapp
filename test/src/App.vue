<template>
  <div class="header"></div>
  <div class="gaming-content-container">
    <!-- 瀑布流区域 -->
    <WaterfallComponent :items="contentList" :loading="isLoading" :hasMore="hasMoreData" @load-more="fetchContent"
      @like="handleLikeEvent" />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import WaterfallComponent from './components/TWaterfall.vue'

// 初始数据
const initData = [
  {
    id: 1,
    url: 'https://picsum.photos/400/600',
    videoUrl: '',
    title: '艾尔登法环实况',
    author: '游戏玩家小拉',
    avatar: '',
    like: 2431,
    tag: '艾尔登法环',
    liked: false,
    ratio:4/6
  },
  {
    id: 2,
    url: 'https://picsum.photos/400/300',
    videoUrl: '',
    title: '艾尔登法环实况',
    author: '游戏玩家小拉',
    avatar: '',
    like: 2431,
    tag: '艾尔登法环',
    liked: false,
    ratio: 4/3
  },
  {
    id: 3,
    url: 'https://picsum.photos/500/300',
    videoUrl: '',
    title: '艾尔登法环实况',
    author: '游戏玩家小拉',
    avatar: '',
    like: 2431,
    tag: '艾尔登法环',
    liked: false,
    ratio: 5/3
  }
  // 可添加更多初始数据...
]

// 响应式数据
const contentList = ref([...initData])
const isLoading = ref(false)
const hasMoreData = ref(true)
let currentPage = 1

// 数据请求方法
const fetchContent = async () => {
  if (!hasMoreData.value || isLoading.value) return

  isLoading.value = true
  try {
    console.log('fetch')
  } finally {
    isLoading.value = false
  }
}

// 高度计算规则
const calcItemHeight = (item) => {
  if (item.videoUrl) return 540  // 视频类型
  if (item.tag === '原神') return 400 // 特殊标签
  return Math.random() * 200 + 300 // 随机高度
}

// 点赞处理
const handleLikeEvent = (item) => {
  const target = contentList.value.find(i => i.id === item.id)
  if (target) {
    target.like += item.liked ? -1 : 1
    target.liked = !item.liked
  }
}

// 初次加载
onMounted(() => {
  // fetchContent()
})
</script>

<style lang="scss">
.header {
  position: fixed;
  top: 0;
  left: 0;
  background-color: #c4c4c4;
  height: 2rem;
  width: 100%;
}

.gaming-content-container {
  max-width: 1440px;
  margin: 0 auto;
  padding: 20px;

  @media (max-width: 768px) {
    padding: 0;
  }
}
</style>
