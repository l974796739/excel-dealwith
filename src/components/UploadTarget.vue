<script setup>
import { ref, reactive, defineExpose } from 'vue';
import * as XLSX from 'xlsx';
import { ElMessage } from 'element-plus';

// 待处理表格数据
const processingTable = reactive({
  headers: [],
  rows: [],
  originalData: null
});

// 选择的待处理表格列索引
const selectedProductColumn = ref(-1);
const selectedOperationColumn = ref(-1);
const selectedSettlementColumn = ref(-1);
const selectedCommissionColumn = ref(-1);

// 上传待处理表格
const uploadProcessingTable = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const data = new Uint8Array(e.target.result);
      const workbook = XLSX.read(data, { type: 'array' });
      const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
      const jsonData = XLSX.utils.sheet_to_json(firstSheet, { header: 1 });
      
      if (jsonData.length < 2) {
        ElMessage.error('表格数据不足');
        return;
      }
      
      processingTable.headers = jsonData[0];
      processingTable.rows = jsonData.slice(1);
      processingTable.originalData = workbook;
      
      // 重置列选择
      selectedProductColumn.value = -1;
      selectedOperationColumn.value = -1;
      selectedSettlementColumn.value = -1;
      selectedCommissionColumn.value = -1;
      
      // 尝试根据列标题自动选择默认值
      processingTable.headers.forEach((header, index) => {
        // 查找可能的产品列
        if (header === 'D' || header.includes('产品') || header.includes('商品') || header.includes('产品类别')) {
          selectedProductColumn.value = index;
        }
        // 查找可能的操作列
        if (header === 'C' || header.includes('操作') || header.includes('类型') || header.includes('服务类型')) {
          selectedOperationColumn.value = index;
        }
        // 查找可能的厂家结算列
        if (header.includes('厂家结算') || header.includes('结算金额')) {
          selectedSettlementColumn.value = index;
        }
        // 查找可能的提成列
        if (header === 'N' || header.includes('提成') || header.includes('佣金')) {
          selectedCommissionColumn.value = index;
        }
      });
      
      ElMessage.success('待处理表格上传成功');
    } catch (error) {
      console.error('解析Excel出错:', error);
      ElMessage.error('解析Excel出错');
    }
  };
  reader.readAsArrayBuffer(file);
};

// 获取处理表格数据的方法
const getProcessingData = () => {
  return {
    headers: processingTable.headers,
    rows: processingTable.rows,
    originalData: processingTable.originalData,
    selectedProductColumn: selectedProductColumn.value,
    selectedOperationColumn: selectedOperationColumn.value,
    selectedSettlementColumn: selectedSettlementColumn.value,
    selectedCommissionColumn: selectedCommissionColumn.value
  };
};

// 导出组件方法和数据
defineExpose({
  processingTable,
  selectedProductColumn,
  selectedOperationColumn,
  selectedSettlementColumn,
  selectedCommissionColumn,
  uploadProcessingTable,
  getProcessingData
});
</script>

<template>
  <div class="upload-target">
    <h2>第二步：上传待处理表格</h2>
    <p>上传需要填充提成的表格</p>
    <div class="upload-area">
      <input type="file" accept=".xlsx,.xls" @change="uploadProcessingTable" />
    </div>
    
    <div v-if="processingTable.headers.length > 0" class="column-selection">
      <h3>选择数据列</h3>
      <div class="selection-group">
        <div class="selection-item">
          <label>产品列：</label>
          <select v-model="selectedProductColumn">
            <option value="-1">请选择产品列</option>
            <option v-for="(header, index) in processingTable.headers" :key="index" :value="index">
              {{ header }} (第{{ index + 1 }}列)
            </option>
          </select>
        </div>
        
        <div class="selection-item">
          <label>操作列：</label>
          <select v-model="selectedOperationColumn">
            <option value="-1">请选择操作列</option>
            <option v-for="(header, index) in processingTable.headers" :key="index" :value="index">
              {{ header }} (第{{ index + 1 }}列)
            </option>
          </select>
        </div>
        
        <div class="selection-item">
          <label>厂家结算列（可选）：</label>
          <select v-model="selectedSettlementColumn">
            <option value="-1">自动选择</option>
            <option v-for="(header, index) in processingTable.headers" :key="index" :value="index">
              {{ header }} (第{{ index + 1 }}列)
            </option>
          </select>
        </div>
        
        <div class="selection-item">
          <label>提成列（可选）：</label>
          <select v-model="selectedCommissionColumn">
            <option value="-1">自动选择</option>
            <option v-for="(header, index) in processingTable.headers" :key="index" :value="index">
              {{ header }} (第{{ index + 1 }}列)
            </option>
          </select>
        </div>
      </div>
    </div>
    
    <div v-if="processingTable.headers.length > 0" class="table-container">
      <table class="data-table">
        <thead>
          <tr>
            <th v-for="(header, index) in processingTable.headers" :key="index">
              {{ header }}
              <div v-if="index === selectedProductColumn" class="column-tag product-tag">产品列</div>
              <div v-if="index === selectedOperationColumn" class="column-tag operation-tag">操作列</div>
              <div v-if="index === selectedSettlementColumn" class="column-tag settlement-tag">厂家结算列</div>
              <div v-if="index === selectedCommissionColumn" class="column-tag commission-tag">提成列</div>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, rowIndex) in processingTable.rows.slice(0, 5)" :key="rowIndex">
            <td v-for="(cell, colIndex) in row" :key="colIndex">{{ cell }}</td>
          </tr>
        </tbody>
      </table>
      <div v-if="processingTable.rows.length > 5" class="table-note">
        <p>注：仅显示前5行数据预览</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.upload-target {
  margin-bottom: 30px;
}

.upload-area {
  border: 2px dashed #ddd;
  border-radius: 8px;
  padding: 30px;
  text-align: center;
  margin: 20px 0;
  background: #f9f9f9;
}

.column-selection {
  margin: 15px 0;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 5px;
  border: 1px solid #eee;
}

.selection-group {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.selection-item {
  flex: 1;
  min-width: 200px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

select {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.table-container {
  overflow-x: auto;
  margin: 20px 0;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
}

.data-table th, .data-table td {
  border: 1px solid #ddd;
  padding: 8px 12px;
  text-align: left;
  position: relative;
}

.data-table th {
  background-color: #f2f2f2;
  font-weight: bold;
}

.data-table tr:nth-child(even) {
  background-color: #f9f9f9;
}

.column-tag {
  position: absolute;
  top: 0;
  right: 0;
  font-size: 10px;
  padding: 2px 4px;
  border-radius: 2px;
  color: white;
}

.product-tag {
  background-color: #409eff;
}

.operation-tag {
  background-color: #67c23a;
}

.settlement-tag {
  background-color: #9254de;
}

.commission-tag {
  background-color: #e6a23c;
}

.table-note {
  text-align: center;
  color: #999;
  margin-top: 10px;
  font-size: 12px;
}
</style>