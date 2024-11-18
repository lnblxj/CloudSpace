<template>
  <div id="app">
    <!-- 公告 -->
    <div class="announcement" v-if="showAnnouncement" :class="{ 'fade-out': isAnnouncementFading }">
      {{ announcement }}
    </div>
    
    <!-- 标题栏 -->
    <div class="header animate-card">
      <span>Cloud Space</span>
      <i class="fa fa-hdd-o"></i>
    </div>
    
    <!-- 路径导航 -->
    <div class="path-nav animate-card">
      <span>{{ currentPath }}</span>
    </div>
    
    <!-- 文件列表 -->
    <div class="file-list animate-card">
      <TransitionGroup 
        name="list" 
        :css="false"
        @before-enter="onBeforeEnter"
        @enter="onEnter"
        @leave="onLeave"
        @before-leave="onBeforeLeave"
        :tag="'div'"
        :class="{ 'switching': isSwitching }">
        <div class="file-item back-item" 
             v-if="currentPath !== '根目录' && !isSwitching" 
             @click="goBack" 
             :key="'back'">
          <i class="fa fa-arrow-up"></i>
          <span>返回上一级</span>
        </div>
        <div class="file-item" 
             v-for="(file, index) in displayFiles" 
             :key="file.filename + currentPath"
             :data-index="index">
          <!-- 左侧：文件图标和名称 -->
          <div class="file-info" @click="file.type === 'folder' ? openFolder(file) : null">
            <div class="file-icon">
              <i :class="getFileIcon(file)"></i>
            </div>
            <div class="filename">{{ file.filename }}</div>
          </div>
          
          <!-- 中间：文件类型和大小 -->
          <div class="file-meta">
            <div class="meta-item">{{ file.type === 'folder' ? '目录' : file.extension.toUpperCase() }}</div>
            <div class="meta-item">{{ file.type === 'folder' ? '' : formatFileSize(file.size) }}</div>
          </div>
          
          <!-- 右侧：操作按钮 -->
          <div class="file-actions">
            <template v-if="file.type !== 'folder'">
              <button class="action-btn" @click.stop="copyLink(file.link)" title="复制链接">
                <i class="fa fa-link"></i>
              </button>
              <button class="action-btn" @click.stop="downloadFile(file.link)" title="下载">
                <i class="fa fa-download"></i>
              </button>
            </template>
          </div>
        </div>
      </TransitionGroup>
    </div>

    <!-- 文件描述提示 -->
    <div class="description-tooltip" v-if="showDescriptionText">
      {{ descriptionText }}
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { TransitionGroup } from 'vue'
import axios from 'axios'
import gsap from 'gsap'

const files = ref([])
const announcement = ref('')
const showAnnouncement = ref(true)
const isAnnouncementFading = ref(false)
const currentPath = ref('根目录')
const showDescriptionText = ref(false)
const descriptionText = ref('')

let directoryData = null

const isNavigatingToChild = ref(false)
const isSwitching = ref(false)
const displayFiles = computed(() => isSwitching.value ? [] : sortedFiles.value)

onMounted(async () => {
  try {
    const response = await axios.get('https://cnhkbbs.github.io/jstools/cloudspace/filelist.json')
    directoryData = response.data.directory
    updateFileList()
    announcement.value = directoryData.announcement
    setTimeout(() => {
      isAnnouncementFading.value = true
      setTimeout(() => {
        showAnnouncement.value = false
      }, 1000)
    }, 10000)
  } catch (error) {
    console.error('Error fetching data:', error)
  }
})

const updateFileList = async () => {
  await new Promise(resolve => setTimeout(resolve, 100))
  
  let currentDir = directoryData
  if (currentPath.value !== '根目录') {
    const pathArray = currentPath.value.split('/')
    for (const path of pathArray.slice(1)) {
      currentDir = currentDir.subdirectories.find(subdir => subdir.name === path)
    }
  }
  
  // 更新文件列表
  files.value = [
    ...currentDir.subdirectories.map(subdir => ({ filename: subdir.name, type: 'folder' })),
    ...currentDir.files.map(file => ({ 
      ...file, 
      type: 'file', 
      extension: file.filename.split('.').pop(),
      link: file.downloadUrl
    }))
  ]
}

const sortedFiles = computed(() => {
  return files.value.sort((a, b) => a.type === 'folder' ? -1 : 1)
})

const formatFileSize = (size) => {
  if (size < 1024) {
    return `${size} B`
  } else if (size < 1024 * 1024) {
    return `${(size / 1024).toFixed(2)} KB`
  } else {
    return `${(size / (1024 * 1024)).toFixed(2)} MB`
  }
}

const showTooltip = (message) => {
  console.log(message)
}

const hideTooltip = () => {
  console.log('隐藏提示')
}

