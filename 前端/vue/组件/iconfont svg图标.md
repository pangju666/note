```javascript
<template>
  <svg :style="style" aria-hidden="true" class="svg-icon">
    <use :style="svgStyle" :xlink:href="svgName" />
  </svg>
</template>

<script setup>
import { ObjectUtils } from "pangju-utils";
import { computed } from "vue";

const props = defineProps({
  svgStyle: {
    type: String,
    default: "",
  },
  icon: {
    type: String,
    required: true,
  },
  size: {
    type: Number,
    default: undefined,
  },
});

const svgName = computed(() => `#${props.icon}`);
const style = computed(() =>
  ObjectUtils.nonNull(props.size) && props.size > 0
    ? `font-size: ${props.size}px`
    : "",
);
</script>

<style>
.svg-icon {
  overflow: hidden;
  width: 1em;
  height: 1em;
  vertical-align: -0.15em;
  fill: currentcolor;
}
</style>
```