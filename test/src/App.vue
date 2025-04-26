<template>
  <div class="header"></div>
  <div class="gaming-content-container">
    <!-- 瀑布流区域 -->
    <WaterfallComponent 
      :items="contentList" 
      :loading="isLoading" 
      :hasMore="hasMoreData" 
      @load-more="fetchLoadMoreContent"
      @refresh="fetchRefreshContent"
      @like="handleLikeEvent" />
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import WaterfallComponent from './components/TWaterFall.vue'

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
const userid = ref(1);

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
const fetchLoadMoreContent = async () => { // 该方法为获取更多，api和fetchRefreshContent相同，只是获取的值拼接在contentList后面
  console.log('load-more')
  fetch(`http://localhost:3000/data/${userid.value}`)
    .then(response => response.json())
    .then(data => {
      console.log('data:', data)
      contentList.value = [...contentList.value, ...data.data.map(item => ({
        ...item,
        // height: calcItemHeight(item) // 计算高度
      }))]
      hasMoreData.value = data.length > 0
    })
    .catch(error => {
      console.error('Error fetching data:', error)
    })
}
const fetchRefreshContent = async () => { //服务器api链接为http://localhost:3000/data/:userid,该函数功能为请求api数据，get请求
  console.log('refresh')
  console.log('userid:', userid.value)
  fetch(`http://localhost:3000/data/${userid.value}`)
    .then(response => response.json())
    .then(data => {
      console.log('data:', data)
      contentList.value = data.data.map(item => ({
        ...item,
        // height: calcItemHeight(item) // 计算高度
      }))
      hasMoreData.value = data.length > 0
    })
    .catch(error => {
      console.error('Error fetching data:', error)
    })
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

watch(userid, (newVal) => {
  console.log('userId changed:', newVal)
  // 这里可以添加根据 userId 获取数据的逻辑
})

// 初次加载
onMounted(() => {
  // fetchContent()
  const route = window.location.href
  const viewUser = new URLSearchParams(window.location.search).get('viewuser')
  if (viewUser) {
    userid.value = viewUser
  }
  fetchRefreshContent()
  // userid.value = 
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
  z-index: 1000;
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
