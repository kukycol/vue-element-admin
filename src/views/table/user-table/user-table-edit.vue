<template>
  <div class="app-container">
    <el-form ref="dataForm" :inline="false" :rules="rules" :model="temp" label-position="right" label-width="140px" style="width: 100%; ">
      <div v-for="(item,index) in groups" :key="index">
        <div>
          <em>{{ item.groupName }}</em>
          <hr>
        </div>
        <div style="margin-left: 100px;margin-top: 20px;">
          <el-row>
            <el-col v-for="(field,index) in item.groupData" :key="index" :span="12">
              <el-form-item :label="field.fieldLabel" :prop="field.fieldName">
                <el-input
                  v-if="field.fieldType==='text'"
                  v-model="temp[field.fieldName]"
                  :placeholder="`请输入${field.fieldLabel}`"
                />
                <el-date-picker
                  v-if="field.fieldType==='date'"
                  v-model="temp[field.fieldName]"
                  type="date"
                  :placeholder="`请选择${field.fieldLabel}`"
                  :picker-options="pickerOptions"
                />
                <el-input-number
                  v-if="field.fieldType==='number'"
                  v-model="temp[field.fieldName]"
                  :precision="field.precision===null?0:field.precision"
                  :step="field.step===null?1:field.step"
                  controls-position="right"
                  :min="field.min===null?'-Infinity':field.min"
                  :max="field.max===null?'Infinity':field.max"
                  size=""
                  style="width: 100%;text-align: left"
                  :placeholder="`请输入${field.fieldLabel}`"
                />
                <el-select
                  v-if="field.fieldType==='select'"
                  v-model="temp[field.fieldName]"
                  filterable
                  clearable
                  :placeholder="`请选择${field.fieldLabel}`"
                >
                  <el-option
                    v-for="item in dict[`${field.fieldName}Options`]"
                    :key="item.dictValue"
                    :label="item.dictItem"
                    :value="item.dictValue"
                  />
                </el-select>
              </el-form-item>
            </el-col>
          </el-row>
        </div>
      </div>

    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button @click="dialogFormVisible = false">取消</el-button>
      <el-button type="primary" @click="dialogStatus==='create'?createData():updateData()">提交</el-button>
    </div>
  </div>
</template>

<style>

.dialog-footer{
  margin-top: 30px;
  text-align: center;
  background-color: #3A71A8;
  padding: 15px;
  border-radius: 8px;
}

.el-input__inner{
  /*width: 300px;*/
  width: 100%;
}

.el-textarea__inner{
  width: 300px;
}

.el-divider__text{
  color: #0a76a4;
}

hr{
  height: 1px;
  border: none;
  background-color: #c0c4cc;
}
em{
  color: #0a76a4;
}
.el-date-editor.el-input,.el-date-editor.el-input__inner{
  width: 100%;
}

.el-select.el-select--medium{
  width: 100%;
}

.el-input-number.is-controls-right .el-input__inner{
  width: 100%;
  text-align: left;
}

.el-input-number.is-without-controls .el-input__inner{
  text-align: left;
}
</style>
<script>
import { multiList, fieldList, fetchPv, createArticle, updateArticle } from '@/api/userInfo'
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'

const calendarTypeOptions = [
  { key: 'CN', display_name: 'China' },
  { key: 'US', display_name: 'USA' },
  { key: 'JP', display_name: 'Japan' },
  { key: 'EU', display_name: 'Eurozone' }
]

// arr to obj, such as { CN : "China", US : "USA" }
const calendarTypeKeyValue = calendarTypeOptions.reduce((acc, cur) => {
  acc[cur.key] = cur.display_name
  return acc
}, {})

