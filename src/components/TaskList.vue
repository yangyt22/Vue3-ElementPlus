<template>
  <div class="task-app">
    <header class="app-header">
      <h1><i class="el-icon-notebook-2"></i> 任务列表应用</h1>
      <!-- <p class="subtitle">使用 Vue3 Composition API 和 Element Plus 构建</p> -->
    </header>

    <div class="app-container">
      <!-- 添加任务表单 -->
      <el-card class="form-card">
        <template #header>
          <div class="card-header">
            <span><i class="el-icon-circle-plus-outline"></i> 添加新任务</span>
          </div>
        </template>

        <el-form
          :model="newTask"
          :rules="formRules"
          ref="taskFormRef"
          label-width="100px"
          @submit.prevent="addTask"
        >
          <el-form-item label="任务名称" prop="title">
            <el-input
              v-model="newTask.title"
              placeholder="请输入任务名称"
              clearable
            />
          </el-form-item>

          <el-form-item label="任务描述" prop="description">
            <el-input
              v-model="newTask.description"
              type="textarea"
              :rows="3"
              placeholder="请输入任务描述"
              maxlength="200"
              show-word-limit
            />
          </el-form-item>

          <el-form-item label="优先级" prop="priority">
            <el-select v-model="newTask.priority" placeholder="请选择优先级">
              <el-option label="低" value="low" />
              <el-option label="中" value="medium" />
              <el-option label="高" value="high" />
            </el-select>
          </el-form-item>

          <el-form-item label="截止日期" prop="dueDate">
            <el-date-picker
              v-model="newTask.dueDate"
              type="date"
              placeholder="选择截止日期"
              format="YYYY-MM-DD"
              value-format="YYYY-MM-DD"
            />
          </el-form-item>

          <el-form-item>
            <el-button type="primary" @click="addTask" :loading="addingTask">
              <el-icon><Plus /></el-icon> 添加任务
            </el-button>
            <el-button @click="resetForm">
              <i class="el-icon-refresh"></i> 重置
            </el-button>
          </el-form-item>
        </el-form>
      </el-card>

      <!-- 任务列表 -->
      <el-card class="table-card">
        <template #header>
          <div class="card-header">
            <span><i class="el-icon-tickets"></i> 任务列表</span>
            <div class="table-actions">
              <el-input
                v-model="searchQuery"
                placeholder="搜索任务..."
                clearable
                style="width: 200px; margin-right: 10px"
                @input="handleSearch"
              >
                <template #prefix>
                  <i class="el-icon-search"></i>
                </template>
              </el-input>
              <el-select
                v-model="filterStatus"
                placeholder="状态筛选"
                clearable
                style="width: 120px"
                @change="handleFilter"
              >
                <el-option label="全部" value="" />
                <el-option label="待完成" value="pending" />
                <el-option label="进行中" value="in-progress" />
                <el-option label="已完成" value="completed" />
              </el-select>
            </div>
          </div>
        </template>

        <!-- 任务表格 -->
        <el-table
          :data="paginatedTasks"
          stripe
          style="width: 100%"
          v-loading="loading"
          :empty-text="emptyText"
        >
          <el-table-column prop="title" label="任务名称" width="180">
            <template #default="{ row }">
              <div class="task-title">
                <i
                  class="el-icon-ticket"
                  :style="{ color: getPriorityColor(row.priority) }"
                ></i>
                <span :class="{ 'completed-task': row.status === 'completed' }">
                  {{ row.title }}
                </span>
              </div>
            </template>
          </el-table-column>

          <el-table-column
            prop="description"
            label="任务描述"
            width="250"
            show-overflow-tooltip
          />

          <el-table-column prop="priority" label="优先级" width="100">
            <template #default="{ row }">
              <el-tag :type="getPriorityTagType(row.priority)" size="small">
                {{ getPriorityLabel(row.priority) }}
              </el-tag>
            </template>
          </el-table-column>

          <el-table-column prop="status" label="状态" width="120">
            <template #default="{ row }">
              <el-select
                v-model="row.status"
                placeholder="选择状态"
                size="small"
                @change="updateTaskStatus(row.id, $event)"
              >
                <el-option label="待完成" value="pending" />
                <el-option label="进行中" value="in-progress" />
                <el-option label="已完成" value="completed" />
              </el-select>
            </template>
          </el-table-column>

          <el-table-column prop="dueDate" label="截止日期" width="120">
            <template #default="{ row }">
              <span
                :class="{ 'due-date-warning': isDueDateWarning(row.dueDate) }"
              >
                {{ row.dueDate || "无" }}
              </span>
            </template>
          </el-table-column>

          <el-table-column prop="createdAt" label="创建时间" width="140" />

          <el-table-column label="操作" width="120">
            <template #default="{ row }">
              <el-button type="danger" circle @click="deleteTask(row.id)">
                <el-icon>
                  <Delete />
                </el-icon>
              </el-button>
              <el-button
                type="info"
                :icon="Edit"
                circle
                @click="editTask(row)"
                style="margin-left: 5px"
              >
                <el-icon>
                  <Edit />
                </el-icon>
              </el-button>
            </template>
          </el-table-column>
        </el-table>

        <!-- 分页 -->
        <div class="pagination-container">
          <el-pagination
            v-model:current-page="currentPage"
            v-model:page-size="pageSize"
            :page-sizes="[5, 10, 20, 50]"
            layout="total, sizes, prev, pager, next, jumper"
            :total="filteredTasks.length"
            @size-change="handleSizeChange"
            @current-change="handleCurrentChange"
          />
        </div>
      </el-card>
    </div>

    <!-- 编辑任务对话框 -->
    <el-dialog v-model="editDialogVisible" title="编辑任务" width="500px">
      <el-form :model="editingTask" :rules="formRules" ref="editFormRef">
        <el-form-item label="任务名称" prop="title" label-width="100px">
          <el-input v-model="editingTask.title" />
        </el-form-item>
        <el-form-item label="任务描述" prop="description" label-width="100px">
          <el-input
            v-model="editingTask.description"
            type="textarea"
            :rows="3"
          />
        </el-form-item>
        <el-form-item label="优先级" prop="priority" label-width="100px">
          <el-select v-model="editingTask.priority">
            <el-option label="低" value="low" />
            <el-option label="中" value="medium" />
            <el-option label="高" value="high" />
          </el-select>
        </el-form-item>
        <el-form-item label="截止日期" prop="dueDate" label-width="100px">
          <el-date-picker
            v-model="editingTask.dueDate"
            type="date"
            format="YYYY-MM-DD"
            value-format="YYYY-MM-DD"
          />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="editDialogVisible = false">取消</el-button>
          <el-button type="primary" @click="saveEditedTask">保存</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from "vue";
