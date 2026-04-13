```
"v-viewer": "^3.0.10",
```

```
<template>
  <div class="viewer" :style="style">
    <viewer :images="images" :options="options">
      <img
        v-for="(image, index) in images"
        :key="index"
        :src="image"
        alt=""
        class="display-none"
      />
    </viewer>
  </div>
</template>

<script>
export default {
  name: "ImageViewer",
  props: {
    images: {
      type: Array,
      default() {
        return [];
      },
    },
    height: {
      type: String,
      default: "800px",
    },
  },
  data() {
    return {
      viewer: null,
      options: {
        loading: true, // 加载动画
        container: "#viewer-container", // 显示容器元素
        inline: true, // 内联模式
        backdrop: true, // 显示背景遮罩
        button: false, // 隐藏右上角按钮
        navbar: true, // 显示导航栏
        title: false, // 隐藏标题,
        toolbar: {
          //工具栏配置(0隐藏 1 显示)
          play: 0, // 播放按钮
          zoomIn: 1, // 放大
          zoomOut: 1, // 缩小
          oneToOne: 0, // 切换缩放比例（当前和原始）
          reset: 0, // 重置图像
          prev: 1, // 上一张
          next: 1, // 下一张
          rotateLeft: 0, // 向左旋转
          rotateRight: 0, // 向右旋转
          flipHorizontal: 0, // 水平翻转
          flipVertical: 0, // 垂直翻转
        },
        fullscreen: false, // 禁止全屏
      },
    };
  },
  computed: {
    style() {
      return {
        height: this.height,
      };
    },
  },
};
</script>

<style lang="less" scoped>
.viewer {
  border-top: 1px solid #dcdfe6;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
}
</style>

```