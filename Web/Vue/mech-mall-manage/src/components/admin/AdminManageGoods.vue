<script setup>
import apiRequests from '@/apis';
import { computed, onMounted, ref, watch } from 'vue'
import { SystemConsts } from "@/enums/SystemConsts";
import { useStore } from 'vuex';
import { ElMessage } from "element-plus";
import { ElConfigProvider } from 'element-plus'
import zhCn from 'element-plus/lib/locale/lang/zh-cn'
import EditProdutInfo from './EditProdutInfo.vue';
import AddProductInfo from './AddProductInfo.vue';

//data
const store = useStore()
const productList = ref([])
const totalProduct = ref(0)
const columnWidth = '200px'
const currentPage = ref(1)
const pageSize = ref(10)
// 要删除的用户id
let curDelProductId = -1
// 警告提示
const deleteProductDialogVisible = ref(false)
const searchQuery = ref('')

//method
//获取所有产品
const getAllProduct = async (page = 1, size = 10)=> {
  const res = await apiRequests.getAllProduct(page, size)
  productList.value = res.data
  totalProduct.value = res.total
}
// 搜索用户
const searchProduct = async (query, page = 1, size = 10) => {
  if (!query) {
    getAllProduct(page, size) // 如果查询为空，调用获取所有用户的函数
    return;
  }
  const res = await apiRequests.searchProduct(query, page, size);
  console.log(res);

  productList.value = res.data;
  totalProduct.value = res.total;
}
// 复选框操作，可以大量删除用户
const handleSelectionChange = () => {
}
// 页码改变时的处理函数
const handlePageChange = (page) => {
  currentPage.value = page
  searchProduct(searchQuery.value, page, pageSize.value)
}

// 每页条数改变时的处理函数
const handleSizeChange = (size) => {
  pageSize.value = size
  searchProduct(searchQuery.value, currentPage.value, size)
}

//查找产品
const handleSearch = async () => {
  searchProduct(searchQuery.value, currentPage.value, pageSize.value)
}
//添加用户
const handleAddProduct = async () =>{
  //显示对话框
  store.commit('setAddProductInfoDiaVisible', true)
}
//修改信息
const handleEdit = (product) => {
  // 将要编辑的用户信息存入 store 里
  store.dispatch('updateSelectedProductInfo', product)
  // 显示编辑用户对话框
  store.commit('setEditProductInfoDiaVisible', true)

}
//修改状态
const handleHotButtonClick = async(row) => {
      // Toggle isHot value between 1 and 2
    row.isHot = row.isHot === 1 ? 2 : 1;
    const res = await apiRequests.updateStatus(row)
    if(res.success){
      ElMessage.success(res.message)
      console.log(row)
      // 刷新
      searchProduct(searchQuery.value, currentPage.value, pageSize.value)
    }else {
    ElMessage.error(res.message)
  }

}

// 删除用户
const handleDelete = (product) => {
  deleteProductDialogVisible.value = true
  curDelProductId = product.id
}
//删除的具体方法
const deleteProductAction = async () => {
  const res = await apiRequests.deleteProduct(curDelProductId)
  if (res.success) {
    deleteProductDialogVisible.value = false
    ElMessage.success(res.message)
    // 刷新
    searchProduct(searchQuery.value, currentPage.value, pageSize.value)
  } else {
    ElMessage.error(res.message)
  }
}

//改变状态
const getStatusText = (status) => {
      switch (status) {
        case 1:
          return '待售';
        case 2:
          return '出售';
        case 3:
          return '停售';
        default:
          return '未知状态';
      }
}
const handleStatusButtonClick = async(row) => {
      // 状态在点击之后只有上架和停售  待售是初始状态
    row.status = row.status === 2 ? 3 : 2;
    const res = await apiRequests.updateStatus(row)
    if(res.success){
      ElMessage.success(res.message)
      console.log(row)
      // 刷新
      searchProduct(searchQuery.value, currentPage.value, pageSize.value)
    }else {
    ElMessage.error(res.message)
  }

}

const isAdminProductFresh = computed(() => store.getters.isAdminProductFresh)

watch(isAdminProductFresh, (newVal, oldVal) => {
  if (newVal === oldVal)
    return
  searchProduct(searchQuery.value, currentPage.value, pageSize.value)
})
// onMounted
onMounted(() => {
  searchProduct(searchQuery.value, currentPage.value, pageSize.value)
})
getAllProduct();
</script>

