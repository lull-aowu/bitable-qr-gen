<script>
import { bitable } from "@lark-base-open/js-sdk";
import { ref, onMounted, computed, watch } from "vue";
import templates from "../data/templates.json";
import {
  ElButton,
  ElForm,
  ElFormItem,
  ElSelect,
  ElOption,
  ElRadio,
  ElRadioGroup,
  ElSwitch,
  ElMessage,
  ElProgress,
} from "element-plus";

export default {
  components: {
    ElButton,
    ElForm,
    ElFormItem,
    ElSelect,
    ElOption,
    ElRadio,
    ElRadioGroup,
    ElSwitch,
    ElProgress,
  },
  setup() {
    const formData = ref({
      qrCodeField: "",
      qrCodeFunction: "returnField",
      returnField: "",
      templateId: "",
      generateForSelectedRows: true,
    });

    // 存储字段映射关系
    const fieldMappings = ref({});

    // 进度显示相关
    const loading = ref(false);
    const progress = ref(0);
    const totalRecords = ref(0);

    // 获取当前选中的模板
    const selectedTemplate = computed(() => {
      return templates.find((t) => t.id === formData.value.templateId) || null;
    });

    // 监听模板选择变化，清空字段映射
    watch(
      () => formData.value.templateId,
      () => {
        fieldMappings.value = {};
      },
    );

    // 获取模板对应的图片路径
    const templateImage = computed(() => {
      if (selectedTemplate.value) {
        // 在Vite中，使用ES模块动态导入图片
        const imageName = `code-${selectedTemplate.value.id}.png`;
        try {
          // 直接返回图片路径，Vite会自动处理
          return new URL(`../assets/images/${imageName}`, import.meta.url).href;
        } catch (error) {
          console.error(`Image not found: ${imageName}`);
          return null;
        }
      }
      return null;
    });
    const tableMetaList = ref([]);
    const fieldMetaList = ref([]);

    // 获取指定表的字段列表
    const getTableFields = async (tableId) => {
      if (tableId) {
        const table = await bitable.base.getTableById(tableId);
        return await table.getFieldMetaList();
      }
      return [];
    };

    onMounted(async () => {
      try {
        const [tableList, selection] = await Promise.all([
          bitable.base.getTableMetaList(),
          bitable.base.getSelection(),
        ]);

        tableMetaList.value = tableList;

        // 获取当前选中表的字段列表
        if (selection.tableId) {
          fieldMetaList.value = await getTableFields(selection.tableId);
        }
      } catch (error) {
        console.error("初始化失败:", error);
        ElMessage.error("初始化失败，请刷新重试");
      }
    });

    // 检查是否可以点击生成二维码按钮
    const isGenerateButtonDisabled = computed(() => {
      // 检查必填字段
      if (!formData.value.qrCodeField || !formData.value.templateId) {
        return true;
      }

      // 当选择返回指定字段时，检查返回字段是否为空
      if (
        formData.value.qrCodeFunction === "returnField" &&
        !formData.value.returnField
      ) {
        return true;
      }

      return false;
    });

    // 生成二维码按钮点击事件
    const generateQrCodes = async () => {
      try {
        let recordsToProcess = [];
        // 获取当前选中的表格和视图
        const selection = await bitable.base.getSelection();
        const tableId = selection.tableId;

        if (!tableId) {
          ElMessage.error("请先选择一个表格");
          return;
        }

        const viewId = selection.viewId;
        const table = await bitable.base.getTableById(tableId);

        if (!viewId) {
          ElMessage.error("无法获取当前视图ID");
          return;
        }

        // 通过视图ID获取视图实例
        const view = await table.getViewById(viewId);

        // 检查是否仅对选中行生成二维码
        if (formData.value.generateForSelectedRows) {
          // 获取选中的记录ID列表
          recordsToProcess = await view.getSelectedRecordIdList();

          if (recordsToProcess.length === 0) {
            ElMessage.error("您没有选中任何行，请先选中行再进行转换");
            return;
          }
        } else {
          // 获取所有记录
          recordsToProcess = await table.getRecordIdList();
        }

        // 初始化进度
        loading.value = true;
        progress.value = 0;
        totalRecords.value = recordsToProcess.length;

        const qrCodeFieldId = formData.value.qrCodeField;

        // 定义获取字段值的函数（移出循环外提高性能）
        const getFieldValue = (record, fieldId) => {
          if (!record || !record.fields) return "";

          const val = record.fields[fieldId];
          if (val == null) return "";
          if (typeof val === "object" && Array.isArray(val)) {
            return val
              .map((item) => {
                if (typeof item === "object") {
                  return (
                    item.text || item.name || item.id || JSON.stringify(item)
                  );
                }
                return String(item);
              })
              .join(",");
          }
          if (typeof val === "object") {
            return val.text || val.name || val.id || JSON.stringify(val);
          }
          return String(val);
        };

        // 批量生成二维码
        for (let i = 0; i < recordsToProcess.length; i++) {
          const recordId = recordsToProcess[i];

          try {
            // 获取记录数据
            const record = await table.getRecordById(recordId);

            const params = new URLSearchParams();
            params.append("cliT", formData.value.templateId);

            if (
              formData.value.qrCodeFunction === "returnField" &&
              formData.value.returnField
            ) {
              params.append(
                "cliD",
                getFieldValue(record, formData.value.returnField),
              );
            } else {
              const shareUrl = await table.getRecordShareLink(recordId);
              params.append("cliD", shareUrl);
            }

            for (const [key, fieldId] of Object.entries(fieldMappings.value)) {
              if (fieldId) {
                const fieldValue = getFieldValue(record, fieldId);
                let paramValue = fieldValue;

                // 如果key包含cliF，需要加上字段名
                if (key.includes("cliF")) {
                  const field = fieldMetaList.value.find(
                    (f) => f.id === fieldId,
                  );
                  if (field) {
                    paramValue = `${field.name}：${fieldValue}`;
                  }
                }

                params.append(key, paramValue);
              }
            }

            const cliImUrl = `https://open-api.cli.im/cli-open-platform-service/v1/labelStyle/create?${params.toString()}`;

            // 添加请求超时控制
            const qrResponse = await Promise.race([
              fetch(cliImUrl, { method: "get" }),
              new Promise((_, reject) =>
                setTimeout(() => reject(new Error("请求超时")), 30000),
              ),
            ]);

            if (!qrResponse.ok) {
              throw new Error(`草料二维码API请求失败: ${qrResponse.status}`);
            }

            let imageBlob;
            const contentType = qrResponse.headers.get("content-type") || "";

            if (contentType.includes("image")) {
              imageBlob = await qrResponse.blob();
            } else if (contentType.includes("json")) {
              const result = await qrResponse.json();
              const imageUrl =
                result.data?.imageUrl ||
                result.data?.url ||
                result.imageUrl ||
                result.url;

              if (!imageUrl) {
                throw new Error("草料二维码API未返回图片URL");
              }

              // 图片下载也添加超时控制
              const imgResponse = await Promise.race([
                fetch(imageUrl),
                new Promise((_, reject) =>
                  setTimeout(() => reject(new Error("图片下载超时")), 30000),
                ),
              ]);

              imageBlob = await imgResponse.blob();
            } else {
              imageBlob = await qrResponse.blob();
            }

            const file = new File([imageBlob], `qrcode_${recordId}.png`, {
              type: imageBlob.type || "image/png",
            });

            // 上传文件
            const tokens = await bitable.base.batchUploadFile([file]);
            await table.setRecord(recordId, {
              fields: {
                [qrCodeFieldId]: [
                  {
                    name: `qrcode_${recordId}.png`,
                    size: imageBlob.size,
                    type: imageBlob.type || "image/png",
                    token: tokens[0],
                    timeStamp: Date.now(),
                  },
                ],
              },
            });

            // 更新进度
            progress.value = Math.round(((i + 1) / totalRecords.value) * 100);
          } catch (error) {
            console.error(`处理记录 ${recordId} 失败:`, error);
            // 继续处理其他记录，不中断整个流程
            ElMessage.warning(
              `处理记录 ${i + 1}/${totalRecords.value} 失败: ${error.message}`,
            );

            // 仍然更新进度
            progress.value = Math.round(((i + 1) / totalRecords.value) * 100);
          }
        }

        ElMessage.success(
          `二维码生成完成！共处理 ${totalRecords.value} 条记录`,
        );
      } catch (error) {
        console.error("生成二维码失败:", error);
        ElMessage.error(`生成二维码失败: ${error.message}`);
      } finally {
        // 隐藏进度
        loading.value = false;
        // 重置进度值
        progress.value = 0;
        totalRecords.value = 0;
      }
    };

    return {
      formData,
      tableMetaList,
      fieldMetaList,
      templates,
      selectedTemplate,
      templateImage,
      fieldMappings,
      isGenerateButtonDisabled,
      generateQrCodes,
      loading,
      progress,
      totalRecords,
    };
  },
};
</script>

