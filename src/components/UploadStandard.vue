<script setup>
import { ref, reactive, defineExpose } from 'vue';
import * as XLSX from 'xlsx';
import { ElMessage } from 'element-plus';

// 标准表格数据
const standardTable = reactive({
  headers: [],
  rows: [],
  originalData: null
});

// 编辑模式
const editMode = ref(false);

// 当前编辑的单元格
const currentEdit = reactive({
  rowIndex: -1,
  colIndex: -1,
  value: ''
});

// 上传标准表格
const uploadStandardTable = (event) => {
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
      
      standardTable.headers = jsonData[0];
      standardTable.rows = jsonData.slice(1);
      standardTable.originalData = workbook;
      
      ElMessage.success('标准表格上传成功');
    } catch (error) {
      console.error('解析Excel出错:', error);
      ElMessage.error('解析Excel出错');
    }
  };
  reader.readAsArrayBuffer(file);
};

// 切换编辑模式
const toggleEditMode = () => {
  editMode.value = !editMode.value;
  // 重置当前编辑的单元格
  currentEdit.rowIndex = -1;
  currentEdit.colIndex = -1;
  currentEdit.value = '';
};

// 编辑单元格
const editCell = (rowIndex, colIndex, value) => {
  currentEdit.rowIndex = rowIndex;
  currentEdit.colIndex = colIndex;
  currentEdit.value = value || '';
};

// 保存编辑
const saveEdit = () => {
  if (currentEdit.rowIndex >= 0 && currentEdit.colIndex >= 0) {
    // 保存普通单元格的编辑
    standardTable.rows[currentEdit.rowIndex][currentEdit.colIndex] = currentEdit.value;
  } else if (currentEdit.rowIndex === -1 && currentEdit.colIndex >= 0) {
    // 保存表头单元格的编辑
    standardTable.headers[currentEdit.colIndex] = currentEdit.value;
  }
  // 重置当前编辑的单元格
  currentEdit.rowIndex = -1;
  currentEdit.colIndex = -1;
  currentEdit.value = '';
};

// 添加行
const addRow = () => {
  // 创建新行并确保每个单元格都有空字符串值而不是undefined
  const newRow = Array(standardTable.headers.length).fill('');
  standardTable.rows.push(newRow);
};

// 添加列
const addColumn = () => {
  const columnName = prompt('请输入列名');
  if (columnName) {
    standardTable.headers.push(columnName);
    standardTable.rows.forEach(row => row.push(''));
  }
};

// 删除行
const deleteRow = (index) => {
  standardTable.rows.splice(index, 1);
  ElMessage.success('删除成功');
};

// 删除列
const deleteColumn = (index) => {
  standardTable.headers.splice(index, 1);
  standardTable.rows.forEach(row => row.splice(index, 1));
  ElMessage.success('删除成功');
};

// 保存标准表格
const saveStandardTable = () => {
  try {
    // 创建工作簿
    const wb = XLSX.utils.book_new();
    // 创建工作表
    const ws = XLSX.utils.aoa_to_sheet([standardTable.headers, ...standardTable.rows]);
    // 添加工作表到工作簿
    XLSX.utils.book_append_sheet(wb, ws, '标准表格');
    // 导出Excel文件
    XLSX.writeFile(wb, '标准表格.xlsx');
    
    ElMessage.success('保存成功');
  } catch (error) {
    console.error('保存Excel出错:', error);
    ElMessage.error('保存Excel出错');
  }
};

// 获取标准表格数据的方法
const getStandardData = () => {
  return {
    headers: standardTable.headers,
    rows: standardTable.rows,
    originalData: standardTable.originalData
  };
};

// 导出组件方法和数据
defineExpose({
  standardTable,
  uploadStandardTable,
  toggleEditMode,
  editMode,
  currentEdit,
  editCell,
  saveEdit,
  addRow,
  addColumn,
  deleteRow,
  deleteColumn,
  saveStandardTable,
  getStandardData
});
</script>

<template>
  <div class="upload-standard">
    <h2>第一步：上传标准表格</h2>
    <p>上传包含产品和操作对应提成的标准表格</p>
    <div class="upload-area">
      <input type="file" accept=".xlsx,.xls" @change="uploadStandardTable" />
    </div>
    
    <div v-if="standardTable.headers.length > 0" class="table-container">
      <div class="table-actions">
        <button @click="toggleEditMode" class="btn">
          {{ editMode ? '退出编辑' : '编辑表格' }}
        </button>
        <button v-if="editMode" @click="addRow" class="btn">添加行</button>
        <button v-if="editMode" @click="addColumn" class="btn">添加列</button>
        <button v-if="editMode" @click="saveStandardTable" class="btn primary">保存表格</button>
      </div>
      
      <div class="column-selection">
        <h3>标准表格说明</h3>
        <div class="selection-container">
          <p>标准表格结构说明：</p>
          <ul class="standard-table-info">
            <li><span class="info-label">第一行：</span>各种操作类型（如上门调试、安装等）</li>
            <li><span class="info-label">第一列：</span>各种产品名称（如台式净饮机等）</li>
            <li><span class="info-label">单元格值：</span>对应产品进行对应操作的厂家结算金额</li>
          </ul>
        </div>
      </div>
      
      <table class="data-table">
        <thead>
          <tr>
            <th v-if="editMode">#</th>
            <th v-for="(header, index) in standardTable.headers" :key="index">
              <div v-if="editMode" class="editable-cell">
                <input v-if="currentEdit.rowIndex === -1 && currentEdit.colIndex === index" 
                       v-model="currentEdit.value" 
                       @blur="saveEdit" 
                       @keyup.enter="saveEdit" />
                <span v-else @click="editCell(-1, index, header)">{{ header }}</span>
                <button @click="deleteColumn(index)" class="delete-btn">×</button>
              </div>
              <span v-else>{{ header }}</span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, rowIndex) in standardTable.rows" :key="rowIndex">
            <td v-if="editMode">
              <button @click="deleteRow(rowIndex)" class="delete-btn">×</button>
            </td>
            <td v-for="(cell, colIndex) in row" :key="colIndex">
              <div v-if="editMode" class="editable-cell">
                <input v-if="currentEdit.rowIndex === rowIndex && currentEdit.colIndex === colIndex" 
                       v-model="currentEdit.value" 
                       @blur="saveEdit" 
                       @keyup.enter="saveEdit" />
                <span v-else @click="editCell(rowIndex, colIndex, cell || '')">{{ cell || '' }}</span>
              </div>
              <span v-else>{{ cell || '' }}</span>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<style scoped>
.upload-standard {
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

.table-container {
  overflow-x: auto;
  margin: 20px 0;
}

.column-selection {
  margin: 15px 0;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 5px;
  border: 1px solid #eee;
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

.table-actions {
  display: flex;
  margin-bottom: 10px;
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

.editable-cell {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.editable-cell span {
  cursor: pointer;
  flex-grow: 1;
}

.editable-cell input {
  width: 100%;
  padding: 4px;
  border: 1px solid #409eff;
  border-radius: 4px;
}

.delete-btn {
  background-color: #f56c6c;
  color: white;
  border: none;
  border-radius: 4px;
  width: 20px;
  height: 20px;
  line-height: 16px;
  text-align: center;
  cursor: pointer;
  margin-left: 5px;
}

.delete-btn:hover {
  background-color: #f78989;
}

.standard-table-info li {
  margin-bottom: 8px;
  list-style-type: none;
  padding-left: 5px;
  border-left: 3px solid #409eff;
}

.info-label {
  font-weight: bold;
  color: #409eff;
  margin-right: 5px;
}
</style>