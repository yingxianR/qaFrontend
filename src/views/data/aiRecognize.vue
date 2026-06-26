<template>
  <div class="excel-process-container">
    <a-card title="AI Excel文件处理" :bordered="false">
      <div class="upload-section">
        <a-upload
          :file-list="fileList"
          :before-upload="beforeUpload"
          :remove="handleRemove"
          accept=".xlsx,.xls"
          :max-count="1"
        >
          <a-button>
            <a-icon type="upload" />
            选择Excel文件
          </a-button>
        </a-upload>
        <div class="upload-tip">
          仅支持 .xlsx 和 .xls 格式文件
        </div>
      </div>

      <div class="process-section">
        <a-button
          type="primary"
          @click="processFile"
          :disabled="!fileList.length || processing"
          :loading="processing"
        >
          <a-icon type="thunderbolt" v-if="!processing" />
          {{ processing ? '处理中...' : '开始处理' }}
        </a-button>
      </div>

      <div class="result-section" v-if="resultFileUrl">
        <a-alert
          message="处理成功！"
          type="success"
          show-icon
        />
        <div class="result-actions">
          <a-button
            type="primary"
            @click="downloadFile"
            icon="download"
          >
            下载处理结果
          </a-button>
        </div>
      </div>

      <div class="error-section" v-if="errorInfo">
        <a-alert
          :message="errorInfo"
          type="error"
          show-icon
        />
      </div>
    </a-card>
  </div>
</template>

<script>
import { postAction, uploadAction } from '@/api/manage'

export default {
  name: 'AIExcelProcessor',
  data () {
    return {
      fileList: [],
      processing: false,
      resultFileUrl: '',
      errorInfo: '',
      baseUrl: 'http://192.168.3.88:8080/qaPlatform/dhb'
    }
  },

  methods: {
    beforeUpload (file) {
      // 检查文件类型
      const isExcel = file.type === 'application/vnd.ms-excel' ||
        file.type === 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'

      if (!isExcel) {
        this.$message.error('只能上传Excel文件!')
        return false
      }

      // 检查文件大小 (例如限制为10MB)
      const isLt10M = file.size / 1024 / 1024 < 10
      if (!isLt10M) {
        this.$message.error('文件大小不能超过10MB!')
        return false
      }

      // 立即上传文件
      this.uploadFileImmediately(file)

      // 暂时不添加到文件列表，等待上传成功后再添加
      return false
    },

    handleRemove () {
      this.fileList = []
      this.clearResult()
    },

    clearResult () {
      this.resultFileUrl = ''
      this.errorInfo = ''
    },

    async uploadFileImmediately (file) {
      this.uploading = true // 新增：上传状态
      try {
        const formData = new FormData()
        formData.append('file', file)

        const response = await uploadAction(`${this.baseUrl}uploadExcel`, formData)

        if (response.status) {
          this.uploadedFilePath = response.data.inputPath // 保存上传后的文件路径
          this.outFilePath = response.data.outputPath // 保存输出路径
          this.fileList = [file] // 上传成功后添加到文件列表
          this.$message.success('文件上传成功')
        } else {
          throw new Error(response.msg || '文件上传失败')
        }
      } catch (error) {
        this.$message.error(error.message || '文件上传失败')
        console.error('Upload error:', error)
      } finally {
        this.uploading = false
      }
    },

    async processFile () {
      if (!this.uploadedFilePath) {
        this.$message.warning('请先上传文件')
        return
      }

      this.processing = true
      this.clearResult()

      try {
        const aiResult = await postAction(`${this.baseUrl}aiReadExcel`, {
          inputFilePath: this.uploadedFilePath, // 使用已上传的文件路径
          outputFilePath: this.outFilePath
        })

        // 检查接口返回状态
        if (aiResult.status === 'SUCCESS') {
          this.resultFileUrl = this.outFilePath
          this.$message.success('文件处理成功！')
        } else {
          throw new Error(aiResult.message || '文件处理失败')
        }
      } catch (error) {
        this.errorInfo = error.message || '处理过程中发生错误'
        this.$message.error(error.message || '处理失败')
        console.error('Error processing file:', error)
      } finally {
        this.processing = false
      }
    },

    // 调用下载接口
    downloadFile () {
      if (!this.resultFileUrl) {
        this.$message.warning('没有可下载的文件')
        return
      }

      // 使用 downloadAction 下载文件，不需要传递参数，因为后端不接受参数
      // downloadAction(`${this.baseUrl}downloadExcel`, 'output.xlsx', {})

      // 直接创建链接下载，不使用 downloadAction
      const link = document.createElement('a')
      link.href = `${this.baseUrl}downloadExcel`
      link.download = 'output.xlsx'
      link.target = '_blank'
      link.click()
    }
  }
}
</script>

<style scoped>
.excel-process-container {
  max-width: 800px;
  margin: 20px auto;
  padding: 0 20px;
}

.upload-section {
  margin-bottom: 24px;
}

.upload-tip {
  margin-top: 8px;
  color: #999;
  font-size: 12px;
}

.process-section {
  text-align: center;
  margin: 24px 0;
}

.result-section {
  margin-top: 24px;
}

.result-actions {
  margin-top: 16px;
  text-align: center;
}

.error-section {
  margin-top: 24px;
}
</style>
