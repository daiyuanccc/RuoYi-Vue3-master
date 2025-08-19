<!-- 楼层新增编辑弹窗 -->
<template>
  <div class="dialog-form">
    <el-dialog
      :title="pageTitle + '房间'"
      v-model="formVisible"
      @close="onClickCloseBtn"
    >
      <!-- 表单内容 -->
      <div class="dialogCenter">
        <el-form
          ref="formRef"
          :model="formData"
          :rules="rules"
          label-width="100px"
        >
          <el-form-item label="房间号：" prop="code">
            <el-input
              v-model="formData.code"
              class="wt-400"
              placeholder="请输入"
              clearable
              show-limit-number
              :maxlength="5"
            >
            </el-input>
          </el-form-item>
          <el-form-item label="房间类型：" prop="typeName">
            <el-select
              v-model="formData.typeName"
              clearable
              filterable
              placeholder="请选择"
              @change="handleTypeChange" 
            >
              <el-option
                v-for="(item, index) in roomTypeData"
                :key="index"
                :value="item.name"
                :label="item.name"
                title=""
              >
                {{ item.name }}
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="排序：" prop="sort">
            <el-input-number
              v-model="formData.sort"
              :min="minNumber"
              large-number
              :max="maxNumber"
              :decimal-places="0"
              @blur="textBlurNo"
              @change="textBlurNo"
            ></el-input-number>
          </el-form-item>
        </el-form>
        <div class="dialog-footer">
          <el-button @click="onClickCloseBtn">取 消</el-button>
          <el-button type="primary" @click="onSubmit">确 定</el-button>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<script setup>
const { proxy } = getCurrentInstance();
import useUserStore from '@/store/modules/user';
// 获取父组件值、方法
const props = defineProps({
  // 弹层隐藏显示
  visible: {
    type: Boolean,
    default: false
  },
  // 房间类型
  houseTypeData: {
    type: Array,
    default: () => {
      return []
    }
  },
  //   详情数据
  data: {
    type: Object,
    default: () => {
      return {}
    }
  },
  // 最小值
  minNumber: {
    type: Number,
    default: 1
  },
  // 最大值
  maxNumber: {
    type: Number,
    default: 1
  },
  // 标题
  title: {
    type: String,
    default: '新增'
  },
  //   房间类型
  roomTypeData: {
    type: Array,
    default: () => []
  }
})
// ------定义变量------
// 触发父级事件
const emit = defineEmits([
  'handleClose',
  'fetchData',
  'handleAdd',
  'handleEdit'
])
const resetType = ref('empty') // 重置表单
const formVisible = ref(false) // 弹窗
const formRef = ref(null)

// 房间表单数据
const formData = ref({
  code: '',
  sort: 1,
  typeId: '',
  typeName: ''
})
// 表单校验
const rules = {
  code: [
    // 名称校验
    {
      required: true,
      message: '房间号为空，请输入房间号',
      trigger: 'blur'
    }
  ],
  typeName: [
    // 房间校验
    {
      required: true,
      message: '房间类型为空，请选择房间类型',
      trigger: 'change'
    }
  ],
  sort: [
    // 排序校验
    {
      required: true,
      message: '排序为空，请输入排序',
      trigger: 'blur'
    }
  ]
}
// 弹窗标题
const pageTitle = ref()
// 监听器，监听父级传递的visible值，控制弹窗显示隐藏
watch(props, (val) => {
  formVisible.value = val.visible
  pageTitle.value = val.title
})
// 监听器，监听父级传递的data值和roomTypeData值，确保数据完整后回显
watch(
  [() => props.data, () => props.roomTypeData],
  ([newData, newRoomTypes]) => {
    if (newData && Object.keys(newData).length > 0) {
      formData.value.id = newData.id
      formData.value.sort = newData.sort
      formData.value.code = newData.code
      
      // 确保房型数据已加载
      if (newRoomTypes && newRoomTypes.length > 0 && newData.typeId) {
        const roomType = newRoomTypes.find(item => item.id === newData.typeId)
        if (roomType) {
          formData.value.typeId = roomType.id
          formData.value.typeName = roomType.name
        } else {
          formData.value.typeId = newData.typeId
          formData.value.typeName = newData.typeName || ''
        }
      } else {
        formData.value.typeId = newData.typeId
        formData.value.typeName = newData.typeName || ''
      }
    }
  },
  { immediate: true, deep: true }
)
// -----定义方法------
// 提交表单
const onSubmit = () => {
proxy.$refs['formRef'].validate((valid) => {
    if (valid) {
        if (props.title === '新增') {
      // 调用新增接口
      emit('handleAdd', formData.value)
    } else {
      // 调用编辑接口
      emit('handleEdit', formData.value)
    }
    }
});
}

// 清除表单数据
const handleClear = () => {
  // 重置表单
  formRef.value.resetFields();
  formData.value.sort = 1
}
// 点击取消关闭
const onClickCloseBtn = () => {
  handleClear()
  emit('handleClose')
}
// 监听排序
const textBlurNo = () => {
  const data = Number(formData.value.sort)
  if (data <= 1) {
    formData.value.sort = 1
  }
  // 最大值设置
  if (data > 99999999) {
    formData.value.sort = 99999999
  }
}
// 传递给父组件的方法
defineExpose({
  handleClear
})
// 处理房间类型选择变化
const handleTypeChange = (selectedName) => {
  const selectedType = props.roomTypeData.find(item => item.name === selectedName)
  if (selectedType) {
    formData.value.typeId = selectedType.id
  }
}
</script>
