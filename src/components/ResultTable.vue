<script setup>
import { defineProps, defineExpose } from 'vue';
import { ElMessage } from 'element-plus';
import * as XLSX from 'xlsx';

const props = defineProps({
  resultData: {
    type: Object,
    default: () => null
  }
});

// 导出结果表格
const exportResult = () => {
  if (!props.resultData) {
    ElMessage.warning('请先处理数据');
    return;
  }
  
  try {
    // 创建工作簿
    const wb = XLSX.utils.book_new();
    
    // 创建工作表
    const ws = XLSX.utils.aoa_to_sheet([props.resultData.headers, ...props.resultData.rows]);
    
    // 设置单元格样式（标记问题行）
    const problemRows = [];
    
    // 查找有问题的行
    props.resultData.rows.forEach((row, idx) => {
      if (row.notMatched || row.ruleNotFound) {
        problemRows.push(idx + 1); // +1 因为第一行是表头
      }
    });
    
    // 为有问题的行添加背景色
    if (problemRows.length > 0) {
      if (!ws['!rows']) ws['!rows'] = [];
      
      problemRows.forEach(rowIdx => {
        // 设置行样式
        if (!ws['!rows'][rowIdx]) ws['!rows'][rowIdx] = {};
        
        // 遍历该行的所有单元格，设置背景色
        for (let colIdx = 0; colIdx < props.resultData.headers.length; colIdx++) {
          const cellRef = XLSX.utils.encode_cell({r: rowIdx, c: colIdx});
          if (!ws[cellRef]) ws[cellRef] = { v: '' };
          if (!ws[cellRef].s) ws[cellRef].s = {};
          ws[cellRef].s.fill = { fgColor: { rgb: 'FFFFCC' } }; // 浅黄色背景
        }
      });
    }
    
    // 添加工作表到工作簿
    XLSX.utils.book_append_sheet(wb, ws, '处理结果');
    
    // 导出Excel文件
    XLSX.writeFile(wb, '处理结果.xlsx');
    
    ElMessage.success('导出成功');
  } catch (error) {
    console.error('导出Excel出错:', error);
    ElMessage.error('导出Excel出错');
  }
};

// 导出组件方法
defineExpose({
  exportResult
});
</script>

<template>
  <div class="result-table" v-if="resultData">
    <h2>处理结果</h2>
    <div class="table-container">
      <table class="data-table">
        <thead>
          <tr>
            <th v-for="(header, index) in resultData.headers" :key="index">{{ header }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, rowIndex) in resultData.rows.slice(0, 10)" :key="rowIndex" 
              :class="{ 'not-matched': row.notMatched, 'rule-not-found': row.ruleNotFound }">
            <td v-for="(cell, colIndex) in row" :key="colIndex">
              {{ colIndex === resultData.headers.length ? '' : cell }}
            </td>
          </tr>
        </tbody>
      </table>
      <div v-if="resultData.rows.length > 10" class="table-note">
        <p>注：仅显示前10行数据，完整数据请导出查看</p>
      </div>
    </div>
    
    <div class="actions">
      <button @click="exportResult" class="btn primary">导出结果</button>
    </div>
  </div>
</template>

<style scoped>
.result-table {
  margin-bottom: 30px;
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
}

.data-table th {
  background-color: #f2f2f2;
  font-weight: bold;
}

.data-table tr:nth-child(even) {
  background-color: #f9f9f9;
}

.not-matched {
  background-color: #fef0f0 !important;
}

.rule-not-found {
  background-color: #fdf6ec !important;
}

.table-note {
  text-align: center;
  color: #999;
  margin-top: 10px;
  font-size: 12px;
}

.actions {
  display: flex;
  justify-content: center;
  margin: 20px 0;
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
</style>