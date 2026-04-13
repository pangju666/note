```
<script setup>
import { onMounted, ref, watch } from "vue";
import { FileApi } from "@/api/FileApi.js";
import SvgIcon from "@/components/commons/SvgIcon.vue";
import {
  LOCATION_TYPE,
  POLYGON_TYPE,
  POLYLINE_TYPE,
  SPHERE_LOCATION_TYPE,
} from "@/utils/constant.js";
import { DateUtils } from "pangju-utils";
import { SearchOutline } from "@vicons/ionicons5";
import { useRouter } from "vue-router";
import { GeoApi } from "@/api/GeoApi.js";

const router = useRouter();
defineProps({
  show: {
    type: Boolean,
    required: true,
  },
});

defineEmits(["update:show", "click-navigation", "click-item"]);

const types = [
  { value: LOCATION_TYPE, label: "地点" },
  { value: SPHERE_LOCATION_TYPE, label: "全景地点" },
  { value: POLYLINE_TYPE, label: "线段" },
  { value: POLYGON_TYPE, label: "区域" },
];

const loading = ref(false);
const page = ref(0);
const pageSize = ref(10);
const geoData = ref([]);
const selectType = ref(LOCATION_TYPE);
const keyword = ref("");
const noMore = ref(false);

onMounted(() => {
  loadData();
});

watch(selectType, () => {
  noMore.value = false;
  keyword.value = null;
  page.value = 0;
  geoData.value = [];
  loadData();
});

const loadData = async () => {
  if (noMore.value || loading.value) {
    return;
  }

  loading.value = true;
  ++page.value;
  const result = await GeoApi.page(
    page.value,
    pageSize.value,
    selectType.value,
    keyword.value,
  );
  geoData.value.push(...result.records);
  noMore.value = page.value === result.pages;
  loading.value = false;
};

const onClickSearchIcon = () => {
  noMore.value = false;
  page.value = 0;
  geoData.value = [];
  loadData();
};
const onClickSphereIcon = (item) => {
  const url = router.resolve(`/sphere/${item.id}`).href;
  window.open(url, "_blank");
};

defineExpose({
  loadData,
});
</script>

<template>
  <n-drawer
    :show="show"
    :width="500"
    style="height: 100%"
    placement="right"
    @update:show="$emit('update:show', false)"
  >
    <n-drawer-content title="Geo列表" closable native-scrollbar>
      <div class="full-size">
        <n-radio-group v-model:value="selectType" class="mb-10">
          <n-radio v-for="type in types" :key="type.value" :value="type.value">
            {{ type.label }}
          </n-radio>
        </n-radio-group>
        <div style="height: 35px" class="mb-10">
          <n-input
            v-model:value.trim="keyword"
            maxlength="30"
            type="text"
            clearable
            round
            placeholder="请输入关键字搜索"
            @keydown.enter="onClickSearchIcon"
          >
            <template #suffix>
              <n-icon
                class="cursor-pointer"
                :size="30"
                @click="onClickSearchIcon"
              >
                <search-outline />
              </n-icon>
            </template>
          </n-input>
        </div>
        <n-infinite-scroll
          :distance="10"
          style="height: calc(100% - 34px - 10px - 35px - 10px)"
          @load="loadData"
        >
          <div
            v-for="item in geoData"
            :key="item.id"
            class="geo-card w-100 mb-10"
          >
            <n-image
              :preview-src="FileApi.getFileDownloadUrl(item.coverImageMd5)"
              :src="FileApi.getImagePreviewUrl(item.coverImageMd5)"
              class="mr-10"
              lazy
              width="100px"
            >
              <template #placeholder>
                <img
                  src="
                  width="100"
                />
              </template>
            </n-image>
            <n-thing style="width: calc(100% - 130px)">
              <template #header>
                <n-ellipsis
                  :style="{
                    maxWidth:
                      item.type === SPHERE_LOCATION_TYPE ? `260px` : '290px',
                  }"
                >
                  <span
                    class="cursor-pointer"
                    @click="$emit('click-item', item)"
                  >
                    {{ item.name }}
                  </span>
                </n-ellipsis>
              </template>
              <template #header-extra>
                <svg-icon
                  v-if="item.type === POLYLINE_TYPE"
                  :size="25"
                  color="#1A73E8"
                  icon="gtzicon-relation-copy"
                />
                <svg-icon
                  v-else-if="item.type === POLYGON_TYPE"
                  :size="25"
                  color="#1A73E8"
                  icon="gtzicon-mianji-copy"
                />
                <svg-icon
                  v-else
                  :size="25"
                  color="#1A73E8"
                  icon="gtzicon-biaoji2-copy"
                />
              </template>
              <template #description>
                <n-ellipsis
                  :style="{
                    maxWidth:
                      item.type === SPHERE_LOCATION_TYPE ? `260px` : '290px',
                  }"
                >
                  {{ item.address ?? "暂无" }}
                </n-ellipsis>
              </template>
              <template #default>
                <n-ellipsis
                  :line-clamp="3"
                  :style="{
                    maxWidth:
                      item.type === SPHERE_LOCATION_TYPE ? `260px` : '290px',
                  }"
                >
                  {{ item.remark ?? "暂无" }}
                </n-ellipsis>
              </template>
              <template #footer>
                {{ DateUtils.dateFns().format(item.createTime, "yyyy-MM-dd") }}
              </template>
              <template #action>
                <div class="display-flex" style="align-items: center">
                  <n-button
                    class="mr-5"
                    round
                    size="small"
                    @click="$emit('click-navigation', item)"
                  >
                    <template #icon>
                      <svg-icon
                        :size="25"
                        color="#000"
                        icon="gtzicon-daohang"
                      />
                    </template>
                    地图跳转
                  </n-button>
                  <n-button
                    v-if="item.type === SPHERE_LOCATION_TYPE"
                    round
                    size="small"
                    @click="onClickSphereIcon(item)"
                  >
                    <template #icon>
                      <svg-icon
                        :size="30"
                        color="#000"
                        icon="gtzicon-quanjingtu-copy"
                      />
                    </template>
                    全景浏览
                  </n-button>
                </div>
              </template>
            </n-thing>
          </div>
          <div v-if="loading" class="text">加载中...</div>
          <div v-if="noMore" class="text">没有更多了 🤪</div>
        </n-infinite-scroll>
      </div>
    </n-drawer-content>
  </n-drawer>
</template>

<style scoped lang="less">
.geo-card {
  display: flex;
  align-items: center;
}

.text {
  text-align: center;
}
</style>

```