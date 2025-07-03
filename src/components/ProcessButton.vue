<script setup>
import { ref, defineExpose } from 'vue';
import { ElMessage } from 'element-plus';
import * as XLSX from 'xlsx';

// 结果表格数据
const resultTable = ref(null);

// 根据提成规则计算提成金额
const calculateCommissionByRules = (settlement, rules) => {
  // 确保结算金额是数字
  const settlementValue = parseFloat(settlement);
  
  // 如果结算金额不是有效数字，返回0
  if (isNaN(settlementValue) || settlementValue === 0) {
    return { value: 0, notFound: false };
  }
  
  // 查找匹配的规则
  for (const rule of rules) {
    const min = parseFloat(rule.min);
    const max = parseFloat(rule.max);
    
    // 检查结算金额是否在规则范围内
    if (settlementValue >= min && (max === 0 || settlementValue <= max)) {
      // 计算提成金额
      let commission = 0;
      
      if (rule.type === 'percentage') {
        // 百分比提成
        commission = settlementValue * (parseFloat(rule.value) / 100);
      } else {
        // 固定金额提成
        commission = parseFloat(rule.value);
      }
      
      // 四舍五入到两位小数
      commission = Math.round(commission * 100) / 100;
      
      return { value: commission, notFound: false };
    }
  }
  
  // 如果没有找到匹配的规则，返回0并标记为未找到
  return { value: 0, notFound: true };
};