<template>
  <el-form ref="form" class="form" :model="formData" label-position="top">
    <el-form-item
      label="将生成的二维码存储在"
      size="mini"
      required
      class="field-wrap"
    >
      <el-select
        v-model="formData.qrCodeField"
        placeholder="请选择二维码储存字段"
        style="width: 100%"
      >
        <el-option
          v-for="field in fieldMetaList.filter((f) => f.type === 17)"
          :key="field.id"
          :label="field.name"
          :value="field.id"
        />
      </el-select>
    </el-form-item>

    <el-form-item label="" size="mini">
      <el-radio-group
        v-model="formData.qrCodeFunction"
        class="qr-function-radio"
      >
        <el-radio label="returnField">扫码后返回指定字段</el-radio>
        <el-radio label="jumpRecord">扫码后跳转当前记录</el-radio>
      </el-radio-group>
    </el-form-item>

    <el-form-item
      v-if="formData.qrCodeFunction === 'returnField'"
      label=""
      size="mini"
    >
      <el-select
        v-model="formData.returnField"
        placeholder="请选择扫码返回字段"
        style="width: 100%"
      >
        <el-option
          v-for="field in fieldMetaList.filter(
            (f) => f.type === 1 || f.type === 15,
          )"
          :key="field.id"
          :label="field.name"
          :value="field.id"
        />
      </el-select>
    </el-form-item>

    <el-form-item label="选择模板" size="mini" required class="field-wrap">
      <el-select
        v-model="formData.templateId"
        placeholder="请选择模板"
        style="width: 100%"
        filterable
      >
        <el-option
          v-for="template in templates"
          :key="template.id"
          :label="template.name"
          :value="template.id"
        />
      </el-select>
    </el-form-item>

    <!-- 模板预览和字段映射 -->
    <div v-if="selectedTemplate" class="template-preview">
      <!-- 左侧：模板图片 -->
      <div class="template-image">
        <img v-if="templateImage" :src="templateImage" alt="模板预览" />
        <div v-else class="no-image">暂无模板图片</div>
      </div>

      <!-- 右侧：动态字段映射 -->
      <div class="field-mappings">
        <el-form-item
          v-for="(child, index) in selectedTemplate.child"
          :key="child.vaue"
          :label="child.name"
          size="mini"
          class="field-wrap"
        >
          <el-select
            v-model="fieldMappings[child.vaue]"
            placeholder="请选择字段"
            style="width: 100%"
          >
            <el-option
              v-for="field in fieldMetaList"
              :key="field.id"
              :label="field.name"
              :value="field.id"
            />
          </el-select>
        </el-form-item>
      </div>
    </div>

    <!-- 仅对选中的行生成二维码开关 -->
    <el-form-item label="" size="mini">
      <div class="selected-rows-toggle">
        <span class="toggle-label">仅对选中的行生成二维码</span>
        <el-switch v-model="formData.generateForSelectedRows" />
      </div>
    </el-form-item>
    <el-button
      type="primary"
      size="mini"
      @click="generateQrCodes"
      :disabled="isGenerateButtonDisabled || loading"
    >
      生成二维码
    </el-button>

    <!-- 处理进度显示 -->
    <div v-if="loading" class="progress-container">
      <div class="progress-wrapper">
        <el-progress
          type="circle"
          :percentage="progress"
          :width="120"
          :show-text="true"
          :format="(percentage) => `处理进度 ${percentage}%`"
        />
      </div>
    </div>
  </el-form>
