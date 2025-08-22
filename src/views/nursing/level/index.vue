<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="等级名称" prop="name">
        <el-input v-model="queryParams.name" placeholder="请输入等级名称" clearable @keyup.enter="handleQuery" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button type="primary" plain icon="Plus" @click="handleAdd" v-hasPermi="['nursing:level:add']">新增</el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="levelList" @selection-change="handleSelectionChange">
      <el-table-column label="序号" type="index" width="50" />
      <el-table-column label="等级名称" align="center" prop="name" />
      <el-table-column label="护理计划" align="center" prop="lplanId">
        <template #default="scope">
          {{ formatPlanName(scope.row.lplanId) }}
        </template>
      </el-table-column>
      <el-table-column label="护理费用" align="center" prop="fee" />
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="等级说明" align="center" prop="description" />
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleEdit(scope.row)"
            v-hasPermi="['nursing:level:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)"
            v-hasPermi="['nursing:level:remove']">删除</el-button>
          <el-button link type="primary" :icon="scope.row.status == 0 ? 'Unlock' : 'lock'"
            @click="handleEnable(scope.row)">{{ scope.row.status == 1 ? '禁用' : '启用' }}</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total > 0" :total="total" v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize" @pagination="getList" />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="90px">
        <el-form-item label="等级名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入等级名称" style="width: 300px" />
        </el-form-item>
        <el-form-item label="护理计划" prop="lplanId">
          <el-select v-model="form.lplanId" placeholder="请选择护理计划" style="width: 300px">
            <el-option v-for="item in lplanList" :key="item.id" :label="item.planName" :value="item.id" />
          </el-select>
        </el-form-item>
        <el-form-item label="护理费用" prop="fee">
          <el-input v-model="form.fee" placeholder="请输入护理费用" style="width: 300px" />
        </el-form-item>

        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <!-- <el-radio> 的 label 属性值是字符串类型（"0"和"1"）-->
            <el-radio label="1">正常</el-radio>
            <el-radio label="0">停用</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="等级说明" prop="description">
          <el-input v-model="form.description" placeholder="请输入等级说明" style="width: 300px" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Level">

import { ref, reactive, onMounted, getCurrentInstance } from 'vue';
import { ElRadioGroup, ElRadio } from 'element-plus'; // 添加这行导入语句
import { listLevel, getLevel, delLevel, addLevel, updateLevel } from "@/api/nursing/level";
import { listNursingPlanAll } from "@/api/nursing/nursingPlan";
const { proxy } = getCurrentInstance();
const { nursing_plan_status } = proxy.useDict('nursing_plan_status');
const levelList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref("");
// 声明护理计划列表变量
const lplanList = ref([]);
const data = reactive({
  form: {
    id: null,
    name: null,
    lplanId: null,
    fee: 0,
    status: null,
    description: null,
    createTime: null,
    createBy: null,
    updateBy: null,
    remark: null,
    updateTime: null

  },
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: null,
    status: null,
  },
});
data.rules = {
  name: [
    { required: true, message: '请输入等级名称', trigger: 'blur' },
    { min: 1, max: 50, message: '等级名称长度在1到50个字符之间', trigger: 'blur' }
  ],
  lplanId: [
    { required: true, message: '请选择护理计划', trigger: 'change' }
  ],
  fee: [
    { required: true, message: '请输入护理费用', trigger: 'blur' },
    {
      validator: (_, value, callback) => {
        const num = Number(value);
        if (isNaN(num)) {
          callback(new Error('护理费用必须为数字'));
        } else if (num < 0) {
          callback(new Error('护理费用不能为负数'));
        } else {
          callback();
        }
      },
      trigger: 'blur'
    }],
  status: [
    { required: true, message: '请选择状态', trigger: 'change' }
  ],
  description: [
    { max: 200, message: '等级说明不能超过200个字符', trigger: 'blur' }
  ]
};
const { queryParams, form, rules } = toRefs(data);


/** 查询护理等级列表 */
function getList() {
  loading.value = true;
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}

// 取消按钮
function cancel() {
  open.value = false;
  reset();
}

// 表单重置
function reset() {
  form.value = {
    id: null,
    name: null,
    lplanId: null,
    fee: null,
    status: null,
    description: null,
    createTime: null,
    createBy: null,
    updateBy: null,
    remark: null,
    updateTime: null
  };
  proxy.resetForm("levelRef");
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getList();
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef");
  handleQuery();
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}


getList();

const handleEnable = async (row) => {
  try {
    // 获取状态
    const status = row.status;
    const info = status === 0 ? '启用' : '禁用';

    // 使用模板字符串
    const confirmMessage = `是否确认${info}护理项目的数据项？`;

    // 确认操作
    if (await proxy.$modal.confirm(confirmMessage)) {
      // 更新参数
      const param = {
        id: row.id,
        status: status === 0 ? 1 : 0,
      };

      // 执行更新操作
      await updateLevel(param);
      // 刷新列表
      getList();
      // 成功消息
      proxy.$modal.msgSuccess(`${info}成功`);
    }
  } catch (error) {
    // 异常处理：这里可以根据实际需求进行调整，例如打印错误日志或显示用户友好的错误信息
    console.error('操作失败，请重试或联系管理员。');
  }
};

// 获取护理计划列表数据
const getNursingPlanList = async () => {
  try {
    const response = await listNursingPlanAll();

    // API返回格式为 { code: 200, rows: [...] }
    if (response.code === 200) {
      lplanList.value = response.data;
      console.log(lplanList.value);

    } else {
      proxy.$modal.msgError('获取护理计划失败');
    }
  } catch (error) {
    console.error('获取护理计划异常:', error);
    proxy.$modal.msgError('网络错误，无法加载护理计划');
  }
};

// 新增按钮操作
function handleAdd() {
  reset();
  open.value = true;
  title.value = "添加护理等级";
  getNursingPlanList();

}
// 编辑按钮操作
function handleEdit(row) {
  reset();
  open.value = true;
  title.value = "编辑护理等级";
  getNursingPlanList();
  // 将状态值转换为字符串类型，确保与radio的label类型匹配
  form.value = { ...row, status: row.status.toString() };
  console.log(form.value);
}
// 删除按钮操作
async function handleDelete(row) {
  try {
    // 确认删除
    if (await proxy.$modal.confirm('确定删除选中的护理等级吗？')) {
      // 执行删除操作
      delLevel(row.id).then(() => {
        // 删除成功后刷新列表
        getList();
        proxy.$modal.msgSuccess('删除成功');
      });
    }
  } catch (error) {
    console.error('删除操作异常:', error);
    proxy.$modal.msgError('删除失败');
  }
}


// 提交表单
async function submitForm() {
  try {
    await proxy.$refs.levelRef.validate();
    if (form.value.id) {
      await updateLevel(form.value);
      proxy.$modal.msgSuccess('更新成功');
    } else {
      await addLevel(form.value);
      proxy.$modal.msgSuccess('新增成功');
    }
    getList();
    open.value = false;
  } catch (error) {
    console.error('提交表单异常:', error);
    proxy.$modal.msgError('提交表单失败');
  }
}

// 格式化护理计划名称
/* const formatPlanName = (lplanId) => {
  const plan = lplanList.value.find(item => item.id === lplanId);
  return plan ? plan.planName : '未设置';
}; */

function formatPlanName(lplanId) {
  const plan = lplanList.value.find(item => item.id === lplanId);
  return plan ? plan.planName : '未设置';
}
// 在页面加载时获取护理计划列表
onMounted(() => {
  getNursingPlanList();
});

</script>
