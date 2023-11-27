<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : wangchao
 * @LastTime   : 2023-11-27 09:55
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from "vue";
  import { bitable, FieldType, DateFormatter } from "@lark-base-open/js-sdk";
  import { ElMessage, ElMessageBox } from "element-plus";
  import { addressCodeMap } from "@/addressCodeMap";

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  let dateFormatList = [
    {
      name: "格式: 01-30",
      value: "MM-dd",
    },
    {
      name: "格式: 2021/01/30",
      value: "yyyy/MM/dd",
    },
    {
      name: "格式: 2021-01-30",
      value: "yyyy-MM-dd",
    },
    {
      name: "格式: 01/30/2021",
      value: "MM/dd/yyyy",
    },
    {
      name: "格式: 30/01/2021",
      value: "dd/MM/yyyy",
    },
  ];
  let dateFormat = ref("yyyy-MM-dd");

  let table;
  let recordList;
  let recordIds;
  let tableMetaList;

  const age = ref(false);
  const sex = ref(false);
  const birthday = ref(false);
  const constellation = ref(false);
  const animal = ref(false);
  const address = ref(false);

  onMounted(async () => {
    table = await base.getActiveTable();
    recordList = await table.getRecordList();
    recordIds = await table.getRecordIdList(); // 获取所有记录 id

    tableMetaList = await table.getFieldMetaList();
    fieldOptions.value = tableMetaList.filter((item) => item.type === 1);
  });

  // 根据身份证生成生日的信息
  function extractBirthdayAndTimestamp(idCard) {
    const year = parseInt(idCard.substring(6, 10), 10);
    const month = parseInt(idCard.substring(10, 12), 10);
    const day = parseInt(idCard.substring(12, 14), 10);
    const timestamp = new Date(year, month - 1, day).getTime();

    return {
      year,
      month,
      day,
      timestamp,
    };
  }

  function calculateAgeFromTimestamp(birthTimestamp) {
    const currentDate = new Date();
    const birthDate = new Date(birthTimestamp);

    const currentYear = currentDate.getFullYear();
    const birthYear = birthDate.getFullYear();

    let age = currentYear - birthYear;

    // 检查生日是否已经过了今年
    const currentMonth = currentDate.getMonth();
    const birthMonth = birthDate.getMonth();

    if (currentMonth < birthMonth || (currentMonth === birthMonth && currentDate.getDate() < birthDate.getDate())) {
      age--;
    }

    return age;
  }

  async function confirm() {
    if (!fieldId.value) {
      ElMessage({
        type: "error",
        message: "请选择身份证号码列!",
      });
      return;
    }

    ElMessage({
      type: "success",
      message: "开始生成数据~",
    });

    // 获取字段列表
    const fieldMetaList = tableMetaList;

    if (birthday.value) {
      const hasBirthday = tableMetaList.find((item) => item.name === "生日");
      await judgeCreate(hasBirthday, "生日", "DateTime", generateBirthdayRow);
    }

    if (age.value) {
      const hasAge = tableMetaList.find((item) => item.name === "年龄");
      await judgeCreate(hasAge, "年龄", "Number", generateAgeRow);
    }

    if (sex.value) {
      const hasSex = tableMetaList.find((item) => item.name === "性别");
      await judgeCreate(hasSex, "性别", "Text", generateSexRow);
    }

    if (constellation.value) {
      const hasConstellation = tableMetaList.find((item) => item.name === "星座");
      await judgeCreate(hasConstellation, "星座", "Text", generateConstellationRow);
    }

    if (animal.value) {
      const hasAnimal = tableMetaList.find((item) => item.name === "生肖");
      await judgeCreate(hasAnimal, "生肖", "Text", generateAnimalRow);
    }

    if (address.value) {
      const hasAddress = tableMetaList.find((item) => item.name === "籍贯");
      await judgeCreate(hasAddress, "籍贯", "Text", generateAddressRow);
    }

    ElMessage({
      type: "success",
      message: "数据生成结束!",
    });
  }

  /**
   * @desc  : 判断是否生成
   * @param  {any} has 是否有字段
   * @param  {any} label 字段名
   * @param  {any} type 字段类型
   * @param  {any} fn 生成函数
   */
  async function judgeCreate(has, label, type, fn) {
    if (!has) {
      await table.addField({ type: FieldType[type], name: label });
    }

    fn();
  }

  /**
   * @desc  : 生成生日列
   */
  async function generateBirthdayRow() {
    const field = await table.getField("生日");

    await field.setDateFormat(dateFormat.value);

    let _list = [];
    for (const record of recordList) {
      const id = record.id;

      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: extractBirthdayAndTimestamp(val[0]?.text).timestamp,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  /**
   * @desc  : 生成年龄列
   */
  async function generateAgeRow() {
    const field = await table.getField("年龄");

    await field.setFormatter("0");

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: calculateAgeFromTimestamp(extractBirthdayAndTimestamp(val[0]?.text).timestamp),
        },
      });
    }

    await table.setRecords(_list);
  }
  /**
   * @desc  : 生成性别列
   */
  async function generateSexRow() {
    const field = await table.getField("性别");

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: getGenderByIdCard(val[0]?.text),
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getGenderByIdCard(idCard) {
    // 获取身份证号码中的第17位数字
    const checkCode = parseInt(idCard.charAt(16), 10);

    // 判断奇偶性来确定性别
    return checkCode % 2 === 0 ? "女" : "男";
  }

  /**
   * @desc  : 生成星座列
   */
  async function generateConstellationRow() {
    const field = await table.getField("星座");

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: getConstellationByIdCard(val[0]?.text),
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getConstellationByIdCard(idCard) {
    const birthdayStr = idCard.substring(6, 14);
    const birthday = new Date(
      parseInt(birthdayStr.substring(0, 4)),
      parseInt(birthdayStr.substring(4, 6)) - 1,
      parseInt(birthdayStr.substring(6, 8)),
    );

    const month = birthday.getMonth() + 1;
    const day = birthday.getDate();

    const constellations = [
      { name: "白羊座", startMonth: 3, startDay: 21, endMonth: 4, endDay: 19 },
      { name: "金牛座", startMonth: 4, startDay: 20, endMonth: 5, endDay: 20 },
      { name: "双子座", startMonth: 5, startDay: 21, endMonth: 6, endDay: 21 },
      { name: "巨蟹座", startMonth: 6, startDay: 22, endMonth: 7, endDay: 22 },
      { name: "狮子座", startMonth: 7, startDay: 23, endMonth: 8, endDay: 22 },
      { name: "处女座", startMonth: 8, startDay: 23, endMonth: 9, endDay: 22 },
      { name: "天秤座", startMonth: 9, startDay: 23, endMonth: 10, endDay: 23 },
      { name: "天蝎座", startMonth: 10, startDay: 24, endMonth: 11, endDay: 22 },
      { name: "射手座", startMonth: 11, startDay: 23, endMonth: 12, endDay: 21 },
      { name: "摩羯座", startMonth: 12, startDay: 22, endMonth: 1, endDay: 19 },
      { name: "水瓶座", startMonth: 1, startDay: 20, endMonth: 2, endDay: 18 },
      { name: "双鱼座", startMonth: 2, startDay: 19, endMonth: 3, endDay: 20 },
    ];

    for (const constellation of constellations) {
      if (
        (month === constellation.startMonth && day >= constellation.startDay) ||
        (month === constellation.endMonth && day <= constellation.endDay)
      ) {
        return constellation.name;
      }
    }

    // 如果没有匹配的星座，返回空字符串或其他默认值
    return "";
  }

  /**
   * @desc  : 生成生肖列
   */
  async function generateAnimalRow() {
    const field = await table.getField("生肖");

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: getChineseZodiacByIdCard(val[0]?.text),
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getChineseZodiacByIdCard(idCard) {
    // 获取身份证中的年份
    const year = parseInt(idCard.substring(6, 10));

    // 生肖映射表
    const zodiacMap = ["猴", "鸡", "狗", "猪", "鼠", "牛", "虎", "兔", "龙", "蛇", "马", "羊"];

    // 计算生肖的索引
    const zodiacIndex = year % 12;

    // 返回对应的生肖
    return zodiacMap[zodiacIndex];
  }

  /**
   * @desc  : 生成籍贯列
   */
  async function generateAddressRow() {
    const field = await table.getField("籍贯");

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: getNativePlaceByIdCard(val[0]?.text),
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);
  }

  function getNativePlaceByIdCard(idCard) {
    const provinceCode = idCard.substring(0, 3) + "000";
    const countyCode = idCard.substring(0, 6);

    // 根据省和县行政区划代码获取地址
    const province = addressCodeMap.province[provinceCode] || "未知省份";
    const county = addressCodeMap.county[countyCode] || "未知县区";

    return `${province}-${county}`;
  }
</script>

<template>
  <div>
    <div class="idCard">
      <div class="title">身份证号码列</div>
      <div>
        <el-select
          v-model="fieldId"
          placeholder="请选择身份证号码列"
          size="large"
        >
          <el-option
            v-for="item in fieldOptions"
            :key="item.id"
            :label="item.name"
            :value="item.id"
          />
        </el-select>
      </div>
    </div>

    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">生日</span>列:</div>
      <el-switch v-model="birthday" />
    </div>
    <div
      v-if="birthday"
      class="birthday"
    >
      <div class="title">生日格式</div>
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
    </div>
    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">年龄</span>列:</div>
      <el-switch v-model="age" />
    </div>

    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">性别</span>列:</div>
      <el-switch v-model="sex" />
    </div>
    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">星座</span>列:</div>
      <el-switch v-model="constellation" />
    </div>
    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">生肖</span>列:</div>
      <el-switch v-model="animal" />
    </div>
    <div class="switch">
      <div class="switch-tip">是否生成<span class="high">籍贯</span>列:</div>
      <el-switch v-model="address" />
    </div>

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
      >确认生成</el-button
    >
  </div>
</template>

<style scoped>
  .idCard {
    margin-bottom: 24px;
  }

  .title {
    font-size: 16px;
    font-weight: 700;
    color: rgb(31, 35, 41);
    margin-bottom: 10px;
  }

  .btn {
    margin-top: 14px;
  }

  .birthday {
    margin-bottom: 14px;
  }

  .switch {
    display: flex;
    align-items: center;
    margin-bottom: 14px;

    .switch-tip {
      margin-right: 5px;
    }

    .high {
      color: dodgerblue;
    }
  }
</style>
