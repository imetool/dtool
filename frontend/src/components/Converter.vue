<template>
  <n-form
    ref="formRef"
    :model="model"
    label-placement="left"
    :label-width="100"
    :style="{
      minWidth: '640px',
      maxWidth: '800px',
    }"
  >
    <div style="height: 2rem"></div>
    <n-card title="输入" hoverable content-style="padding-bottom: unset;">
      <n-form-item label=" " path="uploadValue">
        <n-upload
          v-model:action="uploadUrl"
          :max="1"
          v-model:file-list="dictList"
        >
          <n-upload-dragger>
            <div style="margin-bottom: 10px">
              <n-icon size="48" :depth="3">
                <archive-icon />
              </n-icon>
            </div>
            <n-text style="font-size: 16px"> 点击或者拖动文件到该区域 </n-text>
          </n-upload-dragger>
        </n-upload>
      </n-form-item>
      <n-form-item label="源格式" path="inputFormat" style="margin-bottom: 0">
        <n-select
          v-model:value="model.inputFormat"
          placeholder="Select"
          :options="generalOptions"
        />
      </n-form-item>
    </n-card>
    <div style="height: 1rem"></div>

    <n-card title="输出" hoverable content-style="padding-bottom: unset;">
      <n-form-item label="输出方案" path="schemaGroupValue">
        <n-radio-group v-model:value="model.kind" name="schemaGroup">
          <n-radio value="pinyin"> 拼音 </n-radio>
          <n-radio value="wubi"> 五笔 </n-radio>
          <n-radio value="words"> 纯词组 </n-radio>
        </n-radio-group>
      </n-form-item>
      <n-form-item label="输出格式" path="outputFormat">
        <n-select
          v-model:value="model.outputFormat"
          placeholder="Select"
          :options="outputFormatOptions"
        />
      </n-form-item>

      <n-form-item
        label="编码方案"
        path="mbGroupValue"
        v-if="model.kind === 'wubi'"
      >
        <n-radio-group v-model:value="model.schema" name="mbgroup">
          <n-radio value="original"> 原始 </n-radio>
          <n-radio value="phrase"> 拼音短语 </n-radio>
          <n-radio value="86"> 86五笔 </n-radio>
          <n-radio value="98"> 98五笔 </n-radio>
          <n-radio value="06"> 新世纪 </n-radio>
          <n-radio value="custom"> 自定义 </n-radio>
        </n-radio-group>
      </n-form-item>

      <n-form-item label=" " path="uploadValue" v-if="displayCustom">
        <n-upload
          v-model:action="uploadMbUrl"
          :max="1"
          v-model:file-list="mbList"
        >
          <n-upload-dragger>
            <div style="margin-bottom: 10px">
              <n-icon size="48" :depth="3">
                <archive-icon />
              </n-icon>
            </div>
            <n-text style="font-size: 16px"> 点击或者拖动文件到该区域 </n-text>
          </n-upload-dragger>
        </n-upload>
      </n-form-item>

      <n-form-item
        label="三字词规则"
        path="ruleGroupValue"
        v-if="displayCustom"
      >
        <n-radio-group v-model:value="model.rule" name="rulegroup">
          <n-radio value="AABC"> AABC </n-radio>
          <n-radio value="ABCC"> ABCC </n-radio>
        </n-radio-group>
      </n-form-item>
    </n-card>
    <n-card :bordered="false">
      <div style="float: right">
        <n-button
          class="button"
          type="primary"
          @click="handleClick"
          v-model:disabled="disable"
          :loading="loading"
        >
          转换并保存
        </n-button>
      </div>
    </n-card>
  </n-form>
</template>

<script setup lang="ts">
import { defineComponent, computed, ref, watch } from "vue";
import { ArchiveOutline as ArchiveIcon } from "@vicons/ionicons5";
import axios from "axios";
import type { UploadFileInfo } from "naive-ui";
import { useDialog } from "naive-ui";

// 判断是否是开发环境
const host =
  import.meta.env.MODE === "development" ? "http://localhost:7800" : "";
