<template>
  <div class="sand-table-wrapper">
    <div class="sand-table-box" ref="sandTableRef">
      <div
        class="sand-img-box"
        ref="imgBoxRef"
        @mousedown="imgMouseDownEvent"
        @mousemove="imgMouseMoveEvent"
        @mouseup="imgMouseUpEvent"
        draggable="true"
      >
        <img :src="imgUrl" style="display: block" @load="imgLoad" ref="imgRef" />
        <div
          class="marker"
          v-for="(item, index) in markerGroup"
          :key="index"
          :style="{ left: (item as any).x + 'px',top: (item as any).y + 'px' }"
          @mousedown="markerMouseDownEvent(item, $event)"
        ></div>
      </div>
      <div class="scale-btn" @click="scalePic">{{ isScaled ? '缩小' : '放大' }}</div>
    </div>
    <div class="test">
      <div>开始鼠标点位:X:{{ mousePosition.startX }} Y:{{ mousePosition.startY }}</div>
      <div>当前鼠标点位:X:{{ mousePosition.currentX }} Y:{{ mousePosition.currentY }}</div>
      <div>偏移量：X:{{ mousePosition.offsetX }} Y:{{ mousePosition.offsetY }}</div>
      <div class="test-btn" @click="addMarker">添加标记点</div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { reactive } from 'vue'
import { ref, toRefs } from 'vue'

// let props = defineProps<{ mapStateObj: any }>();
// const { mapStateObj } = toRefs(props);

let imgUrl = new URL(`../assets/test.jpg`, import.meta.url).href
// 沙盘dom节点
let sandTableRef = ref()
// 图片dom节点
let imgRef = ref()
// 图片包裹dom节点
let imgBoxRef = ref()
// 是否已经缩放
let isScaled = ref(false)
// 缩放比例
let scaleRatio = ref(3)
// 图是片否可以移动
let imgCanMove = ref(false)
// 图是片否正在移动
let imgIsMoving = ref(false)
// 鼠标在图片上的位置
let mousePosition = reactive({
  startX: 0,
  startY: 0,
  currentX: 0,
  currentY: 0,
  offsetX: 0,
  offsetY: 0
})
// 图片移动的极限
let translateLimit = reactive({
  maxY: 0,
  minY: 0,
  maxX: 0,
  minX: 0,
  canXmove: true,
  canYmove: true
})
// 图片移动距离是否受父级窗口限制
let isLimit = ref(true)

// 标记点
let markerGroup: any = ref([] as any)
// 标记点是否可移动
let markerCanMove = ref(true)

// 图片加载完成
function imgLoad() {
  //获取沙盘窗口的尺寸
  let sandTableSize = sandTableRef.value.getBoundingClientRect()
  // 获取图片的尺寸
  let imgSize = imgRef.value.getBoundingClientRect()

  // 判断图片尺寸是否大于窗口尺寸
  if (imgSize.width > sandTableSize.width || imgSize.height > sandTableSize.height) {
    // 获取图片比例
    let picRatio = imgSize.width / imgSize.height
    if (picRatio >= 1) {
      // 即宽图
      imgRef.value.style.width = sandTableSize.width + 'px'
      imgRef.value.style.height = sandTableSize.width / picRatio + 'px'
    } else {
      // 即长图
      imgRef.value.style.width = sandTableSize.width * picRatio + 'px'
      imgRef.value.style.height = sandTableSize.height + 'px'
    }
    imgBoxRef.value.style.transform = 'scale(1) translate(0px,0px)'
  }
  console.log(imgBoxRef.value.offsetTop)
}

// 添加标记点
function addMarker() {
  // 获取图片的尺寸
  let imgSize = imgRef.value.getBoundingClientRect()
  let marker = {
    x: imgSize.width * 0.5,
    y: imgSize.height * 0.5
  }
  markerGroup.value.push(marker)
}

// 标记点鼠标按下事件
function markerMouseDownEvent(data: any, e: any) {
  console.log('data: ', data)
  console.log(e.target.offsetTop)
  e.stopPropagation()
  e.preventDefault()
}

// 缩放图片
function scalePic() {
  isScaled.value = !isScaled.value
  if (isScaled.value) {
    scalePicByRatio(scaleRatio.value)
  } else {
    scalePicByRatio(1)
  }
}

// 根据比例缩放图片
function scalePicByRatio(ratio: any) {
  imgCanMove.value = ratio > 1 ? true : false
  // 设置缩放原点
  // setTransformOrigin()
  // 设置缩放比例
  updateImgTransform('scale', `scale(${ratio})`)
  imgBoxRef.value.style.cursor = ratio > 1 ? 'move' : 'default'

  // 设置图片受窗口限制的极限值
  setTranslateLimit()
}

// 设置缩放原点
function setTransformOrigin() {
  // 图片缩放前的尺寸
  let sandTableSize = sandTableRef.value.getBoundingClientRect()
  let imgSize = imgBoxRef.value.getBoundingClientRect()
  if (imgSize.width > sandTableSize.width && imgSize.height > sandTableSize.height) {
    //图片已放大并超出父级窗口尺寸
  } else {
    imgBoxRef.value.style.transformOrigin = `0% 0%`
  }
}