// 处理数据方法
const processData = (standardData, processingData, rules) => {
  if (!standardData.originalData || !processingData.originalData) {
    ElMessage.warning('请先上传两份表格');
    return;
  }
  
  // 检查是否已选择列
  if (processingData.selectedProductColumn === -1 || processingData.selectedOperationColumn === -1) {
    ElMessage.warning('请先选择产品列和操作列');
    return;
  }
  
  try {
    // 使用用户选择的待处理表格列索引
    const processingProductColIndex = processingData.selectedProductColumn;
    const processingOperationColIndex = processingData.selectedOperationColumn;
    const processingCommissionColIndex = processingData.selectedCommissionColumn;
    const settlementColIndex = processingData.selectedSettlementColumn;
    
    // 确保厂家结算列和提成列已经存在
    if (settlementColIndex === -1 || processingCommissionColIndex === -1) {
      ElMessage.warning('厂家结算列或提成列未正确设置');
      return;
    }
    
    // 复制数据以便处理
    const resultData = JSON.parse(JSON.stringify(processingData.rows));
    
    // 从标准表格中提取产品和操作列表
    const products = standardData.rows.map(row => row[0]);
    const operations = standardData.headers.slice(1); // 第一列是产品名称，所以从索引1开始
    
    // 处理每一行数据
    for (let i = 0; i < resultData.length; i++) {
      const row = resultData[i];
      const product = row[processingProductColIndex];
      const operation = row[processingOperationColIndex];
      
      // 跳过空产品或操作
      if (!product || !operation) {
        console.log(`跳过第${i+1}行: 产品或操作为空`);
        continue;
      }
      
      console.log(`处理第${i+1}行:`, { 产品: product, 操作: operation });
      
      // 在标准表中查找匹配的产品和操作
      let found = false;
      
      // 查找产品在标准表中的行索引
      const productIndex = products.findIndex(p => p === product);
      if (productIndex !== -1) {
        // 查找操作在标准表中的列索引（支持模糊匹配操作名称）
        let operationIndex = operations.findIndex(o => o === operation);
        
        // 如果没有找到精确匹配的操作，尝试模糊匹配
        if (operationIndex === -1) {
          operationIndex = operations.findIndex(o => 
            o.includes(operation) || operation.includes(o)
          );
        }
        
        if (operationIndex !== -1) {
          // 获取对应的厂家结算金额（标准表格中的行和列交叉处的值）
          const settlement = standardData.rows[productIndex][operationIndex + 1];
          if (settlement !== undefined && settlement !== '') {
            console.log('找到匹配项，厂家结算金额:', settlement);
            // 更新待处理表格的厂家结算列
            row[settlementColIndex] = settlement;
            
            // 计算提成金额
            const commission = calculateCommissionByRules(settlement, rules);
            row[processingCommissionColIndex] = commission.value;
            
            // 检查结算金额是否在提成规则中
            if (commission.notFound && settlement !== 0) {
              row[processingCommissionColIndex] = '提成规则中未包含该厂家结算金额';
              row.ruleNotFound = true;
            }
            
            console.log(`已填充: 厂家结算=${settlement}, 提成=${commission.value}`);
            found = true;
          }
        }
      }
      
      // 如果没有找到精确匹配，尝试模糊匹配产品名称
      if (!found) {
        // 存储最佳匹配结果
        let bestMatch = null;
        let bestMatchScore = 0;
        
        for (let j = 0; j < products.length; j++) {
          // 检查产品名称是否包含在标准表中的产品名称中，或者标准表中的产品名称是否包含在当前产品中
          if (product.includes(products[j]) || products[j].includes(product)) {
            // 查找操作在标准表中的列索引（支持模糊匹配操作名称）
            let operationIndex = operations.findIndex(o => o === operation);
            
            // 如果没有找到精确匹配的操作，尝试模糊匹配
            if (operationIndex === -1) {
              operationIndex = operations.findIndex(o => 
                o.includes(operation) || operation.includes(o)
              );
            }
            
            if (operationIndex !== -1) {
              const settlement = standardData.rows[j][operationIndex + 1];
              if (settlement !== undefined && settlement !== '') {
                // 计算匹配分数（基于字符串长度的简单评分）
                const matchScore = Math.min(product.length, products[j].length);
                
                // 如果找到更好的匹配，更新最佳匹配
                if (matchScore > bestMatchScore) {
                  bestMatchScore = matchScore;
                  bestMatch = {
                    productIndex: j,
                    operationIndex: operationIndex,
                    settlement: settlement,
                    standardProduct: products[j]
                  };
                }
              }
            }
          }
        }
        
        // 如果找到最佳匹配，使用它
        if (bestMatch) {
          console.log(`找到最佳模糊匹配项: 标准表产品="${bestMatch.standardProduct}", 当前产品="${product}", 厂家结算金额:`, bestMatch.settlement);
          
          // 更新待处理表格的厂家结算列
          row[settlementColIndex] = bestMatch.settlement;
          
          // 计算提成金额
          const commission = calculateCommissionByRules(bestMatch.settlement, rules);
          row[processingCommissionColIndex] = commission.value;
          
          // 检查结算金额是否在提成规则中
          if (commission.notFound && bestMatch.settlement !== 0) {
            row[processingCommissionColIndex] = '提成规则中未包含该厂家结算金额';
            row.ruleNotFound = true;
          }
          
          console.log(`已填充: 厂家结算=${bestMatch.settlement}, 提成=${commission.value}`);
          found = true;
        }
      }
      
      if (!found) {
        console.log(`未找到匹配项: 产品=${product}, 操作=${operation}`);
        // 标记未匹配的行
        row.notMatched = true;
      }
    }
    
    // 更新结果表格
    resultTable.value = {
      headers: processingData.headers,
      rows: resultData
    };
    
    // 检查是否有成功匹配的提成数据
    let hasCommission = false;
    for (const row of resultData) {
      if (row[processingCommissionColIndex]) {
        hasCommission = true;
        break;
      }
    }
    
    if (hasCommission) {
      ElMessage.success('数据处理完成');
    } else {
      ElMessage.warning('未找到匹配的提成数据，请检查标准表格和待处理表格的数据是否对应');
    }
    
    console.log('处理结果:', resultTable.value);
    return resultTable.value;
  } catch (error) {
    console.error('处理数据出错:', error);
    ElMessage.error('处理数据出错');
    return null;
  }
};

// 导出组件方法和数据
defineExpose({
  resultTable,
  processData
});
</script>

<template>
  <div class="process-button">
    <slot></slot>
  </div>
</template>

<style scoped>
.process-button {
  display: flex;
  justify-content: center;
  margin: 20px 0;
}
</style>