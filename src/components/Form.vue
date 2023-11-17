<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-11-17 23:45
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from 'vue';
  import { bitable, FieldType, DateFormatter } from '@lark-base-open/js-sdk';
  import { ElMessage, ElMessageBox } from 'element-plus';

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  let dateFormatList = [
    {
      name: '格式: 01-30',
      value: 'MM-dd',
    },
    {
      name: '格式: 2021/01/30',
      value: 'yyyy/MM/dd',
    },
    {
      name: '格式: 2021-01-30',
      value: 'yyyy-MM-dd',
    },
    {
      name: '格式: 01/30/2021',
      value: 'MM/dd/yyyy',
    },
    {
      name: '格式: 30/01/2021',
      value: 'dd/MM/yyyy',
    },
  ];
  let dateFormat = ref();

  onMounted(async () => {
    const table = await base.getActiveTable();
    const tableMetaList = await table.getFieldMetaList();
    fieldOptions.value = tableMetaList.map((item) => ({ value: item.id, label: item.name }));
  });

  // 根据身份证生成生日的时间戳
  function extractBirthdayAndTimestamp(idCard) {
    const year = parseInt(idCard.substring(6, 10), 10);
    const month = parseInt(idCard.substring(10, 12), 10);
    const day = parseInt(idCard.substring(12, 14), 10);
    const timestamp = new Date(year, month - 1, day).getTime();

    return timestamp;
  }

  async function confirm() {
    if (!fieldId.value) {
      ElMessage({
        type: 'error',
        message: '请选择身份证号码列!',
      });
      return;
    }

    if (!dateFormat.value) {
      ElMessage({
        type: 'error',
        message: '请选择生日格式!',
      });
      return;
    }

    const table = await base.getActiveTable();
    // 无生日列表则创建, 有则提醒会覆盖数据
    const fieldMetaList = await table.getFieldMetaList();
    const hasBirthday = fieldMetaList.find((item) => item.name === '生日');

    if (!hasBirthday) {
      await table.addField({ type: FieldType.DateTime, name: '生日' });
      generateBirthdayRow();
    } else {
      ElMessageBox.confirm('已存在(生日列) ,后续操作会覆盖前面生日列数据,请确认是否继续?', 'Warning', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning',
      }).then(() => {
        ElMessage({
          type: 'success',
          message: '开始生成(生日列)数据...',
        });
        generateBirthdayRow();
      });
    }
  }

  /**
   * @desc  : 生成生日列
   */
  async function generateBirthdayRow() {
    const table = await base.getActiveTable();
    const recordList = await table.getRecordList();
    const field = await table.getField('生日'); // 选择某个多行文本字段
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    // const dateTimeField = await table.getField(field.id);
    await field.setDateFormat(dateFormat.value);

    // return;
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // 根据身份证号码获取生日
      await table.setCellValue(field.id, recordIds[index], extractBirthdayAndTimestamp(val[0]?.text));
    }

    ElMessage({
      message: '(生日列) 数据生成结束!',
      type: 'success',
    });
  }
</script>

<template>
  <div>
    <div class="title">请选择身份证号码列</div>
    <div>
      <el-select
        v-model="fieldId"
        placeholder="请选择身份证号码列"
        size="large"
      >
        <el-option
          v-for="item in fieldOptions"
          :key="item.value"
          :label="item.label"
          :value="item.value"
        />
      </el-select>
    </div>

    <div class="title top">请选择生日格式</div>
    <div>
      <el-select
        v-model="dateFormat"
        placeholder="请选择生日格式"
        size="large"
      >
        <el-option
          v-for="item in dateFormatList"
          :key="item.value"
          :label="item.name"
          :value="item.value"
        />
      </el-select>
    </div>

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
      >请确认</el-button
    >
  </div>
</template>

<style scoped>
  .title {
    font-size: 16px;
    font-weight: 700;
    color: rgb(31, 35, 41);
    margin-bottom: 14px;
  }

  .top {
    margin-top: 20px;
  }

  .btn {
    margin-top: 20px;
  }
  /* .form :deep(.el-form-item__label) {
    font-size: 16px;
    color: var(--el-text-color-primary);
    margin-bottom: 0;
  }
  .form :deep(.el-form-item__content) {
    font-size: 16px;
  } */
</style>