import { ElMessage, ElMessageBox } from "element-plus";

// 任务表单引用
const taskFormRef = ref();
const editFormRef = ref();

// 任务状态
const tasks = ref([]);
const newTask = reactive({
  title: "",
  description: "",
  priority: "medium",
  dueDate: "",
  status: "pending",
  createdAt: "",
});

const editingTask = reactive({
  id: null,
  title: "",
  description: "",
  priority: "medium",
  dueDate: "",
  status: "pending",
});

// 表单验证规则
const formRules = {
  title: [
    { required: true, message: "请输入任务名称", trigger: "blur" },
    { min: 2, max: 50, message: "长度在 2 到 50 个字符", trigger: "blur" },
  ],
  description: [
    { required: true, message: "请输入任务描述", trigger: "blur" },
    { min: 5, max: 200, message: "长度在 5 到 200 个字符", trigger: "blur" },
  ],
  priority: [{ required: true, message: "请选择优先级", trigger: "change" }],
};

// 添加任务状态
const addingTask = ref(false);
const editDialogVisible = ref(false);
const loading = ref(false);

// 搜索和筛选
const searchQuery = ref("");
const filterStatus = ref("");

// 分页状态
const currentPage = ref(1);
const pageSize = ref(10);

// 生成示例数据
const generateSampleTasks = () => {
  const sampleTasks = [
    {
      id: 1,
      title: "完成项目文档",
      description: "编写项目技术文档和用户手册",
      priority: "high",
      status: "in-progress",
      dueDate: "2025-12-15",
      createdAt: "2025-11-01",
    },
    {
      id: 2,
      title: "学习Vue3 Composition API",
      description: "深入学习Vue3的组合式API特性",
      priority: "high",
      status: "pending",
      dueDate: "2025-11-30",
      createdAt: "2025-10-25",
    },
    {
      id: 3,
      title: "购买办公用品",
      description: "为办公室购买笔、纸等日常用品",
      priority: "low",
      status: "pending",
      dueDate: "2025-11-20",
      createdAt: "2025-11-05",
    },
    {
      id: 4,
      title: "修复登录页面Bug",
      description: "修复用户登录页面的验证码显示问题",
      priority: "medium",
      status: "completed",
      dueDate: "2025-11-10",
      createdAt: "2025-10-30",
    },
    {
      id: 5,
      title: "团队周会",
      description: "每周团队会议，讨论项目进展和问题",
      priority: "medium",
      status: "completed",
      dueDate: "2025-11-13",
      createdAt: "2025-11-06",
    },
    {
      id: 6,
      title: "设计新功能原型",
      description: "为新产品功能设计UI/UX原型",
      priority: "high",
      status: "in-progress",
      dueDate: "2025-12-01",
      createdAt: "2025-11-02",
    },
    {
      id: 7,
      title: "更新项目依赖",
      description: "更新项目中的第三方库到最新版本",
      priority: "low",
      status: "pending",
      dueDate: "2025-11-25",
      createdAt: "2025-11-08",
    },
    {
      id: 8,
      title: "编写单元测试",
      description: "为核心模块编写单元测试",
      priority: "medium",
      status: "in-progress",
      dueDate: "2025-11-28",
      createdAt: "2025-11-03",
    },
  ];

  tasks.value = sampleTasks;
};