</template>

<style scoped>
.form :deep(.el-form-item__label) {
  font-size: 16px;
  color: var(--el-text-color-primary);
  margin-bottom: 0;
}
.form :deep(.el-form-item__content) {
  font-size: 16px;
}
.field-wrap :deep(.el-form-item__label) {
  font-size: 14px;
  font-weight: bold;
  line-height: 28px;
}

/* 模板预览样式 */
.template-preview {
  display: flex;
  gap: 20px;
  margin-top: 20px;
  align-items: flex-start;
}

/* 进度显示样式 */
.progress-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.progress-container h3 {
  margin: 0 0 20px 0;
  color: #303133;
  font-size: 20px;
}

.progress-wrapper {
  background-color: #fff;
  padding: 40px;
  border-radius: 12px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  text-align: center;
}

.progress-text {
  color: #606266;
  font-size: 14px;
  margin: 15px 0 0 0;
}

.template-image {
  width: 200px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: flex-start;
  overflow: visible;
}

.template-image img {
  max-width: 100%;
  height: auto;
  object-fit: contain;
}

.no-image {
  color: #909399;
  font-size: 14px;
}

.field-mappings {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.field-mappings :deep(.el-form-item) {
  margin-bottom: 0;
}

/* 选中行开关样式 */
.selected-rows-toggle {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 20px;
}

.toggle-label {
  font-size: 14px;
  color: var(--el-text-color-primary);
  font-weight: normal;
}
</style>
