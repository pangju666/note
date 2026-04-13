```javascript
<script setup>
import { ref } from "vue";
import { ObjectUtils, StringUtils } from "pangju-utils";
import { FileApi } from "@/apis/file/FileApi.js";
import FormModal from "@/components/common/FormModal.vue";
import { getValueType } from "@/utils/utils.js";
import isJSON from "is-valid-json";
import { useOptions } from "@/hooks/Options.js";
import { renderToolTipOption } from "@/utils/render.js";

const props = defineProps({
  show: {
    type: Boolean,
    default: false,
  },
  fileId: {
    type: Number,
    required: true,
  },
  onSuccess: {
    type: Function,
    required: true,
  },
});

defineEmits(["update:show"]);

const rules = {
  filename: [
    {
      required: true,
      message: "请输入文件名",
      trigger: "blur",
    },
    {
      pattern: /^[^\\<>:"/|?*.]+(\.[^\\<>:"/|?*]+)?$/,
      message: "文件名称格式不正确",
      trigger: "blur",
    },
  ],
  mimeType: [
    {
      required: true,
      message: "请输入文件类型",
      trigger: "blur",
    },
    {
      pattern: /^.+\/.+$/,
      message: "文件类型格式不正确",
      trigger: "blur",
    },
  ],
};
const model = ref({
  md5: null,
  filename: null,
  mimeType: null,
  metaData: [],
});

const onLoadForm = async () => {
  if (ObjectUtils.nonNull(props.fileId)) {
    const result = await FileApi.getInfo(props.fileId);
    model.value.md5 = result.md5;
    model.value.mimeType = result.mimeType;
    model.value.filename = result.extension
      ? result.name + "." + result.extension
      : result.name;
    for (let metaDataKey in result.metaData) {
      const type = getValueType(result.metaData[metaDataKey]);
      model.value.metaData.push({
        key: metaDataKey,
        value:
          type === "object"
            ? JSON.stringify(result.metaData[metaDataKey])
            : result.metaData[metaDataKey],
        type: type,
      });
    }
    await getMimeTypeOptions();
  }
};
const onSubmitForm = async () => {
  const metaData = {};
  for (let item of model.value.metaData) {
    if (StringUtils.isBlank(item.key)) {
      continue;
    }
    let value = item.value;
    if (item.type === "object") {
      if (!isJSON(item.value)) {
        continue;
      }
      value = JSON.parse(item.value);
    }
    metaData[item.key] = value;
  }
  await FileApi.updateInfo(props.fileId, {
    filename: model.value.filename,
    mimeType: model.value.mimeType,
    metaData: metaData,
  });
  props.onSuccess();
};
const onClearForm = () => {
  model.value = {
    md5: null,
    filename: null,
    mimeType: null,
    metaData: [],
  };
};

const {
  loading: mimeTypesLoading,
  options: mimeTypeOptions,
  getOptions: getMimeTypeOptions,
} = useOptions(FileApi.getMimeTypes);

const metaDataTypeOptions = [
  {
    label: "字符串",
    value: "string",
  },
  {
    label: "数字",
    value: "number",
  },
  {
    label: "布尔",
    value: "boolean",
  },
  {
    label: "JSON",
    value: "object",
  },
];

const getRailStyle = ({ checked }) => {
  const style = {};
  if (checked) {
    style.background = "#2080f0";
  } else {
    style.background = "#d03050";
  }
  return style;
};
const onMetaDataItemCreate = () => {
  return {
    key: null,
    value: null,
    type: "string",
  };
};
</script>

<template>
  <form-modal
    :show="show"
    :model="model"
    title="文件信息"
    :on-clear-form="onClearForm"
    :on-load-form="onLoadForm"
    :on-submit-form="onSubmitForm"
    :rules="rules"
    @update:show="$emit('update:show', $event)"
  >
    <n-form-item label="文件MD5" path="md5">
      <n-input :value="model.md5" readonly />
    </n-form-item>
    <n-form-item first label="文件名称" path="filename">
      <n-input
        v-model:value.trim="model.filename"
        clearable
        maxlength="250"
        placeholder="请输入文件名称"
      />
    </n-form-item>
    <n-form-item first label="文件类型" path="mimeType">
      <n-select
        v-model:value="model.mimeType"
        :loading="mimeTypesLoading"
        :options="mimeTypeOptions"
        :render-option="renderToolTipOption"
        clearable
        filterable
        placeholder="请选择文件类型"
        tag
      />
    </n-form-item>
    <n-form-item first label="元数据" path="metaData">
      <n-dynamic-input
        v-model:value="model.metaData"
        :on-create="onMetaDataItemCreate"
        preset="pair"
      >
        <template #default="{ value, index }">
          <div class="meta-data-input w-100">
            <n-select
              v-model:value="model.metaData[index].type"
              :options="metaDataTypeOptions"
              class="type-select"
            />
            <n-input
              v-model:value.trim="value.key"
              :maxlength="50"
              class="key-input"
              clearable
              key-placeholder="请输入属性名"
            />
            <n-input
              v-show="model.metaData[index].type === 'string'"
              v-model:value="value.value"
              clearable
              placeholder="请输入属性值"
            />
            <n-input-number
              v-show="model.metaData[index].type === 'number'"
              v-model:value="value.value"
              :show-button="false"
              clearable
              placeholder="请输入属性值"
            />
            <n-input
              v-show="model.metaData[index].type === 'object'"
              v-model:value.trim="value.value"
              :autosize="{
                minRows: 4,
                maxRows: 8,
              }"
              clearable
              placeholder="请输入JSON字符串"
              type="textarea"
            />
            <n-switch
              v-show="model.metaData[index].type === 'boolean'"
              v-model:value="value.value"
              :rail-style="getRailStyle"
            >
              <template #checked> true</template>
              <template #unchecked> false</template>
            </n-switch>
          </div>
        </template>
      </n-dynamic-input>
    </n-form-item>
  </form-modal>
</template>

<style lang="less" scoped>
.meta-data-input {
  display: flex;
  align-items: center;

  .type-select {
    margin-right: 12px;
    max-width: 100px;
  }

  .key-input {
    margin-right: 12px;
    max-width: 210px;
  }
}
</style>

```