// 添加任务
const addTask = () => {
  taskFormRef.value.validate((valid) => {
    if (valid) {
      addingTask.value = true;

      // 模拟API调用延迟
      setTimeout(() => {
        const newId =
          tasks.value.length > 0
            ? Math.max(...tasks.value.map((t) => t.id)) + 1
            : 1;
        const today = new Date().toISOString().split("T")[0];

        tasks.value.unshift({
          id: newId,
          title: newTask.title,
          description: newTask.description,
          priority: newTask.priority,
          status: "pending",
          dueDate: newTask.dueDate || "",
          createdAt: today,
        });

        ElMessage.success("任务添加成功！");
        resetForm();
        addingTask.value = false;
      }, 500);
    } else {
      ElMessage.error("请填写完整的任务信息");
      return false;
    }
  });
};

// 重置表单
const resetForm = () => {
  taskFormRef.value.resetFields();
  newTask.dueDate = "";
};

// 删除任务
const deleteTask = (id) => {
  ElMessageBox.confirm("确定要删除这个任务吗？", "删除确认", {
    confirmButtonText: "确定",
    cancelButtonText: "取消",
    type: "warning",
  })
    .then(() => {
      tasks.value = tasks.value.filter((task) => task.id !== id);
      ElMessage.success("任务删除成功！");

      // 如果当前页没有数据了，回到上一页
      if (paginatedTasks.value.length === 0 && currentPage.value > 1) {
        currentPage.value -= 1;
      }
    })
    .catch(() => {
      // 取消删除
    });
};

// 编辑任务
const editTask = (task) => {
  Object.assign(editingTask, task);
  editDialogVisible.value = true;
};

// 保存编辑
const saveEditedTask = () => {
  editFormRef.value.validate((valid) => {
    if (valid) {
      const index = tasks.value.findIndex((task) => task.id === editingTask.id);
      if (index !== -1) {
        tasks.value[index] = { ...editingTask };
        ElMessage.success("任务更新成功！");
        editDialogVisible.value = false;
      }
    }
  });
};

// 更新任务状态
const updateTaskStatus = (id, status) => {
  const task = tasks.value.find((task) => task.id === id);
  if (task) {
    task.status = status;
    ElMessage.success("任务状态已更新");
  }
};

// 搜索和筛选逻辑
const handleSearch = () => {
  currentPage.value = 1; // 重置到第一页
};

const handleFilter = () => {
  currentPage.value = 1; // 重置到第一页
};

// 计算属性
const filteredTasks = computed(() => {
  return tasks.value.filter((task) => {
    // 搜索过滤
    const matchesSearch =
      searchQuery.value === "" ||
      task.title.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      task.description.toLowerCase().includes(searchQuery.value.toLowerCase());

    // 状态过滤
    const matchesStatus =
      filterStatus.value === "" || task.status === filterStatus.value;

    return matchesSearch && matchesStatus;
  });
});

const paginatedTasks = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value;
  const end = start + pageSize.value;
  return filteredTasks.value.slice(start, end);
});

const emptyText = computed(() => {
  if (searchQuery.value || filterStatus.value) {
    return "没有找到匹配的任务";
  }
  return "暂无任务，请添加新任务";
});

