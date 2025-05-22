<script setup>
import { ref, onMounted, watch } from 'vue';
import { ElMessage } from 'element-plus';

// 默认提成规则
const defaultRules = {
  '≤40': 30,
  '50': 45,
  '55,65': 45,
  '70,75,80': 55,
  '120,150': 100,
  '200': 130
};

// 当前提成规则
const rules = ref({...defaultRules});

// 新规则的键和值
const newRuleKey = ref('');
const newRuleValue = ref('');

// 编辑模式的规则键
const editingKey = ref('');

// 从本地存储加载规则
onMounted(() => {
  const savedRules = localStorage.getItem('commissionRules');
  if (savedRules) {
    try {
      rules.value = JSON.parse(savedRules);
    } catch (error) {
      console.error('解析保存的规则出错:', error);
      // 如果解析出错，使用默认规则
      rules.value = {...defaultRules};
    }
  }
});

// 监听规则变化，保存到本地存储
watch(rules, (newRules) => {
  localStorage.setItem('commissionRules', JSON.stringify(newRules));
}, { deep: true });

// 添加新规则
const addRule = () => {
  if (!newRuleKey.value || !newRuleValue.value) {
    ElMessage.warning('请输入规则范围和提成金额');
    return;
  }
  
  // 验证规则键格式
  const keyPattern = /^(≤\d+|\d+(,\d+)*)$/;
  if (!keyPattern.test(newRuleKey.value)) {
    ElMessage.warning('规则范围格式不正确，请使用 "≤数字" 或 "数字,数字,..." 格式');
    return;
  }
  
  // 验证规则值是否为数字
  const valueNum = Number(newRuleValue.value);
  if (isNaN(valueNum) || valueNum <= 0) {
    ElMessage.warning('提成金额必须是大于0的数字');
    return;
  }
  
  // 添加或更新规则
  rules.value[newRuleKey.value] = valueNum;
  
  // 清空输入
  newRuleKey.value = '';
  newRuleValue.value = '';
  
  ElMessage.success('规则添加成功');
};

// 删除规则
const deleteRule = (key) => {
  delete rules.value[key];
  ElMessage.success('规则删除成功');
};

// 开始编辑规则
const startEdit = (key, value) => {
  editingKey.value = key;
  newRuleKey.value = key;
  newRuleValue.value = value.toString();
};

// 保存编辑的规则
const saveEdit = () => {
  if (!newRuleKey.value || !newRuleValue.value) {
    ElMessage.warning('请输入规则范围和提成金额');
    return;
  }
  
  // 验证规则键格式
  const keyPattern = /^(≤\d+|\d+(,\d+)*)$/;
  if (!keyPattern.test(newRuleKey.value)) {
    ElMessage.warning('规则范围格式不正确，请使用 "≤数字" 或 "数字,数字,..." 格式');
    return;
  }
  
  // 验证规则值是否为数字
  const valueNum = Number(newRuleValue.value);
  if (isNaN(valueNum) || valueNum <= 0) {
    ElMessage.warning('提成金额必须是大于0的数字');
    return;
  }
  
  // 如果键已更改，删除旧键
  if (editingKey.value !== newRuleKey.value) {
    delete rules.value[editingKey.value];
  }
  
  // 添加或更新规则
  rules.value[newRuleKey.value] = valueNum;
  
  // 清空编辑状态
  editingKey.value = '';
  newRuleKey.value = '';
  newRuleValue.value = '';
  
  ElMessage.success('规则更新成功');
};

// 取消编辑
const cancelEdit = () => {
  editingKey.value = '';
  newRuleKey.value = '';
  newRuleValue.value = '';
};

// 重置为默认规则
const resetToDefault = () => {
  rules.value = {...defaultRules};
  ElMessage.success('已重置为默认规则');
};

// 计算提成金额
const calculateCommission = (settlement) => {
  if (!settlement) return 0;
  
  // 尝试将结算金额转换为数字
  const settlementAmount = Number(settlement);
  if (isNaN(settlementAmount)) return 0;
  
  // 遍历规则查找匹配项
  for (const key in rules.value) {
    // 处理小于等于类型的规则
    if (key.startsWith('≤')) {
      const threshold = Number(key.substring(1));
      if (settlementAmount <= threshold) {
        return rules.value[key];
      }
    } 
    // 处理精确匹配或多值匹配类型的规则
    else {
      const values = key.split(',').map(v => Number(v.trim()));
      if (values.includes(settlementAmount)) {
        return rules.value[key];
      }
    }
  }
  
  return 0; // 如果没有匹配的规则，返回0
};

