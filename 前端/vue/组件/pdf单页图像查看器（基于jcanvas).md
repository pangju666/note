```javascript
"jcanvas": "^21.0.1",
"jquery": "^3.6.0",
```

```
<template>
  <div id="container">
    <div id="outerContainer">
      <div v-show="showSidebar" id="sidebarContainer">
        <!-- 侧边工具栏 -->
        <div id="toolbarSidebar">
          <el-checkbox
            v-if="checkbox"
            id="select-all-checkbox"
            v-model="selectAllValue"
            title="全选"
            class="m-5"
          />
          <span
            v-if="showDownload"
            title="下载"
            class="cursor-pointer m-5 v-center"
            @click="handleClickDownload"
          >
            <el-icon id="download-icon" :size="20">
              <download />
            </el-icon>
          </span>
        </div>
        <!-- 侧边栏内容 -->
        <div id="sidebarContent">
          <!-- 缩略图 -->
          <div id="thumbnailView" ref="thumbnailView">
            <ul id="thumbnail-list">
              <li
                v-for="(page, index) in pages"
                :id="`thumbnail-${page[props.pageNumber]}`"
                :key="index"
                class="list-item"
              >
                <el-image
                  :src="page[props.pageThumbnail]"
                  class="thumbnail"
                  :fit="fit"
                  lazy
                  @click="handleClickPageThumbnail(page[props.pageNumber])"
                />
                <div class="item-bottom">
                  <input
                    v-show="checkbox"
                    v-model="pageSelectStatus[index].select"
                    type="checkbox"
                    class="checkbox"
                  />
                  <span class="page-number">{{ page[props.pageNumber] }}</span>
                </div>
              </li>
            </ul>
          </div>
        </div>
      </div>
      <div id="mainContainer">
        <!-- 工具栏        -->
        <div class="toolbar">
          <div id="toolbarContainer">
            <div id="toolbarViewer">
              <div id="toolbarViewerLeft">
                <button
                  id="sidebarToggle"
                  class="toolbarButton"
                  title="缩略图"
                  @click="showSidebar = !showSidebar"
                >
                  <span>缩略图</span>
                </button>
                <div class="toolbarButtonSpacer" />
                <div class="splitToolbarButton hiddenSmallView">
                  <button
                    id="previous"
                    class="toolbarButton pageUp"
                    title="上一页"
                    @click="handlePrevious"
                  >
                    <span>上一页</span>
                  </button>
                  <div class="splitToolbarButtonSeparator" />
                  <button
                    id="next"
                    class="toolbarButton pageDown"
                    title="下一页"
                    @click="handleNext"
                  >
                    <span>下一页</span>
                  </button>
                </div>
                <input
                  id="pageNumber"
                  v-model.number="currentPage"
                  class="toolbarField pageNumber"
                  title="页码"
                  size="4"
                  autocomplete="off"
                  @change="handlePageNumberChange"
                />
                <span id="numPages" class="toolbarLabel"
                  >/{{ totalPages }}</span
                >
              </div>
              <div id="toolbarViewerRight">
                <div class="splitToolbarButton">
                  <button
                    id="zoomOut"
                    class="toolbarButton zoomOut"
                    title="缩小"
                    @click="handleZoomOut"
                  >
                    <span>缩小</span>
                  </button>
                  <div class="splitToolbarButtonSeparator" />
                  <button
                    id="zoomIn"
                    class="toolbarButton zoomIn"
                    title="放大"
                    @click="handleZoomIn"
                  >
                    <span>放大</span>
                  </button>
                </div>
                <span
                  id="scaleSelectContainer"
                  class="dropdownToolbarButton"
                  style="margin: 2px 1px"
                >
                  <select
                    id="scaleSelect"
                    ref="scaleSelect"
                    title="缩放"
                    @change="handleScaleChange"
                  >
                    <option
                      id="pageAutoOption"
                      title="自动缩放"
                      value="page-auto"
                      selected
                    >
                      自动缩放
                    </option>
                    <option id="pageActualOption" :value="1.0" title="实际大小">
                      实际大小
                    </option>
                    <option
                      id="pageFitOption"
                      title="适合页面"
                      value="page-fit"
                    >
                      适合页面
                    </option>
                    <option
                      id="pageWidthOption"
                      title="适合页宽"
                      value="page-width"
                    >
                      适合页宽
                    </option>
                    <option
                      v-show="customScale"
                      id="customScaleOption"
                      title="自定义缩放"
                      value="custom"
                    >
                      {{ customScaleLabel }}
                    </option>
                    <option :value="0.5" title="50%">50%</option>
                    <option :value="0.75" title="75%">75%</option>
                    <option :value="1" title="100%">100%</option>
                    <option :value="1.25" title="125%">125%</option>
                    <option :value="1.5" title="150%">150%</option>
                    <option :value="2" title="200%">200%</option>
                    <option :value="3" title="300%">300%</option>
                    <option :value="4" title="400%">400%</option>
                  </select>
                </span>
              </div>
              <div id="toolbarViewerMiddle" style="height: 100%">
                <div class="title">《{{ title }}》</div>
              </div>
            </div>
          </div>
        </div>
        <!-- 页面浏览容器        -->
        <div id="viewerContainer" ref="viewerContainer" @scroll="handleScroll">
          <div id="viewer" class="pdfViewer">
            <div
              v-for="pageNumber in totalPages"
              :key="pageNumber"
              class="page"
              :style="style"
            >
              <!-- 遮罩              -->
              <div :id="`mask-page-${pageNumber}`" class="mask">
                <div class="mask-content">
                  <img alt="" src="@/assets/images/icon/loading-icon.gif" />
                </div>
              </div>
              <div :style="style" class="canvasWrapper">
                <canvas
                  :id="`page-${pageNumber}`"
                  :ref="`page-${pageNumber}`"
                  :style="style"
                />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import "@/assets/css/viewer.css";

import { Download } from "@element-plus/icons-vue";
import { ArrayUtils, ObjectUtils } from "pangju-utils";

import $ from "jquery";
import jcanvas from "jcanvas";

jcanvas($, window);

export default {
  name: "PdfPageViewer",
  components: { Download },
  props: {
    pages: {
      type: Array,
      required: true,
    },
    props: {
      type: Object,
      default: () => {
        return {
          pageNumber: "pageNumber",
          pageThumbnail: "thumbnailUrl",
          pageSource: "pageUrl",
        };
      },
      validator(value) {
        return (
          ObjectUtils.nonNull(value.pageNumber) &&
          ObjectUtils.nonNull(value.pageThumbnail) &&
          ObjectUtils.nonNull(value.pageSource)
        );
      },
    },
    startPageNumber: {
      type: Number,
      default: 1,
    },
    totalPages: {
      type: Number,
      required: true,
    },
    title: {
      type: String,
      required: true,
    },
    checkbox: {
      type: Boolean,
      default: true,
    },
    showDownload: {
      type: Boolean,
      default: true,
    },
    fit: {
      type: String,
      default: "scale-down",
      validator(value) {
        return ["fill", "contain", "cover", "none", "scale-down"].includes(
          value
        );
      },
    },
    width: {
      type: Number,
      required: true,
    },
    height: {
      type: Number,
      required: true,
    },
    initPages: {
      type: Number,
      default: 5,
    },
    maxCachePages: {
      type: Number,
      default: 30,
    },
    maxLoadedPages: {
      type: Number,
      default: 10,
    },
    scrollTimeout: {
      type: Number,
      default: 1000,
    },
  },
  emits: ["page-change", "download"],
  data() {
    return {
      pagesData: [],
      // 是否显示侧边栏
      showSidebar: false,
      // 是否为自定义缩放
      customScale: false,
      // 是否全选
      selectAllValue: false,
      //当前页面
      currentPage: 1,
      // 缩放率
      scale: 1.0,
      // 页面源映射
      pageSourceMap: new Map(),
      // 页码映射
      pageNumberMap: new Map(),
      // 页面加载状态
      pageLoadStatus: [],
      // 页面选择状态
      pageSelectStatus: [],
      // 页面滚动状态
      scrollStatus: {
        top: 0,
        left: 0,
      },
      scrollTimer: null,
      // 滚动事件锁
      scrollLock: true,
    };
  },
  computed: {
    maxPageNumber() {
      return Math.max(
        ...this.pagesData.map((page) => page[this.props.pageNumber])
      );
    },
    pageWidth() {
      return Math.round(this.width * this.scale);
    },
    pageElementHeight() {
      const pageElement = $(".page");
      const borderHeight = Number.parseInt(
        pageElement.css("border-image-slice"),
        10
      );
      const marginTopHeight = Number.parseInt(
        pageElement.css("margin-top"),
        10
      );
      const marginBottomHeight = Number.parseInt(
        pageElement.css("margin-bottom"),
        10
      );
      return Math.round(
        this.pageHeight +
          borderHeight * 2 +
          marginTopHeight +
          marginBottomHeight
      );
    },
    pageHeight() {
      return Math.round(this.height * this.scale);
    },
    canvasMaxWidth() {
      return Math.round(this.canvasMaxHeight * (this.width / this.height));
    },
    canvasMaxHeight() {
      for (let i = 5; i > 1; i -= 0.5) {
        const height = this.height * i;
        if (height < 5000) {
          return Math.round(height);
        }
      }
      return 5000;
    },
    customScaleLabel() {
      return `${(this.scale * 100).toFixed(0)}%`;
    },
    pageAutoScale() {
      const pageFitWidth = this.width * this.pageFitScale;
      return pageFitWidth > window.innerWidth
        ? this.pageWidthScale
        : this.pageFitScale;
    },
    pageFitScale() {
      return (this.$refs.viewerContainer.offsetHeight - 18) / this.height;
    },
    pageWidthScale() {
      return (this.$refs.viewerContainer.offsetWidth - 30 - 18) / this.width;
    },
    style() {
      return {
        width: `${this.pageWidth}px`,
        height: `${this.pageHeight}px`,
      };
    },
  },
  watch: {
    currentPage(newVal, oldVal) {
      this.$emit("page-change", newVal);
      this.$refs.thumbnailView.scrollTo({
        // 270 为元素高度，页码减一保证滚动至窗口中间
        top: 270 * this.pageNumberMap.get(newVal - 1),
      });
      $(`#thumbnail-${oldVal}`).removeClass("page-active");
      $(`#thumbnail-${newVal}`).addClass("page-active");
    },
    selectAllValue(newVal) {
      if (newVal) {
        this.pageSelectStatus.forEach((page) => {
          page.select = true;
        });
      } else {
        this.pageSelectStatus.forEach((page) => {
          page.select = false;
        });
      }
    },
  },
  mounted() {
    this.$nextTick(() => {
      $(".mask").show();
      this.scale = this.pageAutoScale;
      $("canvas").attr("width", this.pageWidth).attr("height", this.pageHeight);

      const startPageNumber =
        this.startPageNumber < this.maxPageNumber
          ? this.startPageNumber
          : this.maxPageNumber;
      const initPages = Math.min(this.pagesData.length, this.initPages);
      if (this.pagesData.length <= initPages) {
        for (let i = 0; i < this.pagesData.length; i++) {
          this.renderPage(this.pagesData[i][this.props.pageNumber]);
        }
      } else {
        const endPageNumber =
          Math.min(startPageNumber + initPages, this.pagesData.length) - 1;
        for (let i = startPageNumber; i <= endPageNumber; i++) {
          this.renderPage(this.pagesData[i - 1][this.props.pageNumber]);
        }
      }

      if (startPageNumber !== 1) {
        this.currentPage = startPageNumber;
        this.jumpToPage(startPageNumber);
      }
    });
  },
  created() {
    this.pagesData = this.pages;
    this.pagesData.sort(
      (first, second) =>
        first[this.props.pageNumber] - second[this.props.pageNumber]
    );
    this.pagesData.forEach((page, index) => {
      const pageNumber = page[this.props.pageNumber];
      this.pageNumberMap.set(pageNumber, index);
      this.pageSelectStatus.push({
        pageNumber: page[this.props.pageNumber],
        select: false,
      });
    });
  },
  methods: {
    // 获取页面数据源
    getPageSource(pageNumber) {
      // 判断是否超过最大缓存页数
      if (this.pageSourceMap.size > this.maxCachePages) {
        const delPage = Math.min(...this.pageSourceMap.keys());
        this.pageSourceMap.delete(delPage);
      }
      // 判断当前页是否存在于缓存中
      if (this.pageSourceMap.has(pageNumber)) {
        return this.pageSourceMap.get(pageNumber);
      }
      // 不存在则添加
      const pageSource = new Image();
      const index = this.pageNumberMap.get(pageNumber);
      pageSource.src = this.pagesData[index][this.props.pageSource];
      this.pageSourceMap.set(pageNumber, pageSource);
      return pageSource;
    },
    // 渲染页面
    async renderPage(pageNumber) {
      // 更新页面加载状态
      this.updatePageLoadStatus(pageNumber);
      // 显示加载遮罩
      $(`#mask-page-${pageNumber}`).show();
      // 判断是否需要重绘画布
      this.resizeCanvas(pageNumber);
      // 绘制图像
      $(`#page-${pageNumber}`).drawImage({
        name: `page-${pageNumber}`,
        source: this.getPageSource(pageNumber),
        /*x: this.pageWidth / 2,
        y: this.pageHeight / 2,
        width: this.pageWidth,
        height: this.pageHeight,*/
        x: this.width / 2,
        y: this.height / 2,
        width: this.width,
        height: this.height,
        load: () => {
          // 隐藏加载遮罩
          $(`#mask-page-${pageNumber}`).hide();
        },
      });
    },
    // 修改缩放比率
    changeScale() {
      $(".mask").show();
      $("canvas").removeLayers().drawLayers();

      this.scrollStatus.top =
        this.pageElementHeight * (this.currentPage - 1) + 10;
      this.scrollLock = true;
      this.$refs.viewerContainer.scrollTo(this.scrollStatus);
      this.pageLoadStatus = [];

      if (!this.pageNumberMap.has(this.currentPage)) {
        return;
      }

      const index = this.pageNumberMap.get(this.currentPage);
      this.renderPage(this.currentPage);

      const nextIndex = index + 1;
      if (nextIndex >= 0 && nextIndex < this.pagesData.length) {
        const nexPage = this.pagesData[index + 1][this.props.pageNumber];
        this.renderPage(nexPage);
      }

      const lastIndex = index - 1;
      if (lastIndex >= 0 && lastIndex < this.pagesData.length) {
        const lastPage = this.pagesData[index - 1][this.props.pageNumber];
        this.renderPage(lastPage);
      }
    },
    // 加载与当前页码和当前页码的上下页
    loadPages(page) {
      if (!this.pageNumberMap.has(page)) {
        return;
      }

      const index = this.pageNumberMap.get(page);
      if (!this.pageLoadStatus.includes(page)) {
        this.renderPage(page);
      }

      const nextIndex = index + 1;
      if (nextIndex >= 0 && nextIndex < this.pagesData.length) {
        const nexPage = this.pagesData[index + 1][this.props.pageNumber];
        if (!this.pageLoadStatus.includes(nexPage)) {
          this.renderPage(nexPage);
        }
      }

      const lastIndex = index - 1;
      if (lastIndex >= 0 && lastIndex < this.pagesData.length) {
        const lastPage = this.pagesData[index - 1][this.props.pageNumber];
        if (!this.pageLoadStatus.includes(lastPage)) {
          this.renderPage(lastPage);
        }
      }
    },
    // 重置画布
    resizeCanvas(pageNumber) {
      const pageCanvas = $(`#page-${pageNumber}`);
      /* // 判断是否需要修改canvas尺寸
       const canvasWidth = pageCanvas.attr("width");
       const actualWidth = Math.min(this.pageWidth, this.canvasMaxWidth);
       if (canvasWidth !== actualWidth) {
         pageCanvas.attr("width", actualWidth);
       }
       const canvasHeight = pageCanvas.attr("height");
       const actualHeight = Math.min(this.pageHeight, this.canvasMaxHeight);
       if (canvasHeight !== actualHeight) {
         pageCanvas.attr("height", actualHeight);
       }*/
      pageCanvas.attr("width", this.width);
      pageCanvas.attr("height", this.height);
    },
    // 更新页面缓存
    updatePageLoadStatus(pageNumber) {
      // 判断是否超过最大缓存页数
      if (this.pageLoadStatus.length > this.maxLoadedPages) {
        // 清除第一条页面数据缓存
        const delPage = this.pageLoadStatus.shift();
        $(`#page-${delPage}`).removeLayers().drawLayers();
      }
      // 判断当前页是否存在于缓存中
      if (!this.pageLoadStatus.includes(pageNumber)) {
        // 不存在则添加
        this.pageLoadStatus.push(pageNumber);
      }
    },
    // 跳转至指定页码
    jumpToPage(pageNumber) {
      this.scrollStatus.top = this.pageElementHeight * (pageNumber - 1);
      this.scrollLock = true;
      this.$refs.viewerContainer.scrollTo(this.scrollStatus);
      this.loadPages(pageNumber);
    },
    handleClickDownload() {
      const selectPages = this.pageSelectStatus
        .filter((page) => page.select)
        .map((page) => page.pageNumber);
      if (!ArrayUtils.isEmpty(selectPages)) {
        this.$emit("download", selectPages);
      } else {
        this.$emit("download", []);
      }
      this.selectAllValue = false;
    },
    handleClickPageThumbnail(pageNumber) {
      if (this.currentPage !== pageNumber) {
        this.currentPage = pageNumber;
        const index = this.pageNumberMap.get(pageNumber);
        this.scrollStatus.top = this.pageElementHeight * index;
        this.scrollLock = true;
        this.$refs.viewerContainer.scrollTo(this.scrollStatus);
        this.loadPages(pageNumber);
      }
    },
    handleScroll() {
      if (this.scrollLock) {
        this.scrollLock = false;
        return;
      }

      clearTimeout(this.scrollTimer);
      this.scrollTimer = setTimeout(() => {
        const viewer = this.$refs.viewerContainer;
        const scrollStatus = {
          top: viewer.scrollTop,
          left: viewer.scrollLeft,
        };

        if (
          scrollStatus.top === this.scrollStatus.top &&
          scrollStatus.left === this.scrollStatus.left
        ) {
          const index =
            this.scrollStatus.top > this.pageElementHeight
              ? Math.ceil(this.scrollStatus.top / this.pageElementHeight)
              : 1;
          if (index >= 0 && index <= this.pagesData.length) {
            const pageNumber = this.pagesData[index - 1][this.props.pageNumber];
            if (this.currentPage !== pageNumber) {
              this.currentPage = pageNumber;
              this.loadPages(pageNumber);
            }
          }
        }
      }, this.scrollTimeout);

      const viewer = this.$refs.viewerContainer;
      this.scrollStatus = {
        top: viewer.scrollTop,
        left: viewer.scrollLeft,
      };
    },
    handlePrevious() {
      const index = this.pageNumberMap.get(this.currentPage);
      if (index - 1 >= 0) {
        const pageNumber = this.pagesData[index - 1][this.props.pageNumber];
        if (this.currentPage !== pageNumber) {
          this.currentPage = pageNumber;
          this.scrollStatus.top -= this.pageElementHeight;
          this.scrollLock = true;
          this.$refs.viewerContainer.scrollTo(this.scrollStatus);
          this.loadPages(pageNumber);
        }
      }
    },
    handleNext() {
      const index = this.pageNumberMap.get(this.currentPage);
      if (index + 1 < this.pagesData.length) {
        const pageNumber = this.pagesData[index + 1][this.props.pageNumber];
        if (this.currentPage !== pageNumber) {
          this.currentPage = pageNumber;
          this.scrollStatus.top += this.pageElementHeight;
          this.scrollLock = true;
          this.$refs.viewerContainer.scrollTo(this.scrollStatus);
          this.loadPages(pageNumber);
        }
      }
    },
    handleZoomOut() {
      this.$refs.scaleSelect.value = "custom";
      this.scale = Math.round(this.scale * 10) / 10;
      if (this.scale > 0.1) {
        this.scale -= 0.1;
        this.changeScale();
      }
    },
    handleZoomIn() {
      this.$refs.scaleSelect.value = "custom";
      this.scale = Math.round(this.scale * 10) / 10;
      if (this.scale < 10.0) {
        this.scale += 0.1;
        this.changeScale();
      }
    },
    handlePageNumberChange() {
      if (this.currentPage > this.maxPageNumber) {
        this.currentPage = this.maxPageNumber;
      }
      this.jumpToPage(this.currentPage);
    },
    handleScaleChange() {
      const value = $("#scaleSelect").val();
      switch (value) {
        case "page-auto":
          this.scale = this.pageAutoScale;
          break;
        case "page-actual":
          this.scale = 1.0;
          break;
        case "page-fit":
          this.scale = this.pageFitScale;
          break;
        case "page-width":
          this.scale = this.pageWidthScale;
          break;
        default:
          this.scale = value;
          break;
      }
      this.changeScale();
    },
  },
};
</script>