const showDescription = (description) => {
  if (description) {
    descriptionText.value = description
    showDescriptionText.value = true
  }
}

const hideDescription = () => {
  showDescriptionText.value = false
}

const copyLink = (link) => {
  if (link) {
    navigator.clipboard.writeText(link).then(() => {
      console.log('链接已复制')
    }).catch(err => {
      console.error('复制失败:', err)
    })
  } else {
    console.error('链接为空')
  }
}

const downloadFile = (link) => {
  if (link) {
    window.open(link, '_blank')
  } else {
    console.error('下载链接为空')
  }
}

// 列表项进入前的处理
const onBeforeEnter = (el) => {
  el.style.opacity = '0'
  el.style.transform = 'translateX(30px)'
}

// 列表项进入动画
const onEnter = (el, done) => {
  const delay = el.dataset.index * 50
  setTimeout(() => {
    el.style.transition = 'all 0.3s ease'
    el.style.opacity = '1'
    el.style.transform = 'translateX(0)'
  }, delay)

  setTimeout(done, delay + 300)
}

// 列表项离开动画
const onLeave = (el, done) => {
  if (isNavigatingToChild.value) {
    // 如果是导航到子目录，立即移除
    done()
  } else {
    // 其他情况（如返回上级）使用动画
    el.style.transition = 'all 0.3s ease'
    el.style.opacity = '0'
    el.style.transform = 'translateX(-30px)'
    setTimeout(done, 300)
  }
}

// 列表项离开前的处理
const onBeforeLeave = (el) => {
  el.style.position = 'absolute'
  el.style.width = '100%'
}

// 打开文件夹
const openFolder = async (folder) => {
  isSwitching.value = true
  await new Promise(resolve => setTimeout(resolve, 100))
  
  currentPath.value += `/${folder.filename}`
  await updateFileList()
  
  await new Promise(resolve => setTimeout(resolve, 50))
  isSwitching.value = false
}

// 返回上级
const goBack = async () => {
  if (!currentPath.value.includes('/')) return
  
  isSwitching.value = true
  await new Promise(resolve => setTimeout(resolve, 100))
  
  const pathArray = currentPath.value.split('/')
  pathArray.pop()
  currentPath.value = pathArray.join('/')
  await updateFileList()
  
  await new Promise(resolve => setTimeout(resolve, 50)) 
  isSwitching.value = false
}


const onEnterWithGSAP = (el, done) => {
  const delay = el.dataset.index * 0.05
  gsap.from(el, {
    duration: 0.3,
    opacity: 0,
    x: 30,
    delay,
    onComplete: done
  })
}

const onLeaveWithGSAP = (el, done) => {
  if (isNavigatingToChild.value) {
    done()
  } else {
    gsap.to(el, {
      duration: 0.3,
      opacity: 0,
      x: -30,
      onComplete: done
    })
  }
}


const formatFileName = (filename) => {
  if (window.innerWidth <= 768) { 
    if (filename.length > 10) {
      return filename.slice(0, 8) + '...' + filename.slice(filename.lastIndexOf('.'));
    }
  }
  return filename;
}


const fileIconMap = {
  // 文本文件
  'txt': 'fa fa-file-text-o',
  'doc': 'fa fa-file-word-o',
  'docx': 'fa fa-file-word-o',
  'pdf': 'fa fa-file-pdf-o',
  'md': 'fa fa-file-text-o',
  
  // 图片文件
  'jpg': 'fa fa-file-image-o',
  'jpeg': 'fa fa-file-image-o',
  'png': 'fa fa-file-image-o',
  'gif': 'fa fa-file-image-o',
  'webp': 'fa fa-file-image-o',
  
  // 压缩文件
  'zip': 'fa fa-file-archive-o',
  'rar': 'fa fa-file-archive-o',
  '7z': 'fa fa-file-archive-o',
  
  // 音视频文件
  'mp3': 'fa fa-file-audio-o',
  'wav': 'fa fa-file-audio-o',
  'mp4': 'fa fa-file-video-o',
  'avi': 'fa fa-file-video-o',
  
  // 代码文件
  'js': 'fa fa-file-code-o',
  'css': 'fa fa-file-code-o',
  'html': 'fa fa-file-code-o',
  'json': 'fa fa-file-code-o',
  'py': 'fa fa-file-code-o',
  
  // 表格文件
  'xls': 'fa fa-file-excel-o',
  'xlsx': 'fa fa-file-excel-o',
  
  // 幻灯片
  'ppt': 'fa fa-file-powerpoint-o',
  'pptx': 'fa fa-file-powerpoint-o'
}

// 获取文件图标
const getFileIcon = (file) => {
  if (file.type === 'folder') {
    return 'fa fa-folder'
  }
  
  const extension = file.filename.split('.').pop().toLowerCase()
  return fileIconMap[extension] || 'fa fa-file-o' 
}
</script>

