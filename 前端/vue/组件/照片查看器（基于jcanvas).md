```
"jcanvas": "^21.0.1",
"jquery": "^3.6.0",
```

```
<template>
  <div class="photo-panel">
    <div class="photo-container">
      <canvas
        v-loading="loading"
        id="canvas"
        :width="canvasWidth"
        :height="canvasHeight"
        @mouseup="onMarkLayerMouseUp"
      ></canvas>
      <div v-show="enableTurnPages" class="turn-page is-left">
        <div title="上一页" class="cursor-pointer" @click="onClickLast">
          <svg-icon icon="gtzicon-zhixiangzuo" color="#fff" :size="42" />
        </div>
      </div>
      <div v-show="enableTurnPages" class="turn-page is-right">
        <div title="下一页" class="cursor-pointer" @click="onClickNext">
          <svg-icon icon="gtzicon-zhixiangzuo" color="#fff" :size="42" />
        </div>
      </div>
      <div v-if="toolbar" v-show="photoId" class="toolbar">
        <div>
          <div title="下载" class="cursor-pointer" @click="onClickDownload">
            <svg-icon icon="gtzicon-xiazai1" :size="32" />
          </div>
          <div title="缩放" class="cursor-pointer" @click="onClickZoomOut">
            <svg-icon icon="gtzicon-suoxiao" :size="32" />
          </div>
          <div title="放大" class="cursor-pointer" @click="onClickZoomIn">
            <svg-icon icon="gtzicon-sousuofangda" :size="32" />
          </div>
          <div title="还原比例" class="cursor-pointer" @click="onClickRefresh">
            <svg-icon icon="gtzicon-suoxiao1" :size="26" />
          </div>
          <div
            v-if="enableMark"
            class="cursor-pointer"
            title="显示(隐藏)标记"
            @click="onClickShowMarks"
          >
            <svg-icon
              v-show="!markVisible"
              icon="gtzicon-kaiguan-guanbi"
              :size="32"
            />
            <svg-icon
              v-show="markVisible"
              icon="gtzicon-kaiguan-dakai"
              :size="32"
            />
          </div>
          <div
            v-if="admin"
            class="cursor-pointer"
            title="编辑"
            @click="saveDialogVisible = true"
          >
            <svg-icon icon="gtzicon-bianji1" :size="32" />
          </div>
          <div
            v-if="admin"
            class="cursor-pointer"
            title="收藏"
            @click="onFavoriteClick"
          >
            <svg-icon
              v-show="!favoriteUsers.includes(username)"
              icon="gtzicon-shoucang1"
              :size="32"
            />
            <svg-icon
              v-show="favoriteUsers.includes(username)"
              icon="gtzicon-shoucangxiao"
              :size="32"
            />
          </div>
        </div>
      </div>
    </div>
    <photo-description
      class="photo-description"
      :photo="photo"
      :loading="descriptionLoading"
      @click-figure="onClickName($event)"
    >
      <template #operator>
        <slot name="operator" />
      </template>
    </photo-description>
    <el-dialog
      v-if="admin"
      v-model="markDialogVisible"
      :modal="false"
      title="标记人物"
      width="400px"
    >
      <div class="display-flex">
        <figure-auto-complete v-model="selectMarkName" class="mr-10" />
        <contributor-select
          v-model="selectContributorNames"
          ref="contributorSelect"
        />
      </div>
      <template #footer>
        <div>
          <el-button @click="markDialogVisible = false">取消</el-button>
          <el-button type="primary" @click="saveMark">确认</el-button>
        </div>
      </template>
    </el-dialog>
    <photo-edit-dialog
      v-if="admin"
      v-model="saveDialogVisible"
      :photo-id="photoId"
      @confirm="onDialogConfirm"
    />
  </div>
</template>

<script>
import $ from "jquery";
import jCanvas from "jcanvas";

import { MarkApi } from "@/api/MarkApi";
import { ArrayUtils, ObjectUtils, StringUtils } from "pangju-utils";
import { ElMessageBox } from "element-plus";
import PhotoEditDialog from "@/components/custom/common/dialog/PhotoEditDialog";
import ContributorSelect from "@/components/custom/common/form/select/ContributorSelect";
import { PhotoApi } from "@/api/PhotoApi";
import { mapActions } from "vuex";
import FigureAutoComplete from "@/components/custom/common/form/select/FigureAutoComplete";
import { TokenUtils } from "@/assets/js/TokenUtils";
import SvgIcon from "@/components/common/SvgIcon";
import PhotoDescription from "@/components/custom/common/data/PhotoDescription";
import { getSignatureUrl } from "@/assets/js/ApiSignature";

jCanvas($, window);

export default {
  name: "PhotoViewer",
  components: {
    PhotoDescription,
    FigureAutoComplete,
    ContributorSelect,
    PhotoEditDialog,
    SvgIcon,
  },
  props: {
    startPhotoId: {
      type: Number,
    },
    photoIds: {
      type: Array,
      required: true,
    },
    faces: {
      type: Array,
      default: () => [],
    },
    sidebar: {
      type: Boolean,
      default: false,
    },
    toolbar: {
      type: Boolean,
      default: false,
    },
    enableMark: {
      type: Boolean,
      default: false,
    },
    defaultMarkVisible: {
      type: Boolean,
      default: false,
    },
    canvasWidth: {
      type: Number,
      default: 1200,
    },
    canvasHeight: {
      type: Number,
      default: 800,
    },
  },
  emits: ["dialog-confirm"],
  data() {
    return {
      loading: false,
      descriptionLoading: true,
      selectMarkName: "",
      selectContributorNames: [],
      currentPhotoIndex: 0,
      photoMarks: [],
      markDialogVisible: false,
      saveDialogVisible: false,
      markVisible: this.defaultMarkVisible && this.enableMark,
      drawEnable: false,
      drawData: {},
      scale: 1.0,
      existMarkLayer: false,
      image: null,
      photo: {
        referenceMaterials: [],
        marks: [],
        labels: [],
      },
      drawerVisible: true,
      favoriteUsers: [],
      username: TokenUtils.getUsername(),
      favoriteLoading: false,
      showMarkRect: false,
    };
  },
  computed: {
    photoIdMap() {
      const photoIdMap = new Map();
      this.photoIds.forEach((photoId, index) => {
        photoIdMap.set(photoId, index);
      });
      return photoIdMap;
    },
    enableTurnPages() {
      return !ArrayUtils.isEmpty(this.photoMarks) && this.photoMarks.length > 1;
    },
    fileMd5() {
      if (ArrayUtils.isEmpty(this.photoMarks)) {
        return "";
      }
      return this.photoMarks[this.currentPhotoIndex].fileMd5;
    },
    photoId() {
      if (ArrayUtils.isEmpty(this.photoMarks)) {
        return null;
      }
      const photoId = this.photoMarks[this.currentPhotoIndex].photoId;
      this.getFavoriteUsers(photoId);
      return photoId;
    },
    face() {
      if (ArrayUtils.isEmpty(this.faces)) {
        return null;
      }
      return this.faces.find((avatar) => avatar.photoId === this.photoId);
    },
    marks() {
      if (ArrayUtils.isEmpty(this.photoMarks)) {
        return [];
      }
      return this.photoMarks[this.currentPhotoIndex].marks;
    },
    serialNumber() {
      if (ArrayUtils.isEmpty(this.photoMarks)) {
        return "";
      }
      const serialNumber = this.photoMarks[this.currentPhotoIndex].serialNumber;
      let zeroFill = "";
      for (let i = 0; i < 4 - serialNumber.toString().length; i++) {
        zeroFill += "0";
      }
      return zeroFill + serialNumber;
    },
    admin() {
      return TokenUtils.isAdmin();
    },
  },
  watch: {
    async photoIds() {
      this.loading = true;
      let photoMarks = await MarkApi.listMarksByPhotoIds(this.photoIds);
      photoMarks.sort(
        (left, right) =>
          this.photoIdMap.get(left.photoId) - this.photoIdMap.get(right.photoId)
      );
      this.photoMarks = photoMarks;

      if (ObjectUtils.isNotNull(this.startPhotoId)) {
        const currentPhotoIndex = this.photoMarks.findIndex(
          (photoMark) => photoMark.photoId === this.startPhotoId
        );

        if (currentPhotoIndex === this.currentPhotoIndex) {
          this.reloadImage();
        } else {
          this.currentPhotoIndex = currentPhotoIndex;
        }
      } else {
        if (this.currentPhotoIndex === 0) {
          this.reloadImage();
        } else {
          this.currentPhotoIndex = 0;
        }
      }
    },
    currentPhotoIndex() {
      this.reloadImage();
    },
  },
  async created() {
    if (!ArrayUtils.isEmpty(this.photoIds)) {
      this.loading = true;
      let photoMarks = await MarkApi.listMarksByPhotoIds(this.photoIds);
      photoMarks.sort(
        (left, right) =>
          this.photoIdMap.get(left.photoId) - this.photoIdMap.get(right.photoId)
      );
      this.photoMarks = photoMarks;

      if (ObjectUtils.isNotNull(this.startPhotoId)) {
        this.currentPhotoIndex = this.photoMarks.findIndex(
          (photoMark) => (photoMark.photoId = this.startPhotoId)
        );
      }
    }

    this.$nextTick(() => {
      if (!ArrayUtils.isEmpty(this.photoMarks)) {
        this.renderImage();
      }
    });
  },
  methods: {
    ...mapActions("ResultDialog", ["success", "warning"]),
    async onDialogConfirm() {
      this.descriptionLoading = true;
      this.photo = await PhotoApi.getPhotoDetail(this.photoId);
      /*if (StringUtils.isNotEmpty(this.photo.startDate)) {
        this.photo.endDate = `${this.photo.startDate}至${this.photo.endDate}`;
      }*/
      this.descriptionLoading = false;
      this.$emit("dialog-confirm");
    },
    onClickName(name) {
      if (this.scale !== 1.0) {
        this.scale = 1.0;
        this.clearLayers();
        this.renderImage();
      }
      const photoMarks = this.photoMarks[this.currentPhotoIndex].marks;
      const mark = photoMarks.find((photoMark) => photoMark.name === name);
      this.renderMarkRect(mark);
    },
    onClickLast() {
      if (!ArrayUtils.isEmpty(this.photoIds) && this.currentPhotoIndex > 0) {
        --this.currentPhotoIndex;
      } else {
        this.warning("已经是第一张了");
      }
    },
    onClickNext() {
      if (
        !ArrayUtils.isEmpty(this.photoIds) &&
        this.currentPhotoIndex < this.photoIds.length - 1
      ) {
        ++this.currentPhotoIndex;
      } else {
        this.warning("已经是最后一张了");
      }
    },
    onClickDownload() {
      this.$store.dispatch("FileTransferPanel/downloadFile", {
        fileMd5: this.fileMd5,
        fileName: `${this.serialNumber}.tiff`,
      });
    },
    onClickZoomIn() {
      this.scale += 0.1;
      this.markVisible = false;
      this.clearLayers();
      this.renderImage();
    },
    onClickZoomOut() {
      this.scale -= 0.1;
      this.markVisible = false;
      this.clearLayers();
      this.renderImage();
    },
    onClickRefresh() {
      this.scale = 1.0;
      this.clearLayers();
      this.renderImage();
    },
    onMarkLayerMouseUp() {
      this.drawEnable = false;
    },
    onMarkLayerMouseDown(layer) {
      if (!this.admin) {
        return;
      }

      const userAgent = navigator.userAgent;
      if (
        /(?:iPad|PlayBook)/.test(userAgent) ||
        (/(?:Android)/.test(userAgent) && !/(?:Mobile)/.test(userAgent)) ||
        (/(?:Firefox)/.test(userAgent) && /(?:Tablet)/.test(userAgent))
      ) {
        return;
      }

      if (!this.drawEnable && this.markVisible) {
        this.drawEnable = true;
        this.drawData.posX = layer.eventX;
        this.drawData.posY = layer.eventY;

        if (this.existMarkLayer) {
          $("#canvas")
            .setLayer("mark-layer", {
              x: layer.eventX,
              y: layer.eventY,
            })
            .drawLayers();
        } else {
          $("#canvas")
            .removeLayer("mark-layer")
            .drawLayers()
            .drawArc({
              layer: true,
              name: "mark-layer",
              strokeStyle: "red",
              strokeWidth: 3,
              x: layer.eventX,
              y: layer.eventY,
              opacity: 0.6,
              radius: 10,
              index: 2,
              mousemove: this.onMarkLayerMouseMove,
              dblclick: () => (this.markDialogVisible = true),
            });
          this.existMarkLayer = true;
        }
      }
    },
    onMarkLayerMouseMove(layer) {
      if (this.drawEnable && this.markVisible) {
        this.drawData.radius = Math.sqrt(
          Math.pow(layer.eventX - this.drawData.posX, 2) +
            Math.pow(layer.eventY - this.drawData.posY, 2)
        );
        $("#canvas")
          .setLayer("mark-layer", { radius: this.drawData.radius })
          .drawLayers();
      }
    },
    onClickShowMarks() {
      this.markVisible = !this.markVisible;
      if (!this.markVisible) {
        this.hideMarkLayers();
        this.renderFace(this.face, this.image.width, this.image.height);
      } else {
        $("#canvas")
          .removeLayerGroup("mark-layers")
          .removeLayerGroup("text-layers")
          .removeLayer("face-layer")
          .drawLayers();
        if (this.scale !== 1.0) {
          this.scale = 1.0;
          $("#canvas").removeLayer("image-layer").drawLayers();
          this.renderImage();
        } else {
          this.marks.forEach((mark, index) => {
            this.renderMarkAndText(mark, index);
          });
        }
      }
    },
    reloadImage() {
      this.image = null;
      this.scale = 1.0;
      this.markVisible = this.defaultMarkVisible;
      this.drawEnable = false;
      this.drawData = {};
      this.existMarkLayer = false;
      this.descriptionLoading = true;
      this.photo = {};
      this.clearLayers();
      this.renderImage();
    },
    saveMark() {
      if (StringUtils.isNotEmpty(this.selectMarkName)) {
        const mark = {
          name: this.selectMarkName.trim(),
          posX: this.drawData.posX,
          posY: this.drawData.posY,
          radius: ObjectUtils.getSafeValue(this.drawData.radius, 10),
          contributors: this.selectContributorNames,
        };

        this.drawData = {};
        $("#canvas").removeLayer("mark-layer").drawLayers();
        this.renderMarkAndText(mark, this.marks.length);

        MarkApi.create(this.photoId, mark).then((res) => {
          if (res !== -1) {
            mark.id = res;
            this.selectMarkName = "";
            this.selectContributorNames = [];
            this.dialogVisible = false;
            this.existMarkLayer = false;
            this.marks.push(mark);
            this.photo.marks.push(mark.name);
          } else {
            this.warning("标记添加失败");
            $("#canvas")
              .removeLayer(`mark-${this.marks.length - 1}`)
              .removeLayer(`text-${this.marks.length - 1}`)
              .drawLayers();
          }
        });
        this.markDialogVisible = false;
      }
    },
    removeMark(index) {
      ElMessageBox.confirm("是否确认删除此标记", "警告", {
        confirmButtonText: "确认",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(async () => {
          $("#canvas")
            .removeLayer(`mark-${index + 1}`)
            .removeLayer(`text-${index + 1}`)
            .drawLayers();
          if (await MarkApi.remove(this.marks[index].id)) {
            const markName = this.marks[index].name;
            const photoMarkIndex = this.photo.marks.findIndex(
              (photoMark) => photoMark === markName
            );
            this.photo.marks.splice(photoMarkIndex, 1);
            this.marks.splice(index, 1);
          } else {
            this.warning("删除失败");
            const index = this.marks.length - 1;
            this.renderMarkAndText(this.marks[index], index);
          }
        })
        .catch(() => {});
    },
    loadImage(callback) {
      if (ObjectUtils.isNotNull(this.image)) {
        callback();
        return;
      }
      this.loading = true;
      this.image = new Image();
      this.image.addEventListener(
        "load",
        (event) => {
          if (ObjectUtils.isNull(event.path)) {
            this.image.width = event.target.width;
            this.image.height = event.target.height;
          } else {
            this.image.width = event.path[0].width;
            this.image.height = event.path[0].height;
          }
          callback();

          PhotoApi.getPhotoDetail(this.photoId).then((res) => {
            this.photo = res;
            /*if (StringUtils.isNotEmpty(this.photo.startDate)) {
              this.photo.endDate = `${this.photo.startDate}至${this.photo.endDate}`;
            }*/
            this.descriptionLoading = false;
          });

          if (this.enableMark) {
            this.marks.forEach((mark, index) => {
              this.renderMarkAndText(mark, index);
            });
          }
          if (
            !this.markVisible &&
            ObjectUtils.isNotNull(this.face) &&
            this.scale === 1.0
          ) {
            this.renderFace(this.face, this.image.width, this.image.height);
          }
        },
        false
      );
      this.image.src = getSignatureUrl(
        `${process.env.VUE_APP_BASE_FILE_URL}/image/preview/${this.fileMd5}?height=800&width=1200&placeholder=true`
      );
    },
    renderImage() {
      this.loadImage(() => {
        $("#canvas").drawImage({
          layer: true,
          name: "image-layer",
          source: this.image,
          x: this.canvasWidth / 2,
          y: this.canvasHeight / 2,
          width: this.image.width * this.scale,
          height: this.image.height * this.scale,
          draggable: this.scale !== 1.0,
          fromCenter: true,
          index: 0,
          load: () => {
            this.loading = false;
          },
          mousedown: this.onMarkLayerMouseDown,
          mousemove: this.onMarkLayerMouseMove,
        });
      });
    },
    renderMarkAndText(mark, index) {
      $("#canvas")
        .drawArc({
          layer: true,
          groups: [`mark-layers`],
          name: `mark-${index + 1}`,
          strokeStyle: "red",
          strokeWidth: 3,
          x: mark.posX,
          y: mark.posY,
          opacity: this.markVisible ? 0.6 : 0,
          radius: mark.radius,
          index: 2,
          dblclick: () => this.removeMark(index),
          mouseover: () => {
            if (!this.markVisible) {
              $("#canvas")
                .setLayer(`mark-${index + 1}`, {
                  opacity: 1,
                })
                .setLayer(`text-${index + 1}`, {
                  opacity: 1,
                })
                .drawLayers();
            }
          },
          mouseout: () => {
            if (!this.markVisible) {
              $("#canvas")
                .setLayer(`mark-${index + 1}`, {
                  opacity: 0,
                })
                .setLayer(`text-${index + 1}`, {
                  opacity: 0,
                })
                .drawLayers();
            }
          },
        })
        .drawText({
          layer: true,
          groups: [`text-layers`],
          name: `text-${index + 1}`,
          index: 3,
          fillStyle: "red",
          /*   strokeStyle: "black",
          strokeWidth: 0.5,*/
          align: "center",
          opacity: this.markVisible ? 0.6 : 0,
          x: mark.posX,
          y: mark.posY,
          fontSize: "9pt",
          text: mark.name,
        });
    },
    renderFace(face, imageWidth, imageHeight) {
      if (ObjectUtils.isNotNull(face)) {
        $("#canvas").drawRect({
          layer: true,
          name: "face-layer",
          strokeStyle: "#409EFF",
          strokeWidth: 2,
          x: face.posX + (this.canvasWidth - imageWidth) / 2,
          y: face.posY + (this.canvasHeight - imageHeight) / 2,
          width: face.width,
          height: face.height,
          fromCenter: false,
          index: 1,
        });
      }
    },
    renderMarkRect(mark) {
      $("#canvas")
        .removeLayer("photo-mark-layer")
        .drawLayers()
        .drawRect({
          layer: true,
          name: "photo-mark-layer",
          strokeStyle: "#1466C7",
          strokeWidth: 3,
          x: mark.posX,
          y: mark.posY,
          width: mark.radius * 2,
          height: mark.radius * 2,
          index: 2,
        });
    },
    hideMarkLayers() {
      $("#canvas")
        .setLayerGroup("mark-layers", {
          opacity: 0,
        })
        .setLayerGroup("text-layers", {
          opacity: 0,
        })
        .removeLayer("mark-layer")
        .drawLayers();
      this.existMarkLayer = false;
    },
    clearLayers() {
      $("#canvas").removeLayers().drawLayers();
      this.existMarkLayer = false;
    },
    async getFavoriteUsers(photoId) {
      this.favoriteLoading = true;
      this.favoriteUsers = await PhotoApi.favoriteUsers(photoId);
      this.favoriteLoading = false;
    },
    async onFavoriteClick() {
      this.favoriteLoading = true;
      let favoriteUsers = [...this.favoriteUsers];
      if (this.favoriteUsers.includes(this.username)) {
        const index = this.favoriteUsers.findIndex(
          (user) => user === this.username
        );
        favoriteUsers.splice(index, 1);
      } else {
        favoriteUsers.push(this.username);
      }
      if (await PhotoApi.updateFavoriteUsers(this.photoId, favoriteUsers)) {
        this.favoriteUsers = favoriteUsers;
      } else {
        this.warning("收藏失败");
      }
      this.favoriteLoading = false;
    },
  },
};
</script>

<style lang="less" scoped>
.photo-panel {
  position: relative;
  display: flex;
  justify-content: center;

  .photo-container {
    width: 1200px;
    height: 800px;
    position: relative;
    background: #f6f6f6;

    .toolbar {
      position: absolute;
      bottom: 10px;
      left: 40%;

      > div {
        display: flex;
        justify-content: space-between;
        height: 42px;
        box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.28);
        border-radius: 4px;
        background: #ffffff;

        > div {
          width: 42px;
          height: 42px;
          display: flex;
          justify-content: center;
          align-items: center;
          color: #595959;

          &:hover {
            background: #0091ff;
            border-radius: 4px;
            color: #ffffff;
          }
        }
      }
    }

    .turn-page {
      height: 100%;
      position: absolute;
      top: 0;
      display: flex;
      align-items: center;

      > div {
        width: 64px;
        height: 60px;
        background: #000000;
        opacity: 0.3;
        display: flex;
        justify-content: center;
        align-items: center;
      }
    }

    .turn-page.is-left {
      left: 0;
    }

    .turn-page.is-right {
      right: 0;
      transform: rotateY(180deg);
    }
  }

  .photo-description {
    width: 300px;
    padding-left: 20px;
  }
}
</style>

```