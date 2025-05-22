<script setup>
import { ref } from 'vue';
import UploadStandard from './components/UploadStandard.vue';
import RuleEditor from './components/RuleEditor.vue';
import UploadTarget from './components/UploadTarget.vue';
import ProcessButton from './components/ProcessButton.vue';
import ResultTable from './components/ResultTable.vue';

// 引用组件
const uploadStandardRef = ref(null);
const ruleEditorRef = ref(null);
const uploadTargetRef = ref(null);
const processButtonRef = ref(null);
const resultTableRef = ref(null);

// 处理结果数据
const resultData = ref(null);

// 处理数据
const handleProcess = async () => {
  // 获取标准表格数据
  const standardData = uploadStandardRef.value.getStandardData();
  if (!standardData) {
    ElMessage.warning('请先上传标准表格');
    return;
  }
  
  // 获取提成规则
  const rules = ruleEditorRef.value.getRules();
  if (rules.length === 0) {
    ElMessage.warning('请先设置提成规则');
    return;
  }
  
  // 获取处理表格数据
  const processingData = uploadTargetRef.value.getProcessingData();
  if (!processingData) {
    ElMessage.warning('请先上传处理表格');
    return;
  }
  
  // 检查是否选择了厂家结算列
  if (processingData.selectedSettlementColumn === -1) {
    ElMessage.warning('请选择厂家结算列');
    return;
  }
  
  // 处理数据
  const result = await processButtonRef.value.processData(standardData, processingData, rules);
  if (result) {
    // 显示结果表格
    resultData.value = result;
  }
};
</script>

<template>
  <div class="container">
    <h1>Excel提成匹配工具</h1>
    
    <div class="card">
      <UploadStandard ref="uploadStandardRef" />
    </div>
    
    <div class="card">
      <RuleEditor ref="ruleEditorRef" />
    </div>
    
    <div class="card">
      <UploadTarget ref="uploadTargetRef" />
    </div>
    
    <ProcessButton ref="processButtonRef">
      <button @click="handleProcess" class="btn primary" :disabled="!uploadStandardRef?.standardTable.originalData || !uploadTargetRef?.processingTable.originalData">
        处理数据
      </button>
    </ProcessButton>
    
    <div class="card" v-if="resultData">
      <ResultTable ref="resultTableRef" :resultData="resultData" />
    </div>
  </div>
</template>

<style>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  margin-bottom: 30px;
  color: #333;
}

.card {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  padding: 20px;
  margin-bottom: 30px;
}

.btn {
  padding: 8px 16px;
  margin: 0 5px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #f0f0f0;
  transition: background-color 0.3s;
}

.btn:hover {
  background-color: #e0e0e0;
}

.btn.primary {
  background-color: #409eff;
  color: white;
}

.btn.primary:hover {
  background-color: #66b1ff;
}

.btn:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
</style>