<style>
@import url('https://s4.zstatic.net/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css');

#app {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  max-width: 1000px;
  margin: 20px auto;
  padding: 0 20px;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 15px 20px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
.header span{
	font-size: 26px;
}

.path-nav {
  margin: 20px 0;
  padding: 10px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.file-list {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  overflow: hidden;
  position: relative;
  min-height: 100px; 
}

.file-item {
  display: flex;
  align-items: center;
  padding: 12px 20px;
  border-bottom: 1px solid #eee;
}


.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 40%;
  min-width: 200px;
  padding-right: 20px;
  cursor: pointer;
}

.file-icon {
  flex-shrink: 0;
  width: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.filename {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  text-align: left;
}


.file-meta {
  display: flex;
  justify-content: center;
  align-items: center;
  flex: 1;
  min-width: 200px;
  gap: 40px;
  padding: 0 20px;
}

.meta-item {
  min-width: 60px;
  text-align: center;
  color: #666;
}


.file-actions {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  min-width: 100px;
  margin-left: auto;
}

.action-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  padding: 0;
  background: none;
  border: 1px solid #eee;
  border-radius: 4px;
  cursor: pointer;
  color: #666;
  transition: all 0.2s ease;
}

.action-btn:hover {
  color: #1e90ff;
  border-color: #1e90ff;
  background-color: rgba(30, 144, 255, 0.1);
}

i.fa {
  font-size: 16px;
}

/* 响应式布局 */
@media screen and (max-width: 768px) {
  .file-item {
    padding: 12px 10px;
  }

  .file-info {
    width: auto;
    min-width: 0;
    flex: 1;
  }

  .file-meta {
    display: none;
  }

  .file-actions {
    min-width: auto;
  }

  .action-btn {
    width: 28px;
    height: 28px;
  }
}

/* 超小屏幕适配 */
@media screen and (max-width: 360px) {
  .file-item {
    padding: 12px 10px;
  }
  
  .filename {
    max-width: 150px;
  }
}

.announcement {
  position: fixed;
  top: 20px;
  right: 20px;
  max-width: 300px;
  padding: 15px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  z-index: 1000;
  transition: opacity 1s;
}

.fade-out {
  opacity: 0;
}

.description-tooltip {
  position: fixed;
  max-width: 300px;
  padding: 10px;
  background-color: rgba(0, 0, 0, 0.8);
  color: white;
  border-radius: 4px;
  font-size: 14px;
  z-index: 1000;
  pointer-events: none;
  top: 0;
  left: 0;
}

/* 移动端适配 */
@media screen and (max-width: 768px) {
  .announcement {
    max-width: calc(100% - 40px);
    top: 10px;
    right: 10px;
    left: 10px;
  }
}

/* 卡片加载动画 */
.animate-card {
  animation: slideIn 0.5s ease-out;
  animation-fill-mode: both;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 文件列表项动画 */
.list-enter-active,
.list-leave-active,
.list-enter-from,
.list-leave-to {
}

/* 文件项淡入动画 */
.file-item {
  position: relative;
  display: flex;
  align-items: center;
  padding: 12px 20px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
}

.header,
.path-nav,
.file-list {
  transition: all 0.3s ease;
}

/* 文件夹打开/返回动画 */
.file-list {
  position: relative;
  perspective: 1000px;
}

.file-list-enter-active,
.file-list-leave-active {
  transition: all 0.3s ease;
}

.file-list-enter-from {
  opacity: 0;
  transform: translateX(30px) rotateY(-10deg);
}

.file-list-leave-to {
  opacity: 0;
  transform: translateX(-30px) rotateY(10deg);
}

.switching .file-item {
  pointer-events: none; /* 切换时禁用交互 */
}

/* TransitionGroup 容器样式 */
.list-move, 
.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}

.list-leave-active {
  position: absolute;
}


.fa-file-pdf-o { color: #ff4444; }
.fa-file-word-o { color: #2b579a; }
.fa-file-excel-o { color: #217346; }
.fa-file-powerpoint-o { color: #d24726; }
.fa-file-image-o { color: #32a852; }
.fa-file-audio-o { color: #9c27b0; }
.fa-file-video-o { color: #e91e63; }
.fa-file-archive-o { color: #795548; }
.fa-file-code-o { color: #0097a7; }
.fa-folder { color: #ffd700; }

@media screen and (min-width: 768px) {
  .file-actions {
    opacity: 1;
    visibility: visible;
    display: flex !important;
  }
  
  .action-btn {
    opacity: 1;
    visibility: visible;
  }
}

html {
  background-image: url('https://cdn-hsyq-static.shanhutech.cn/bizhi/staticwp/202408/8678d97dc81e1436b34b19a49e55f4bd--946961122.jpg');
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
  min-height: 100vh;
  width: 100%;
}

</style>