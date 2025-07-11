<template>
  <div class="forum-container">
    <Navbar />

    <el-row class="forum-header">
      <el-col :span="24">
        <el-input 
          v-model="searchText" 
          placeholder="搜索帖子" 
          prefix-icon="el-icon-search"
          @keyup.enter="searchPost">
          <template #append>
            <el-button @click="searchPost">搜索</el-button>
          </template>
        </el-input>

        <el-button type="primary" @click="openCreatePostDialog">
          发布新帖
        </el-button>
      </el-col>
    </el-row>

    <el-row class="forum-content">
      <el-col :span="24">
        <el-table :data="posts" stripe style="width: 100%">
          <el-table-column prop="title" label="帖子标题" min-width="300">
            <template #default="scope">
              <span @click="goToPostDetail(scope.row)">{{ scope.row.title }}</span>
            </template>
          </el-table-column>
          <el-table-column prop="author" label="作者" width="120"></el-table-column>
          <el-table-column prop="createTime" label="发布时间" width="160"></el-table-column>
          <el-table-column prop="views" label="浏览量" width="80"></el-table-column>
        </el-table>

        <el-pagination
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="currentPage"
          :page-sizes="[10, 20, 30]"
          :page-size="pageSize"
          layout="total, sizes, prev, pager, next, jumper"
          :total="totalPosts">
        </el-pagination>
      </el-col>
    </el-row>

    <!-- 发布帖子对话框 -->
    <el-dialog 
      v-model="showCreateDialog" 
      title="发布新帖"
      :before-close="handleClose">
      <el-form :model="newPost" :rules="postRules" ref="postFormRef" label-width="80px">
        <el-form-item label="标题" prop="title">
          <el-input v-model="newPost.title" placeholder="请输入帖子标题" maxlength="100" show-word-limit />
        </el-form-item>
        <el-form-item label="内容" prop="content">
          <el-input 
            type="textarea" 
            v-model="newPost.content" 
            rows="8" 
            placeholder="请输入帖子内容"
            maxlength="5000" 
            show-word-limit />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button type="primary" @click="submitPost">发布</el-button>
          <el-button @click="handleCloseDialog">取消</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import Navbar from '../components/Navbar.vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import { forumApi } from '../mock/forumData.js'

const router = useRouter()
const searchText = ref('')
const currentUser = 'test_user'

const originalPosts = ref([])
const posts = ref([])

const currentPage = ref(1)
const pageSize = ref(10)
const totalPosts = ref(0)

const showCreateDialog = ref(false)
const postFormRef = ref(null)
const isSubmitting = ref(false)

// 表单数据
const newPost = reactive({
  title: '',
  content: ''
})

// 表单验证规则
const postRules = reactive({
  title: [
    { required: true, message: '请输入帖子标题', trigger: 'blur' },
    { min: 3, max: 100, message: '标题长度在3 - 100个字符', trigger: 'blur' }
  ],
  content: [
    { required: true, message: '请输入帖子内容', trigger: 'blur' },
    { min: 10, message: '内容至少10个字符', trigger: 'blur' }
  ]
})

// 初始化加载帖子数据
const fetchPosts = async () => {
  try {
    const data = await forumApi.getPosts()
    originalPosts.value = data
    posts.value = data
    totalPosts.value = data.length
  } catch (error) {
    console.error('获取帖子列表失败', error)
    ElMessage({
      message: '加载帖子列表失败，请重试',
      type: 'error',
      duration: 3000
    })
  }
}

// 打开创建帖子对话框
const openCreatePostDialog = () => {
  showCreateDialog.value = true
}

// 提交表单
const submitPost = async () => {
  if (isSubmitting.value) return

  try {
    isSubmitting.value = true

    // 验证表单
    await new Promise((resolve, reject) => {
      postFormRef.value.validate((valid) => {
        if (valid) {
          resolve()
        } else {
          const firstError = document.querySelector('.el-form-item__error')
          firstError?.scrollIntoView({ behavior: 'smooth', block: 'center' })
          reject(new Error('表单验证失败'))
        }
      })
    })

    // 准备提交数据
    const postData = {
      title: newPost.title,
      content: newPost.content,
      author: 'test_user'
    }

    // 提交到API
    const createdPost = await forumApi.createPost(postData)

    // 更新本地数据
    posts.value.unshift(createdPost)
    originalPosts.value.unshift(createdPost)
    totalPosts.value = posts.value.length

    // 关闭对话框
    showCreateDialog.value = false

    // 显示成功提示
    ElMessage({
      message: '帖子发布成功！',
      type: 'success',
      duration: 2000
    })

    // 跳转到新帖子
    router.push(`/forum/detail/${createdPost.id}`)

    // 重置表单
    resetForm()

  } catch (error) {
    console.error('发布帖子出错:', error)
    ElMessage({
      message: error.message === '表单验证失败' ? '请检查表单输入' : '发布失败，请重试',
      type: 'error',
      duration: 3000
    })
  } finally {
    isSubmitting.value = false
  }
}

// 重置表单
const resetForm = () => {
  newPost.title = ''
  newPost.content = ''
  postFormRef.value?.resetFields()
}

// 关闭对话框
const handleClose = (done) => {
  handleCloseDialog()
  done()
}

const handleCloseDialog = () => {
  showCreateDialog.value = false
  resetForm()
}

const goToPostDetail = async (post) => {
  try {
    await forumApi.updateViews(post.id, post.views + 1)
    router.push(`/forum/detail/${post.id}`)
  } catch (error) {
    console.error('更新浏览量失败', error)
    router.push(`/forum/detail/${post.id}`)
  }
}

const searchPost = () => {
  if (!searchText.value) {
    posts.value = [...originalPosts.value]
  } else {
    const filteredPosts = originalPosts.value.filter(post => 
      post.title.includes(searchText.value) || 
      post.content.includes(searchText.value) || 
      post.author.includes(searchText.value)
    )
    posts.value = filteredPosts
  }
  totalPosts.value = posts.value.length
  currentPage.value = 1
}

// 分页处理
const handleSizeChange = (newSize) => {
  pageSize.value = newSize
}

const handleCurrentChange = (newPage) => {
  currentPage.value = newPage
}

onMounted(() => {
  fetchPosts()
})
</script>

<style scoped>
.forum-container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.forum-header {
  margin-bottom: 20px;
}

.forum-content {
  min-height: 500px;
}
</style>