export default {
  name: 'UserTableEdit',
  components: { },
  directives: { waves },
  filters: {
    statusFilter(status) {
      const statusMap = {
        published: 'success',
        draft: 'info',
        deleted: 'danger'
      }
      return statusMap[status]
    },
    typeFilter(type) {
      return calendarTypeKeyValue[type]
    }
  },
  data() {
    return {
      pickerOptions: {
        disabledDate(time) {
          return time.getTime() === Date.now()
        },
        shortcuts: [{
          text: '今天',
          onClick(picker) {
            picker.$emit('pick', new Date())
          }
        }, {
          text: '昨天',
          onClick(picker) {
            const date = new Date()
            date.setTime(date.getTime() - 3600 * 1000 * 24)
            picker.$emit('pick', date)
          }
        }, {
          text: '一周前',
          onClick(picker) {
            const date = new Date()
            date.setTime(date.getTime() - 3600 * 1000 * 24 * 7)
            picker.$emit('pick', date)
          }
        }]
      },
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 20,
        importance: undefined,
        title: undefined,
        type: undefined,
        sort: '+id'
      },
      groups: [],
      importanceOptions: [1, 2, 3],
      calendarTypeOptions,
      sortOptions: [{ label: 'ID Ascending', key: '+id' }, { label: 'ID Descending', key: '-id' }],
      statusOptions: ['published', 'draft', 'deleted'],
      showReviewer: false,
      // temp: {
      //   id: undefined,
      //   importance: 1,
      //   remark: '',
      //   timestamp: new Date(),
      //   title: '',
      //   type: '',
      //   status: 'published'
      // },
      dict: {},
      temp: {
        id: undefined,
        // 岗位信息
        job: '', // 职位
        officeArea: '', // 办公区域
        employeeLevel: '', // 层级
        phoneNumber: '', // 手机号码
        workingAge: '', // 工龄
        positiveDate: '', // 转正日期
        commencementDate: '', // 入职日期
        superiorDepartment: '', // 中心
        department: '', // 部门
        jobNumber: '', // 工号
        employeeName: '', // 员工姓名
        // 身份证信息
        idNumber: '', // 身份证号
        birthDate: '', // 出生日期
        age: '', // 年龄
        gender: '', // 性别
        nation: '', // 民族
        idCardAddress: '', // 身份证地址
        validityCertificate: '', // 证件有效期
        // 个人信息
        nativePlace: '', // 籍贯
        nativePlaceType: '', // 籍贯类型
        maritalStatus: '', // 婚姻状况
        childrenSituation: '', // 子女情况
        firstTimeToWork: '', // 首次参加工作的时间（全日制毕业时间）
        employmentYears: '', // 就业年限
        presentAddress: '', // 现广州住址（详细到门牌号）
        politicStatus: '', // 政治面貌
        // 学历信息
        educationBackground: '', // 学历
        graduateInstitutions: '', // 毕业院校
        graduationDate: '', // 毕业时间
        specialty: '', // 专业
        // 合同信息
        contractingCompany: '', // 合同公司
        typeOfLaborContract: '', // 劳动合同类型
        firstCommencementDateOfContract: '', // 首次合同起始日期
        firstContractExpirationDate: '', // 首次合同到期日期
        nowCommencementDateOfContract: '', // 现合同起始日期
        nowContractExpirationDate: '', // 现合同到期日期
        modificationOfContract: '', // 合同变更
        renewalTimes: '', // 续签次数
        // 紧急联系人
        socialSecurityPurchaseTime: '', // 购买社保时间
        emergencyContactName: '', // 紧急联系人姓名
        contactRelationship: '', // 联系人关系
        contactNumber: '', // 联系人电话
        // 其他
        anniversaryReminder: '', // 周年提醒
        birthdayReminder: '', // 生日提醒
        onBoardingReminder: '', // 待入职提醒
        pendingResignationReminder: '', // 待离职提醒
        waitingToBeReminded: '', // 待转正提醒
        informationToBePerfected: '', // 信息待完善
        entry: '', // 入职
        dimission: '', // 离职
        status: 'published'
      },
      dialogFormVisible: false,
      dialogStatus: '',
      textMap: {
        update: 'Edit',
        create: 'Create'
      },
      dialogPvVisible: false,
      pvData: [],
      rules: {
        type: [{ required: true, message: 'type is required', trigger: 'change' }],
        timestamp: [{ type: 'date', required: true, message: 'timestamp is required', trigger: 'change' }],
        title: [{ required: true, message: 'title is required', trigger: 'blur' }]
      },
      downloadLoading: false
    }
  },
  created() {
    this.getMultiList()

    this.temp = Object.assign({}, this.$route.query) // copy obj
  },
  mounted() {
    this.getFields()
  },
  methods: {
    getMultiList() {
      this.listLoading = true
      multiList({ 'formName': 'user_edit' }).then(response => {
        this.dict = response.data
      })
    },
    getFields() {
      this.listLoading = true
      fieldList({ 'formName': 'user_edit' }).then(response => {
        this.groups = response.data
        console.log(this.groups)
      })
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleModifyStatus(row, status) {
      this.$message({
        message: '操作Success',
        type: 'success'
      })
      row.status = status
    },
    sortChange(data) {
      const { prop, order } = data
      if (prop === 'id') {
        this.sortByID(order)
      }
    },
    sortByID(order) {
      if (order === 'ascending') {
        this.listQuery.sort = '+id'
      } else {
        this.listQuery.sort = '-id'
      }
      this.handleFilter()
    },
    resetTemp() {
      this.temp = {
        id: undefined,
        importance: 1,
        remark: '',
        timestamp: new Date(),
        title: '',
        status: 'published',
        type: ''
      }
    },
    handleCreate() {
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    createData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.temp.id = parseInt(Math.random() * 100) + 1024 // mock a id
          this.temp.author = 'vue-element-admin'
          createArticle(this.temp).then(() => {
            this.list.unshift(this.temp)
            this.dialogFormVisible = false
            this.$notify({
              title: 'Success',
              message: 'Created Successfully',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleUpdate(row) {
      this.temp = Object.assign({}, row) // copy obj
      this.temp.timestamp = new Date(this.temp.timestamp)
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    updateData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          const tempData = Object.assign({}, this.temp)
          tempData.timestamp = +new Date(tempData.timestamp) // change Thu Nov 30 2017 16:41:05 GMT+0800 (CST) to 1512031311464
          updateArticle(tempData).then(() => {
            const index = this.list.findIndex(v => v.id === this.temp.id)
            this.list.splice(index, 1, this.temp)
            this.dialogFormVisible = false
            this.$notify({
              title: 'Success',
              message: 'Update Successfully',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleDelete(row, index) {
      this.$notify({
        title: 'Success',
        message: 'Delete Successfully',
        type: 'success',
        duration: 2000
      })
      this.list.splice(index, 1)
    },
    handleFetchPv(pv) {
      fetchPv(pv).then(response => {
        this.pvData = response.data.pvData
        this.dialogPvVisible = true
      })
    },
    handleDownload() {
      this.downloadLoading = true
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['timestamp', 'title', 'type', 'importance', 'status']
        const filterVal = ['timestamp', 'title', 'type', 'importance', 'status']
        const data = this.formatJson(filterVal)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: 'table-list'
        })
        this.downloadLoading = false
      })
    },
    formatJson(filterVal) {
      return this.list.map(v => filterVal.map(j => {
        if (j === 'timestamp') {
          return parseTime(v[j])
        } else {
          return v[j]
        }
      }))
    },
    getSortClass: function(key) {
      const sort = this.listQuery.sort
      return sort === `+${key}` ? 'ascending' : 'descending'
    }
  }
}
</script>
