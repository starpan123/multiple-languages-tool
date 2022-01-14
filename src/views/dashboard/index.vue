<template>
  <div class="app-container">
    <el-row class="export-lang">
      <el-button @click="exportJson(zhDataJson,'zhJson')">导出中文语言包</el-button>
      <el-button @click="exportJson(enDataJson,'enJson')">导出英文语言包</el-button>
    </el-row>
    <upload-excel-component

      :on-success="handleSuccess"
      :before-upload="beforeUpload"
    />
    <el-table
      :data="tableData"
      border
      highlight-current-row
      style="width: 100%;margin-top:20px;"
    >
      <el-table-column
        v-for="item of tableHeader"
        :key="item"
        :prop="item"
        :label="item"
      />
    </el-table>
  </div>
</template>

<script>
import UploadExcelComponent from '@/components/UploadExcel/index.vue'
import enAll from './langs/en'
import { saveAs } from 'file-saver'
import zhAll from './langs/zh'

export default {
  name: 'UploadExcel',
  components: { UploadExcelComponent },
  data() {
    return {
      tableData: [],
      tableHeader: [],
      enDataJson: enAll,
      zhDataJson: zhAll,
      zhData: {},
      enData: {}
    }
  },
  methods: {
    beforeUpload(file) {
      const isLt1M = file.size / 1024 / 1024 < 1

      if (isLt1M) {
        return true
      }

      this.$message({
        message: 'Please do not upload files larger than 1m in size.',
        type: 'warning'
      })
      return false
    },

    isChinese(temp) {
      var re = /[^\u4E00-\u9FA5]/
      if (re.test(temp)) return false
      return true
    },
    // 单独按词条进行匹配的方法可以忽略
    handleSuccessOn({ results, header }) {
      this.tableData = results
      this.tableHeader = header
      // 处理带code的词条
      const newTableData = {}
      for (var index in this.zhData) {
        // 值为字符串
        if (typeof this.zhData[index] === 'string') {
          const rowData = this.tableData.filter(
            (t) => t.name.trim() === this.zhData[index].trim()
          )[0]
          if (rowData) {
            const enname = rowData.enname
            newTableData[index] = enname
          } else {
            newTableData[index] = this.enData[index]
          }
        }

        //  值为对象
        if (typeof this.zhData[index] === 'object') {
          newTableData[index] = {}
          // 非route和general

          for (var oindex in this.zhData[index]) {
            let rowObjData
            if (index !== 'route' && index !== 'general') {
              rowObjData = this.tableData.filter(
                (t) => t.name.trim() === this.zhData[index][oindex].trim()
              )[0]
            } else {
              rowObjData = this.tableData.filter(
                (t) => t.name.trim() === oindex.trim()
              )[0]
            }
            if (rowObjData) {
              const enObjname = rowObjData.enname
              newTableData[index][oindex] = enObjname
            } else {
              newTableData[index][oindex] = this.enData[index][oindex]
            }
          }
        }
      }
      console.log('newTableData', newTableData)
    },

    // 导出Json对象到excel(key,value)
    async exportJson(jsonData, excelName) {
      var keyArryay = []
      // 等待获取json字符串
      await this.getAllJson(jsonData, '', '.', keyArryay)
      // 导出到excel并下载
      this.handleDownload(keyArryay, excelName)
    },
    // 导出多语言的key，value excel
    handleDownload(keyArryay, filename) {
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['key', 'value']
        const data = this.formatJson(tHeader, keyArryay)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: filename
        })
      })
    },
    // json数据map转换放回数组
    formatJson(filterVal, jsonData) {
      return jsonData.map(v => filterVal.map(j => {
        return v[j]
      }))
    },

    // 导入多语言翻译并生成对应的语言包JS文件
    async handleSuccess({ results, header }) {
      this.tableData = results
      this.tableHeader = header
      var jsonDataAll = zhAll
      await this.setLangJson(jsonDataAll, '', '.', this.tableData)
      this.exportLangDoc(jsonDataAll)
    },

    // 获取全部json字符串
    async getAllJson(jsons, name, sign, arr) {
      for (const key in jsons) {
        var k = name ? name + sign + key : key
        if (!(jsons[key] instanceof Object)) {
          arr.push({ 'key': k, 'value': jsons[key] })
        } else {
          this.getAllJson(jsons[key], k, sign, arr) // 如果是Object则递归
        }
      }
    },

    // 设置多语言json字符串
    async setLangJson(jsons, name, sign, arr) {
      for (const key in jsons) {
        var k = name ? name + sign + key : key
        if (!(jsons[key] instanceof Object)) {
          jsons[key] = arr.find(a => a.key.trim() === k).translation ? arr.find(a => a.key === k).translation : ''
        } else {
          this.setLangJson(jsons[key], k, sign, arr) // 如果是Object则递归
        }
      }
    },

    // 导出生成多语言文件
    exportLangDoc(langJsonData) {
      var content = 'module.exports =' + JSON.stringify(langJsonData)
      var blob = new Blob([content], { type: 'text/plain;charset=utf-8' })
      saveAs(blob, 'lang.js')
    }
  }}
</script>
<style lang="scss">
.export-lang{
    text-align: center;
    margin-bottom: 16px;
}
</style>