<template>
  <div class="admin-goods">
    <!-- 搜索栏 -->
    <div class="search-bar">
      <el-input
        v-model="searchQuery"
        placeholder="请输入产品编号或名称"
        class="search-input"
        clearable
      />
      <el-button type="primary" @click="handleSearch">搜索</el-button>

        <!-- 增加商品按钮 -->
    <div class="add-product-button">
      <el-button type="primary" @click="handleAddProduct">增加商品</el-button>
    </div>
  </div>

    <!-- 产品信息表格 -->
    <el-table
      :data="productList"
      style="width: 100%; height: 82%"
      class=""
      highlight-current-row
      @selection-change="handleSelectionChange"
      :header-cell-style="SystemConsts.headerCellStyle"
      :cell-style="SystemConsts.cellStyle"
      :header-cell-class-name="'fixed-header'"
    >
      <el-table-column type="selection" width="80" align="center"></el-table-column>
      <el-table-column prop="id" width="80px" label="编号" align="center" />
      <el-table-column prop="name" :width="columnWidth" label="商品名称" align="center" />
      <el-table-column prop="price" :width="columnWidth" label="价格" align="center" />
      <el-table-column prop="status" :width="columnWidth" label="状态" align="center">
        <template v-slot="{ row }">
          <el-button
            :type="['priamry','danger']"
            :size="'mini'"
          >
            {{ getStatusText(row.status) }}
          </el-button>

          <el-button
            class="custom-red-button"
            :type="['priamry', 'danger']"
            :size="'mini'"
            @click="handleStatusButtonClick(row)" 
          >
            {{ row.status === 2 ? '上架' : '下架' }}
          </el-button>

        </template>
      </el-table-column>
      <el-table-column prop="isHot" :width="columnWidth" label="是否热销" align="center">
        <template v-slot="{ row }">
          <el-button
            :type="danger"
            :size="'mini'"
          >
            {{ row.isHot === 1 ? '热销' : '一般' }}
          </el-button>

          <el-button
            class="custom-red-button"
            :type="danger"
            :size="'mini'"
            @click="handleHotButtonClick(row)" 
          >
            {{ row.isHot === 1 ? '非热销' : '热销' }}
          </el-button>

        </template>
      </el-table-column>



      <el-table-column label="操作" :width="200" align="center">
        <template #default="{ row }">
          <el-button type="primary" size="mini" @click="handleEdit(row)"
            >编辑</el-button
          >
          <el-button type="danger" size="mini" @click="handleDelete(row)"
            >删除</el-button
          >
        </template>
      </el-table-column>
    </el-table>
    <el-config-provider :locale="zhCn"
      ><el-pagination
        @size-change="handleSizeChange"
        @current-change="handlePageChange"
        :current-page="currentPage"
        :page-size="pageSize"
        :page-sizes="[1, 3, 5, 8, 10, 12, 20]"
        layout="total, sizes, prev, pager, next, jumper"
        :total="totalProduct"
    /></el-config-provider>

    <!-- 删除用户警告 -->
    <el-dialog
      v-model="deleteProductDialogVisible"
      width="450px"
      destroy-on-close
      center
      class="delete-user-dia"
    >
      <!-- 使用 title 插槽自定义标题样式 -->
      <template #title>
        <div class="custom-dialog-title">警 告</div>
      </template>

      <span style="font-size: 20px">
        <strong>你是否要删除该用户，确认后操作不可逆。</strong>
      </span>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="deleteProductDialogVisible = false">取消</el-button>
          <el-button type="primary" @click="deleteProductAction">确定</el-button>
        </div>
      </template>
    </el-dialog>

    <EditProdutInfo></EditProdutInfo>
    <AddProductInfo></AddProductInfo>

  </div>
</template>



<style>
.admin-contianer {
  background-color: rgb(255, 255, 255);
}
.el-table {
  overflow: auto;
}
.delete-user-dia {
  font-size: 26px;
}
.search-bar {
  margin: 20px auto;
  display: flex;
}
.fixed-header {
  position: sticky;
  top: 0;
  background-color: white;
  z-index: 1;
}
.search-bar {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.search-input {
  width: 200px;
  margin-right: 10px;
}

.add-product-button .el-button {
  background-color: #409eff; 
  color: #fff; /* 设置按钮文字颜色为白色 */
}

.add-product-button {
  margin-left: 800px; /* 为了与搜索栏之间增加一些间距 */
}

.custom-red-button {
  background-color: rgb(222, 32, 32);
  color: white;
}
</style>