<style scoped lang="less">
#sidebarContainer {
  visibility: visible !important;
  left: 0 !important;
  width: 250px !important;
  z-index: 1000 !important;

  #toolbarSidebar {
    display: flex;
    align-items: center;
    justify-content: space-between;

    #select-all-checkbox {
      height: auto;
      vertical-align: middle;
    }

    #download-icon {
      color: white;
    }
  }

  #thumbnailView {
    width: 100% !important;
    padding: 0 !important;
    background-color: rgba(42, 42, 46, 1) !important;

    #thumbnail-list {
      width: 200px;
      margin: auto;

      .list-item {
        //width: 100%;
        margin: 15px auto;
        border: 2px solid transparent !important;
        border-radius: 4px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
        background-color: white;
        padding: 10px;
        display: flex;
        flex-direction: column;
        justify-content: center;

        .thumbnail {
          margin: auto;
          background-color: white;
          //border: 2px solid #e4e7ed;
          height: 200px;
          width: 150px;
        }

        .item-bottom {
          width: 100%;
          margin-top: 10px;

          .checkbox {
            margin: 0;
            padding: 0;
            width: 20%;
          }

          .page-number {
            width: 80%;
            padding: 0;
            color: black;
            font-size: 18px;
            text-align: center;
            margin-left: 25%;
          }
        }
      }

      .list-item:hover {
        border-color: #4c84ff !important;
      }

      .page-active {
        background-color: #4c84ff !important;
      }
    }
  }
}

.page {
  box-sizing: content-box;
}

.title {
  color: white;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 20px;
  font-weight: 700;
}

.mask {
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  position: absolute;
  top: 0;
  left: 0;
  z-index: 500;
}

.mask-content {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 32px;
  font-weight: 700;
  letter-spacing: 20px;
}
</style>

```