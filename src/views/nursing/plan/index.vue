<template>
  <div class="app-container">
    <el-form
        :model="queryParams"
        ref="queryRef"
        :inline="true"
        v-show="showSearch"
        label-width="68px"
    >
      <el-form-item label="名称" prop="planName">
        <el-input
            v-model="queryParams.planName"
            placeholder="请输入名称"
            clearable
            @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select
            v-model="queryParams.status"
            placeholder="请选择状态"
            clearable
            style="width: 200px"
        >
          <el-option
              v-for="dict in nursing_plan_status"
              :key="dict.value"
              :label="dict.label"
              :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery"
        >搜索
        </el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
            type="primary"
            plain
            icon="Plus"
            @click="handleAdd"
            v-hasPermi="['elder:plan:add']"
        >新增
        </el-button>
      </el-col>
    </el-row>

    <el-table
        v-loading="loading"
        :data="planList"
        @selection-change="handleSelectionChange"
    >
      <el-table-column label="序号" type="index" width="50"/>
      <el-table-column label="名称" align="center" prop="planName"/>
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column
          label="创建时间"
          align="center"
          prop="createTime"
          width="180"
      >
        <template #default="scope">
          <span>{{
              parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}')
            }}</span>
        </template>
      </el-table-column>
      <el-table-column
          label="操作"
          align="center"
          fixed="right"
          width="280"
          class-name="small-padding fixed-width"
      >
        <template #default="scope">
          <el-button
              link
              type="primary"
              icon="Edit"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleUpdate(scope.row)"
              v-hasPermi="['elder:plan:edit']"
          >修改
          </el-button>
          <el-button
              link
              type="primary"
              icon="Delete"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleDelete(scope.row)"
              v-hasPermi="['elder:plan:remove']"
          >删除
          </el-button>
          <el-button
              link
              type="primary"
              icon="Delete"
              @click="handleLook(scope.row)"
              v-hasPermi="['elder:plan:remove']"
          >查看
          </el-button>
          <el-button
              link
              type="primary"
              :icon="scope.row.status == 0 ? 'Unlock' : 'lock'"
              @click="handleEnable(scope.row)"
          >{{ scope.row.status == 1 ? '禁用' : '启用' }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
        v-show="total > 0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getNursingPalnList"
    />

    <!--弹窗 -->
    <el-dialog
        title="新增护理计划"
        v-model="dialogVisible"
        width="900px"
        @close="cancel()"
    >
      <el-form
          ref="planRef"
          :model="formData"
          :rules="rules"
          label-width="120px"
      >
        <!-- 基础信息：改为两行布局，不要挤在一行 -->
        <el-row :gutter="20">
          <el-col :span="12">
            <el-form-item label="护理计划名称" prop="planName">
              <el-input
                  v-model="formData.planName"
                  :disabled="isLook"
                  placeholder="请输入护理计划名称"
              ></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="状态" prop="status">
              <el-radio-group v-model="formData.status" :disabled="isLook">
                <el-radio
                    v-for="dict in nursing_plan_status"
                    :value="dict.value"
                    :key="dict.value"
                >{{ dict.label }}
                </el-radio>
              </el-radio-group>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row :gutter="20">
          <el-col :span="12">
            <el-form-item label="排序" prop="sortNo">
              <el-input-number
                  :disabled="isLook"
                  v-model="formData.sortNo"
                  :min="0"
                  :max="999999"
                  :decimal-places="0"
                  style="width:100%"
                  @blur="textBlurNo"
                  @change="textBlurNo"
              ></el-input-number>
            </el-form-item>
          </el-col>
          <el-col :span="12"></el-col>
        </el-row>

        <!--护理项目列表，改用el-table实现，不再手写div表格，布局更稳定 -->
        <el-form-item label="护理项目" prop="projectPlans">
          <el-table
              :data="nursingPalnList"
              border
              size="small"
              style="width:100%"
          >
            <el-table-column label="护理项目名称" min-width="180">
              <template #default="{row}">
                <el-select
                    v-model="row.projectId"
                    placeholder="请选择护理项目"
                    :disabled="isLook"
                    clearable
                    style="width:100%"
                >
                  <el-option
                      v-for="opt in nursingProjectOptions"
                      :key="opt.value"
                      :label="opt.label"
                      :value="opt.value"
                  />
                </el-select>
              </template>
            </el-table-column>
            <el-table-column label="期望服务时间" min-width="140">
              <template #default="{row}">
                <el-time-picker
                    v-model="row.executeTime"
                    format="HH:mm:ss"
                    value-format="HH:mm:ss"
                    placeholder="请选择"
                    clearable
                    :disabled="isLook"
                    style="width:100%"
                />
              </template>
            </el-table-column>
            <el-table-column label="执行周期" min-width="120">
              <template #default="{row}">
                <el-select
                    v-model="row.executeCycle"
                    placeholder="请选择"
                    :disabled="isLook"
                    clearable
                    style="width:100%"
                >
                  <el-option
                      v-for="opt in executeCycleOptions"
                      :key="opt.value"
                      :label="opt.label"
                      :value="opt.value"
                  />
                </el-select>
              </template>
            </el-table-column>
            <el-table-column label="执行频次(次)" min-width="120">
              <template #default="{row}">
                <el-input-number
                    v-model="row.executeFrequency"
                    :min="1"
                    :max="7"
                    :disabled="isLook"
                    style="width:100%"
                />
              </template>
            </el-table-column>
            <el-table-column label="操作" width="120" v-if="!isLook">
              <template #default="{ $index }">
                <el-button
                    type="primary"
                    link
                    icon="Plus"
                    @click="handleRowAdd"
                >新增行
                </el-button>
                <el-button
                    type="danger"
                    link
                    icon="Delete"
                    :disabled="nursingPalnList.length <= 1"
                    @click="handleRowDel(null, $index)"
                >删除
                </el-button>
              </template>
            </el-table-column>
          </el-table>
        </el-form-item>
      </el-form>

      <div class="dialog-footer" v-if="!isLook">
        <el-button type="primary" @click="submitForm">确定</el-button>
        <el-button @click="cancel()">取消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script setup name="Plan">
import {
  listPlan,
  getPlan,
  delPlan,
  addPlan,
  updatePlan
} from '@/api/nursing/plan';
import {getProjectAll} from '@/api/nursing/project';
import {onMounted} from 'vue';

const {proxy} = getCurrentInstance();
const {nursing_plan_status} = proxy.useDict('nursing_plan_status');
const queryRef = ref(); // 表单
const planList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref('');
const dialogVisible = ref(false);
const nursingProjectOptions = ref([]);
const isLook = ref(false);
const formData = ref({
  status: '1',
  sortNo: 1,
});
// const planRef = ref({})

const queryParams = ref({
  pageNum: 1,
  pageSize: 10,
  planName: null,
  status: null,
});

const rules = ref({
  planName: [
    {
      required: true,
      message: '护理计划名称为空，请输入护理计划名称',
      trigger: 'blur',
    },
  ],
  status: [
    {
      required: true,
      message: '状态为空，请选择状态',
      trigger: 'change',
    },
  ],
  sortNo: [
    {
      required: true,
      message: '排序为空，请选择排序',
      trigger: 'change',
    },
  ],
});

const executeCycleOptions = ref([
  {label: '天', value: '0'},
  {label: '周', value: '1'},
  {label: '月', value: '2'},
]);

onMounted(() => {
  getAllProjectList();
});

//查询所有护理项目
const getAllProjectList = () => {
  getProjectAll().then((res) => {
    nursingProjectOptions.value = res.data;
  });
};

const nursingPalnList = ref([
  {
    projectId: '',
    executeTime: '',
    executeCycle: '',
    executeFrequency: '',
  },
]);

const handleRowAdd = () => {
  const obj = {
    projectId: '',
    executeTime: '',
    executeCycle: '',
    executeFrequency: '',
  };
  nursingPalnList.value.push(obj);
};
//删除行数据
const handleRowDel = (item, index) => {
  if (nursingPalnList.value.length === 1) return;
  nursingPalnList.value.splice(index, 1);
};

/** 查询护理计划列表 */
function getPalnList() {
  loading.value = true;
  listPlan(queryParams.value).then((response) => {
    planList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}

// 取消按钮
function cancel() {
  dialogVisible.value = false;
  isLook.value = false;
  reset();
}

// 表单重置
function reset() {
  formData.value = {
    planName: null,
    status: '1',
    sortNo: 1,
  };
  nursingPalnList.value = [
    {
      projectId: '',
      executeTime: '',
      executeCycle: '',
      executeFrequency: '1',
    },
  ];
  console.log(nursingPalnList.value);
  // proxy.resetForm('planRef');
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getPalnList();
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery();
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map((item) => item.id);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}

/** 新增按钮操作 */
function handleAdd() {
  dialogVisible.value = true;
  title.value = '添加护理计划';
}

/** 修改按钮操作 */
function handleUpdate(row) {
  const _id = row.id || ids.value;
  getDetails(_id);
}

// 获取详情
const getDetails = (id) => {
  getPlan(id).then((response) => {
    formData.value = response.data;
    formData.value.status = String(formData.value.status);
    formData.value.projectId = String(formData.value.projectId);
    nursingPalnList.value = formData.value.projectPlans;
    dialogVisible.value = true;
    title.value = '修改护理计划';
  });
};

/** 提交按钮 */
function submitForm() {
  //判断选择的护理项目是否有重复的
  if (hasDuplicateIds(nursingPalnList.value)) {
    // 提示重复
    proxy.$modal.msgError('请勿选择重复的护理项目');
    return;
  }
  formData.value['projectPlans'] = nursingPalnList.value;
  console.log(formData.value);

  proxy.$refs['planRef'].validate((valid) => {
    if (valid) {
      if (formData.value.id != null) {
        updatePlan(formData.value).then((response) => {
          proxy.$modal.msgSuccess('修改成功');
          cancel();
          getPalnList();
        });
      } else {
        addPlan(formData.value).then((response) => {
          proxy.$modal.msgSuccess('新增成功');
          cancel();
          getPalnList();
        });
      }
    }
  });
}

//帮我使用js代码，编写一段逻辑，判断数组中的对象的id是否有相同的(问心一言提供)
function hasDuplicateIds(objectsArray) {
  // 如果没有提供数组或数组为空，则直接返回false（因为没有元素可比较）
  if (!Array.isArray(objectsArray) || objectsArray.length === 0) {
    return false;
  }

  // 创建一个新的Set来存储id
  const idSet = new Set();

  // 遍历数组中的每个对象
  for (const obj of objectsArray) {
    // 假设每个对象都有一个id属性
    if (obj.projectId !== undefined) {
      // 尝试将id添加到Set中
      idSet.add(obj.projectId);
    }
  }

  // 比较Set的大小和数组的大小
  // 如果Set的大小小于数组的大小，说明有重复的id
  return idSet.size !== objectsArray.length;
}

// 使用 async/await 语法优化异步操作
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
      await updatePlan(param);
      // 刷新列表
      getPalnList();
      // 成功消息
      proxy.$modal.msgSuccess(`${info}成功`);
    }
  } catch (error) {
    // 异常处理：这里可以根据实际需求进行调整，例如打印错误日志或显示用户友好的错误信息
    console.error('操作失败，请重试或联系管理员。');
  }
};

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value;
  proxy.$modal
      .confirm('是否确认删除护理计划编号为"' + _ids + '"的数据项？')
      .then(function () {
        return delPlan(_ids);
      })
      .then(() => {
        getPalnList();
        proxy.$modal.msgSuccess('删除成功');
      })
      .catch(() => {
      });
}

//监听排序
const textBlurNo = () => {
  const data = formData.value.sortNo;
  if (data <= 1) {
    formData.value.sortNo = 1;
  }
  // 最大值设置
  if (data > 99999999) {
    formData.value.sortNo = 99999999;
  }
};
// 查看
const handleLook = (row) => {
  isLook.value = true;
  dialogVisible.value = true;
  getDetails(row.id);
};
getPalnList();
</script>
<style src="./index.scss" lang="scss"></style>
