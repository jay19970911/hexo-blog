---
title: 【PDF.js】PDF文件预览
---
使用PDFJS实现pdf文件的预览，支持预览指定页、关键词搜索、缩略图、页面尺寸调整等等。

# 一、PDF.js
[官方地址](https://mozilla.github.io/pdf.js/)
[文档地址](https://gitcode.gitcode.host/docs-cn/pdf.js-docs-cn/print.html)

# 二、PDF.js 下载
## 1、下载PDF.js
[下载地址](https://mozilla.github.io/pdf.js/getting_started/)

![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/2c10917b12334fbd8d923af03d2f4528.png)
## 2、在项目中引入
将下载的压缩包解压并放入到项目中的public文件夹下，我这里下载的是pdfjs-4.0.379-dist版本，如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/8092e8c61dda4189b6d585b6895f5bfa.png)
## 3、屏蔽跨域错误
在 pdfjs-4.0.379-dist/web/viewer.mjs 内搜索 throw new Error("file origin does not match viewer's") 并注释，如果不注释，可能会出现跨域错误，无法正常预览文件。
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/4c0c5a74bd8b41068699a2b5bc55775e.png)
# 三、项目中使用
内容区域结构(文件预览区域、滑块区域、问答区域)
滑块区域：滑动改变pdf文件预览区域的大小
```javascript
<div
    v-if="filePreviewStore.getFilePreviewFlag"
    ref="resizeBox"
    class="resize"
    @mousedown="onResizeMouseDown"
  />
```
```javascript
<el-main ref="mainContent" class="main-content">
  <!-- 文件预览区域 -->
  <div
    v-if="filePreviewStore.getFilePreviewFlag"
    class="preview-box"
    :style="{width: `${previewBoxWidth}px`}"
  >
    <!-- <PdfPreview v-if="['ppt', 'pptx', 'pdf'].includes(filePreviewStore.getFileType)" /> -->
    <PDF v-if="['ppt', 'pptx', 'pdf'].includes(filePreviewStore.getFileType)" />
    <ExcelPreview v-if="['xls', 'xlsx'].includes(filePreviewStore.getFileType)" />
    <WordPreview v-if="['doc', 'docx'].includes(filePreviewStore.getFileType)" />
    <TxtPreview v-if="filePreviewStore.getFileType === 'txt'" />
  </div>
  <div
    v-if="filePreviewStore.getFilePreviewFlag"
    ref="resizeBox"
    class="resize"
    @mousedown="onResizeMouseDown"
  />
  <!-- 问答区域 -->
  <div class="main_side3 flex1 column-flex">
    <div class="show_content flex1" ref="chatShowRef"> <ChatShow /> </div>
    <div class="chat">
      <askInput />
    </div>
  </div>
</el-main>
```
下面是PDF组件完整代码
```javascript
<template>
  <div class="container">
    <iframe id="myIframe" :src="pdfUrl" width="100%" height="100%"></iframe>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useFilePreviewStore } from "@/stores";
import { fileRouteUrl } from "@/utils/fileRouteUrl"

const filePreviewStore = useFilePreviewStore()
const pdfUrl = ref('') // pdf文件地址
const fileUrl = '/static/dist/pdfjs-4.0.379-dist/web/viewer.html?file=' // pdfjs文件地址

onMounted(() => {
  // encodeURIComponent() 函数可把字符串作为 URI 组件进行编码。
  // 核心就是将 iframe 的 src 属性设置为 pdfjs 的地址，然后将 pdf 文件的地址作为参数传递给 pdfjs
  // 例如：http://localhost:8080/pdfjs-4.0.189-dist/web/viewer.html?file=http%3A%2F%2Flocalhost%3A8080%2Fpdf%2Ftest.pdf
  const url = filePreviewStore.getFileUrl.replace(fileRouteUrl, '/file')
  pdfUrl.value = fileUrl + encodeURIComponent(url) + `&page=${filePreviewStore.getPageNum}`
})

// 当文档页码修改时，重新预览当前页的文档内容
watch(() => filePreviewStore.getPageNum,
  (val) => {
    if (val) {
      // 页码修改时，需要重新保存记录文档页码，否则会出现点击与第一次相同的页码时，不会切换
      filePreviewStore.setFilePage(val)
      const pdfFrame = document.getElementById('myIframe').contentWindow
      // 传递参数,跳转到指定页
      pdfFrame.PDFViewerApplication.pdfViewer.scrollPageIntoView({
        pageNumber: val
      })
    }
  }
)

// 当预览的文件地址修改时，预览新的文档
watch(() => filePreviewStore.getFileUrl,
  (val) => {
    if (val) {
      // 服务器文档地址
      const pdfFileUrl = val.replace(fileRouteUrl, '/file');
      // 加载PDF文件
      pdfUrl.value = fileUrl + encodeURIComponent(pdfFileUrl) + `&page=${filePreviewStore.getPageNum}`
    }
  }
)

</script>

<style scoped lang="less">
.container {
  width: 100%;
  height: 100%;
  border: 1px solid #ccc;
  box-sizing: border-box;
  #myIframe {
    border: none;
  }
}
</style>

```

# 四、说明
1）在文件地址后面添加参数page(预览指定页)
{%  image
    url="https://pic.imgdb.cn/item/661f8a540ea9cb140373e413.png"
    title=""
%}
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/c4b86a3c8ecb4ae78e79df26698873b4.png)
2）在 pdfjs-4.0.379-dist/web/viewer.mjs中的setInitialView方法中添加如下代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/b48b815552c547b0b3afc5279585a1fe.png)
3）改变文件预览区域的宽度
```javascript
// 修改左侧文件预览区域的宽度
const previewBoxWidth = ref(0)
const mainContent = ref()
const resizeBox = ref()
const mainContentWidth = ref(0)
const onResizeMouseDown = (e: MouseEvent) => {
  const startX = e.clientX
  resizeBox.value.left = resizeBox.value.offsetLeft
  // 解决预览pdf文档时，鼠标移入iframe后，无法捕获移动和抬起操作
  const myIframe = document.querySelector('iframe')
  myIframe && (myIframe.style['pointer-events'] = 'none')
  const onDocumentMouseMove = (e: MouseEvent) => {
    const endX = e.clientX
    const previewWidth = resizeBox.value.left + (endX - startX) - side1Width.value - 20
    // 文件预览区域宽度最小为内容区域的30%，最大为内容区域的70%
    if (previewWidth >= mainContentWidth.value * 0.7) {
      previewBoxWidth.value = mainContentWidth.value * 0.7
    } else if (previewWidth <= mainContentWidth.value * 0.3) {
      previewBoxWidth.value = mainContentWidth.value * 0.3
    } else {
      previewBoxWidth.value = previewWidth
    }
  }

  const onDocumentMouseUp = () => {
    myIframe && (myIframe.style['pointer-events'] = 'auto')
    document.removeEventListener('mousemove', onDocumentMouseMove)
    document.removeEventListener('mouseup', onDocumentMouseUp)
    resizeBox.value.releaseCapture && resizeBox.value.releaseCapture()
  }

  document.addEventListener('mousemove', onDocumentMouseMove)
  document.addEventListener('mouseup', onDocumentMouseUp)
  resizeBox.value.setCapture && resizeBox.value.setCapture()
}

// 
const { width } = useWindowSize() // 响应式获取窗口尺寸
// 当浏览器窗口尺寸改变时，重新修改设置文件预览区域的宽度
watch(() => width.value,
  (val) => {
    val && (previewBoxWidth.value = mainContentWidth.value * 0.7)
  }
)

// 获取内容区域的宽度
useResizeObserver(mainContent , (entries) => {
  const entry = entries[0]
  const { width } = entry.contentRect
  mainContentWidth.value = width
})
```
这里需要注意，因为在PDF组件中使用了iframe，当鼠标移入iframe区域时，无法捕获到鼠标的移动和抬起动作，会出现鼠标移出iframe区域后有可以改变该区域宽度，解决办法如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/6f01d9957cbf436a99fb9f535afb3b21.png)
# 五、实现效果
![!\[在这里插入图片描述\](https://img-blog.csdnimg.cn/direct/2b286a949f564cd8958e352b8afe50b9.png
!\[在这里插入图片描述\](https://img-blog.csdnimg.cn/direct/7fc5171aca4e4542b6817910ecab54e9.png](https://img-blog.csdnimg.cn/direct/b24773a2e28546b4a04dcaabfd621390.png)