// 统计数据
const pendingCount = computed(() => {
  return tasks.value.filter((task) => task.status === "pending").length;
});

const completedCount = computed(() => {
  return tasks.value.filter((task) => task.status === "completed").length;
});

const highPriorityCount = computed(() => {
  return tasks.value.filter((task) => task.priority === "high").length;
});

// 辅助函数
const getPriorityColor = (priority) => {
  const colors = {
    high: "#f56c6c",
    medium: "#e6a23c",
    low: "#67c23a",
  };
  return colors[priority] || "#909399";
};

const getPriorityTagType = (priority) => {
  const types = {
    high: "danger",
    medium: "warning",
    low: "success",
  };
  return types[priority] || "info";
};

const getPriorityLabel = (priority) => {
  const labels = {
    high: "高",
    medium: "中",
    low: "低",
  };
  return labels[priority] || "未知";
};

const isDueDateWarning = (dueDate) => {
  if (!dueDate) return false;

  const today = new Date();
  const due = new Date(dueDate);
  const diffDays = Math.ceil((due - today) / (1000 * 60 * 60 * 24));

  return diffDays <= 3 && diffDays >= 0;
};

// 分页事件处理
const handleSizeChange = (newSize) => {
  pageSize.value = newSize;
  currentPage.value = 1;
};

const handleCurrentChange = (newPage) => {
  currentPage.value = newPage;
};

// 生命周期钩子
onMounted(() => {
  loading.value = true;
  // 模拟数据加载延迟
  setTimeout(() => {
    generateSampleTasks();
    loading.value = false;
  }, 800);
});
</script>

<style scoped>
.task-app {
  font-family: "Helvetica Neue", Arial, sans-serif;
  /* background-color: #f5f7fa; */
  min-height: 100vh;
  /* padding: 20px; */
}

.app-header {
  text-align: center;
  margin-bottom: 30px;
}

.app-header h1 {
  color: #409eff;
  margin-bottom: 10px;
  font-size: 2.2rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.app-header h1 i {
  margin-right: 10px;
  font-size: 2rem;
}

.subtitle {
  color: #909399;
  font-size: 1rem;
}

.app-container {
  max-width: 1200px;
  margin: 0 auto;
}

.form-card,
.table-card {
  margin-bottom: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: bold;
  font-size: 1.1rem;
}

.card-header i {
  margin-right: 8px;
  color: #409eff;
}

.table-actions {
  display: flex;
  align-items: center;
}

.task-title {
  display: flex;
  align-items: center;
}

.task-title i {
  margin-right: 8px;
  font-size: 1.2rem;
}

.completed-task {
  text-decoration: line-through;
  color: #909399;
}

.due-date-warning {
  color: #e6a23c;
  font-weight: bold;
}

.pagination-container {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}

.stats-row {
  margin-top: 20px;
}

.stat-card {
  border-radius: 10px;
  transition: transform 0.3s;
}

.stat-card:hover {
  transform: translateY(-5px);
}

.stat-content {
  display: flex;
  align-items: center;
}

.stat-icon {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 15px;
  font-size: 1.8rem;
  color: white;
}

.stat-icon.total {
  background-color: #409eff;
}

.stat-icon.pending {
  background-color: #e6a23c;
}

.stat-icon.completed {
  background-color: #67c23a;
}

.stat-icon.high-priority {
  background-color: #f56c6c;
}

.stat-info {
  display: flex;
  flex-direction: column;
}

.stat-value {
  font-size: 1.8rem;
  font-weight: bold;
  line-height: 1.2;
}

.stat-label {
  color: #909399;
  font-size: 0.9rem;
}

.el-table :deep(.el-table__row) {
  transition: background-color 0.3s;
}

.el-table :deep(.el-table__row:hover) {
  background-color: #f5f7fa;
}

.el-button--primary {
  background-color: #409eff;
  border-color: #409eff;
}

.el-button--primary:hover {
  background-color: #66b1ff;
  border-color: #66b1ff;
}

@media (max-width: 768px) {
  .app-container {
    padding: 0 10px;
  }

  .table-actions {
    flex-direction: column;
    align-items: flex-start;
    margin-top: 10px;
  }

  .table-actions .el-input,
  .table-actions .el-select {
    width: 100%;
    margin-bottom: 10px;
  }

  .stats-row .el-col {
    margin-bottom: 15px;
  }
}
</style>