// 获取格式化的规则数组
const getRules = () => {
  const formattedRules = [];
  
  for (const key in rules.value) {
    // 处理小于等于类型的规则
    if (key.startsWith('≤')) {
      const threshold = Number(key.substring(1));
      formattedRules.push({
        min: 0,
        max: threshold,
        value: rules.value[key],
        type: 'fixed' // 固定金额
      });
    } 
    // 处理精确匹配或多值匹配类型的规则
    else {
      const values = key.split(',').map(v => Number(v.trim()));
      
      // 对每个值创建一个规则
      for (const value of values) {
        formattedRules.push({
          min: value,
          max: value,
          value: rules.value[key],
          type: 'fixed' // 固定金额
        });
      }
    }
  }
  
  // 按照min值排序
  return formattedRules.sort((a, b) => a.min - b.min);
};

// 导出提成规则和计算函数
defineExpose({
  rules,
  calculateCommission,
  getRules
});
</script>

<template>
  <div class="rule-editor">
    <h3>提成规则设置</h3>
    
    <div class="rule-description">
      <h4>规则格式说明</h4>
      <ul class="rule-info-list">
        <li><span class="rule-key">≤40</span>: 表示厂家结算金额小于等于40元时的提成金额</li>
        <li><span class="rule-key">50</span>: 表示厂家结算金额等于50元时的提成金额</li>
        <li><span class="rule-key">55,65</span>: 表示厂家结算金额等于55元或65元时的提成金额</li>
      </ul>
    </div>
    
    <div class="rules-table">
      <table>
        <thead>
          <tr>
            <th>厂家结算金额范围</th>
            <th>提成金额</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(value, key) in rules" :key="key">
            <td>{{ key }}</td>
            <td>{{ value }}</td>
            <td>
              <button @click="startEdit(key, value)" class="btn edit-btn">编辑</button>
              <button @click="deleteRule(key)" class="btn delete-btn">删除</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    
    <div class="rule-form">
      <h4>{{ editingKey ? '编辑规则' : '添加新规则' }}</h4>
      <div class="form-group">
        <label>厂家结算金额范围:</label>
        <input 
          v-model="newRuleKey" 
          placeholder="如: ≤40 或 50 或 55,65" 
          type="text"
        />
      </div>
      <div class="form-group">
        <label>提成金额:</label>
        <input 
          v-model="newRuleValue" 
          placeholder="如: 30" 
          type="number"
          min="1"
        />
      </div>
      <div class="form-actions">
        <button v-if="!editingKey" @click="addRule" class="btn primary">添加规则</button>
        <template v-else>
          <button @click="saveEdit" class="btn primary">保存</button>
          <button @click="cancelEdit" class="btn">取消</button>
        </template>
      </div>
    </div>
    
    <div class="reset-action">
      <button @click="resetToDefault" class="btn warning">重置为默认规则</button>
    </div>
  </div>
</template>

<style scoped>
.rule-editor {
  background: #fff;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
}

.rule-description {
  background-color: #f9f9f9;
  border-radius: 5px;
  padding: 15px 20px;
  margin-bottom: 20px;
  border-left: 4px solid #409eff;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.rule-description h4 {
  margin-top: 0;
  color: #409eff;
  font-size: 16px;
  margin-bottom: 12px;
}

.rule-info-list li {
  margin-bottom: 10px;
  list-style-type: none;
  padding-left: 5px;
  position: relative;
}

.rule-key {
  background-color: #ecf5ff;
  color: #409eff;
  font-weight: bold;
  padding: 2px 8px;
  border-radius: 3px;
  border: 1px solid #d9ecff;
  display: inline-block;
  min-width: 50px;
  text-align: center;
  margin-right: 5px;
}

.rules-table {
  margin-bottom: 20px;
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  border: 1px solid #ddd;
  padding: 8px 12px;
  text-align: left;
}

th {
  background-color: #f2f2f2;
  font-weight: bold;
}

tr:nth-child(even) {
  background-color: #f9f9f9;
}

.rule-form {
  background-color: #f9f9f9;
  border-radius: 5px;
  padding: 15px;
  margin-bottom: 20px;
}

.form-group {
  margin-bottom: 15px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-actions {
  display: flex;
  gap: 10px;
}

.btn {
  padding: 6px 12px;
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

.btn.warning {
  background-color: #e6a23c;
  color: white;
}

.btn.warning:hover {
  background-color: #ebb563;
}

.btn.delete-btn {
  background-color: #f56c6c;
  color: white;
}

.btn.delete-btn:hover {
  background-color: #f78989;
}

.btn.edit-btn {
  background-color: #67c23a;
  color: white;
}

.btn.edit-btn:hover {
  background-color: #85cf5f;
}

.reset-action {
  text-align: center;
  margin-top: 20px;
}
</style>