let uploadUrl = host + "/upload";
let uploadMbUrl = uploadUrl + "/mb";

// 表单
const formRef = ref(null);
const model = ref({
  name: "",
  inputFormat: "sogou_bak",
  kind: "pinyin",
  schema: "original",
  rule: "AABC",
  outputFormat: "sogou",
});

// 获取格式列表
type Data = { name: any; id: any; canMarshal: any; kind: any }[];
let data: Data;
data = await fetch(host + "/list").then((res) => res.json());

function getFirstID(id: string) {
  return id.split(",")[0];
}

const generalOptions = data.map((item) => ({
  label: item.name,
  value: getFirstID(item.id),
}));
console.log(generalOptions);

const pinyinFormatOptions = data
  .filter((item) => item.kind === 1 && item.canMarshal)
  .map((item) => ({ label: item.name, value: getFirstID(item.id) }));

const wubiFormatOptions = data
  .filter((item) => item.kind === 2 && item.canMarshal)
  .map((item) => ({ label: item.name, value: getFirstID(item.id) }));

const outputFormatOptions = computed(() => {
  switch (model.value.kind) {
    case "pinyin":
      return pinyinFormatOptions;
    case "wubi":
      return wubiFormatOptions;
    default:
      return [{ label: "纯词组", value: "words" }];
  }
});

// 词库文件
const dictList = ref<UploadFileInfo[]>([]);
// 是否包含已经完成的文件
const dictHasFinished = computed(() => {
  return dictList.value.length === 1 && dictList.value[0].status === "finished";
});
// 监听文件列表，赋值表单 name
watch(
  () => dictList.value,
  () => {
    if (dictList.value.length === 1) {
      const fi = dictList.value[0];
      if (fi.status === "finished") {
        model.value.name = fi.name;
        // 根据后缀猜测格式
        const dotIndex = fi.name.lastIndexOf(".");
        if (dotIndex !== -1) {
          const suffix = fi.name.substring(dotIndex + 1);
          const foundItem = data.find((item) =>
            item.id.split(",").includes(suffix)
          );
          if (foundItem) {
            const ids = foundItem.id.split(",");
            model.value.inputFormat = ids[0];
          }
        }
      }
    } else {
      model.value.name = "";
    }
  }
);

// 方案类型
watch(
  () => model.value.kind,
  () => {
    if (model.value.kind === "pinyin") {
      model.value.outputFormat = "sogou";
    } else if (model.value.kind === "wubi") {
      model.value.outputFormat = "baidu_def";
    } else {
      model.value.outputFormat = "words";
    }
  }
);
// 自定义码表
const displayCustom = computed(() => {
  return model.value.kind === "wubi" && model.value.schema === "custom";
});
const mbList = ref<UploadFileInfo[]>([]);
const mbHasFinished = computed(() => {
  return mbList.value.length === 1 && mbList.value[0].status === "finished";
});

const disableButton = ref(false);
const disable = computed(() => {
  return (
    disableButton.value || // loading
    !dictHasFinished.value || // 没有选择词库文件
    (displayCustom.value && !mbHasFinished.value) // 没有选择自定义码表
  );
});

// 转换
const loading = ref(false);
const dialog = useDialog();
function handleClick() {
  console.log(model.value);
  loading.value = true;
  disableButton.value = true;
  axios
    .post(host + "/api", JSON.stringify(model.value))
    .then((res) => {
      console.log(res);
      if (res.data !== "error") {
        dialog.success({
          title: "Success",
          content: "保存到：" + res.data,
          positiveText: "确定",
        });
      } else {
        convertError();
      }
    })
    // .catch(() => {
    //   convertError();
    // })
    .finally(() => {
      loading.value = false;
      disableButton.value = false;
    });
}

function convertError() {
  dialog.error({
    title: "Error",
    content: "导出失败!",
    positiveText: "确定",
  });
}

defineComponent({
  components: {
    ArchiveIcon,
  },
});
</script>