// 设置图片受窗口限制的极限值
function setTranslateLimit() {
  // 图片缩放后的最新尺寸
  let sandTableSize = sandTableRef.value.getBoundingClientRect()
  let imgSize = imgBoxRef.value.getBoundingClientRect()

  let offsetTop = sandTableSize.top - imgSize.top
  let offsetLeft = sandTableSize.left - imgSize.left
  translateLimit.maxY = offsetTop / scaleRatio.value
  translateLimit.minY =
    -(imgSize.height - Math.abs(offsetTop) - sandTableSize.height) / scaleRatio.value
  translateLimit.maxX = offsetLeft / scaleRatio.value
  translateLimit.minX =
    -(imgSize.width - Math.abs(offsetLeft) - sandTableSize.width) / scaleRatio.value

  // 是否可以水平方向移动
  translateLimit.canXmove = imgSize.width > sandTableSize.width ? true : false
  // 是否可以垂直方向移动
  translateLimit.canYmove = imgSize.height > sandTableSize.height ? true : false
  console.log('translateLimit: ', translateLimit)
}

// 更新translate的值
function updateTranlateValue(offset: { x: any; y: any }) {
  // 获取原有的Translate属性值
  let orginTranslate = imgBoxRef.value.style.transform
    .split(') ')
    .filter((item: any) => item.includes('translate'))[0]
    .replace('translate(', '')
    .replace(')', '')
    .replace(/px/g, '')
    .split(', ')
    .map((item: any) => +item)
  let translateX = 0
  let translateY = 0
  let tempX = orginTranslate[0] + offset.x
  let tempY = orginTranslate[1] + offset.y
  if (isLimit.value) {
    // 图片移动距离受父级窗口限制
    if (translateLimit.canXmove) {
      // 设置X方向偏移量
      if (tempX >= translateLimit.maxX) {
        // 移动到了最左边
        translateX = translateLimit.maxX
      } else if (tempX <= translateLimit.minX) {
        // 移动到了最右边
        translateX = translateLimit.minX
      } else {
        translateX = tempX
      }
    } else {
      translateX = 0
    }
    if (translateLimit.canYmove) {
      // 设置Y方向偏移量
      if (tempY >= translateLimit.maxY) {
        // 移动到了最上边
        translateY = translateLimit.maxY
      } else if (tempY <= translateLimit.minY) {
        // 移动到了最下边
        translateY = translateLimit.minY
      } else {
        translateY = tempY
      }
    } else {
      translateY = 0
    }
  } else {
    //图片移动距离不受父级窗口限制
    translateX = tempX
    translateY = tempY
  }
  return `translate(${translateX}px, ${translateY}px)`
}

// 更新图片transform属性值
function updateImgTransform(attr: string, params: string) {
  // 获取原有的transform属性值
  let orginTransform = imgBoxRef.value.style.transform.split(') ')
  let newTransform: any = []
  // 更新原transform属性值中的scale数值
  orginTransform.forEach((item: any) => {
    let temp = item.includes(attr) ? params : item
    newTransform.push(temp.includes(')') ? temp : `${temp})`)
  })
  imgBoxRef.value.style.transform = newTransform.join(' ').trim()
}

// 图片鼠标按下事件
function imgMouseDownEvent(e: any) {
  if (imgCanMove.value) {
    e.preventDefault()
    mousePosition.startX = e.offsetX
    mousePosition.startY = e.offsetY
    mousePosition.currentX = e.offsetX
    mousePosition.currentY = e.offsetY
    mousePosition.offsetX = 0
    mousePosition.offsetY = 0
    imgIsMoving.value = true
  }
}

// 图片鼠标移动事件
function imgMouseMoveEvent(e: any) {
  if (imgCanMove.value && imgIsMoving.value) {
    e.preventDefault()

    mousePosition.currentX = e.offsetX
    mousePosition.currentY = e.offsetY
    mousePosition.offsetX = mousePosition.currentX - mousePosition.startX
    mousePosition.offsetY = mousePosition.currentY - mousePosition.startY

    updateImgTransform(
      'translate',
      updateTranlateValue({
        x: mousePosition.offsetX,
        y: mousePosition.offsetY
      })
    )
  }
}

// 图片鼠标抬起事件
function imgMouseUpEvent(e: any) {
  if (imgCanMove.value) {
    e.preventDefault()
    imgIsMoving.value = false
    mousePosition.offsetX = 0
    mousePosition.offsetY = 0
  }
}
</script>
<style scoped lang="scss">
.sand-table-box {
  width: 600px;
  height: 600px;
  border: 2px solid rebeccapurple;
  box-sizing: border-box;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;

  .sand-img-box {
    float: left;
    position: relative;
  }

  .scale-btn {
    position: absolute;
    width: 120px;
    height: 40px;
    text-align: center;
    line-height: 40px;
    cursor: pointer;
    right: 16px;
    bottom: 16px;
    z-index: 2;
    background-color: blueviolet;
    color: #fff;
    border-radius: 6px;
  }

  .marker {
    width: 14px;
    height: 14px;
    background: #031842;
    border: 2px solid #ffffff;
    border-radius: 50%;
    position: absolute;
    cursor: pointer;
  }
}

.test {
  .test-btn {
    width: 120px;
    height: 40px;
    text-align: center;
    line-height: 40px;
    cursor: pointer;
    background-color: blueviolet;
    color: #fff;
    border-radius: 6px;
  }
}
</style>
