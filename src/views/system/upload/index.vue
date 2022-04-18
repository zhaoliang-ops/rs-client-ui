<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryForm" size="small" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="数据名称" prop="productName">
        <el-input
          v-model="queryParams.productName"
          placeholder="请输入数据名称"
          clearable
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="数据类型" prop="productDescription">
        <el-input
          v-model="queryParams.productDescription"
          placeholder="请输入数据描述"
          clearable
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item label="拍摄日期" prop="createDate">
        <el-date-picker clearable
          v-model="queryParams.createDate"
          type="date"
          value-format="yyyy-MM-dd"
          placeholder="请选择拍摄日期">
        </el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
        <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-plus"
          size="mini"
          @click="handleAdd"
          v-hasPermi="['system:localimage:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="danger"
          plain
          icon="el-icon-delete"
          size="mini"
          :disabled="multiple"
          @click="handleDelete"
          v-hasPermi="['system:localimage:remove']"
        >删除</el-button>
      </el-col>
      <right-toolbar :showSearch.sync="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="localimageList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="数据名称" align="center" prop="productName" />
      <el-table-column label="数据格式" align="center" prop="dataFormat" >
          <template slot-scope="scope">
            <span v-if="scope.row.dataFormat === '028'">ZIP</span>  
            <span v-else>TGZ</span>
          </template>
      </el-table-column>
      <el-table-column label="当前状态" align="center" prop="promotionId" >
          <template slot-scope="scope">
            <span v-if="scope.row.promotionId === 3">上传中</span>  
            <span v-else>已完成</span>
          </template>
      </el-table-column>
      <el-table-column label="状态" align="center" prop="wrsRow" >
          <template slot-scope="scope">
              <el-switch
                    v-model="scope.row.wrsRow"
                    :active-value="0"
                    :inactive-value="1"
                    active-color="#02538C"
                    inactive-color="#B9B9B9"
                    @change="changeSwitch(scope.row)"/>
          </template>
      </el-table-column>
      <el-table-column
                        prop="wrsPath"
                        align="center"
                        label="上传进度">
          <template slot-scope="scope">
            <el-progress :text-inside="true" :stroke-width="18" :percentage="scope.row.wrsPath" status="success"></el-progress>
          </template>
      </el-table-column>
      <el-table-column label="操作时间" align="center" prop="createDate" width="180"/>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button
            size="mini"
            type="text"
            icon="el-icon-delete"
            @click="handleDelete(scope.row)"
            v-hasPermi="['system:localimage:remove']"
          >删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <pagination
      v-show="total>0"
      :total="total"
      :page.sync="queryParams.pageNum"
      :limit.sync="queryParams.pageSize"
      @pagination="getList"
    />
    <!-- 添加或修改本地资源管理对话框 -->
    <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="数据名称" prop="productName">
          <el-input v-model="form.productName" placeholder="请输入数据名称" />
        </el-form-item>
        <el-form-item label="数据编码" prop="catalogCode">
                  <el-cascader
                  :options="datainfos"
                  :show-all-levels="false"
                  v-model="form.catalogCode"
                ></el-cascader>
          <!-- <el-input v-model="form.productName" placeholder="选择数据编码" /> -->
        </el-form-item>
          <el-form-item label="文件类型" prop="dataFormat">
            <el-select v-model="form.dataFormat" placeholder="请选择文件格式">
              <el-option label="Shp" value="001"></el-option>
              <el-option label="TiFF" value="006"></el-option>
                    <el-option label="ZIP" value="028"></el-option>
              <el-option label="HDF4" value="005"></el-option>
                    <el-option label="GEOTIFF" value="007"></el-option>
              <el-option label="其它" value="029"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="拍摄时间" prop="createDate">
            <el-col>
              <el-date-picker type="date" placeholder="选择时间" v-model="form.createDate" style="width: 100%;"></el-date-picker>
            </el-col>
          </el-form-item>
          <el-form-item label="文件路径"  prop="filePath">
              <el-upload
                class="upload-demo"
                ref="upload"
                :action="uploadUrl"
                :file-list="fileList"
                :auto-upload="false">
                <el-button slot="trigger"  type="primary">选取文件</el-button>
              </el-upload>
          </el-form-item>
          <el-form-item label="文件校验"  prop="isCheck">
            <el-radio-group v-model="form.isCheck">
              <el-radio label="开启"></el-radio>
              <el-radio label="不开启"></el-radio>
            </el-radio-group>
          </el-form-item>
        <el-form-item label="数据描述" prop="productDescription">
          <el-input type="textarea" v-model="form.productDescription" placeholder="请输入数据描述" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <!-- <el-button style="margin-left: 10px;" size="small" type="success" @click="submitUpload">上传到服务器</el-button> -->
        <el-button type="primary" @click="submitUpload">开始上传</el-button>
        <el-button @click="cancel">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { listLocalimage, getLocalimage,getUploadLocalimageList, delLocalimage, addLocalimage, updateLocalimage,editP2PLocalimage } from "@/api/system/localimage";
import { listAlgorithm, getAlgorithm, delAlgorithm, addAlgorithm, updateAlgorithm } from "@/api/system/algorithm";
export default {
  name: "Localimage",
  data() {
    return {
      uploadUrl: process.env.VUE_APP_BASE_API + "/common/upload", // 上传的图片服务器地址
      datainfos:[],
      fileList:[],
      intAlgorithm: 0,
      sysAlgorithm: '',
      random: 0,
      percentage: 0, // 进度条的占比
      timer: null,
      // 遮罩层
      loading: true,
      // 选中数组
      ids: [],
      // 非单个禁用
      single: true,
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // 本地资源管理表格数据
      localimageList: [],
      // 弹出层标题
      title: "",
      // 是否显示弹出层
      open: false,
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        catalogCode: null,
        productName: null,
        productDescription: null,
        dataFormat: null,
        createDate: null,
        promotionId:0,
        imageId:null,
        diskId:0,
        downloadTime:'',
        wrsPath:0
      },
      // 表单参数
      form: {},
      // 表单校验
      rules: {
        productName: [
          { required: true, message: "数据名称不能为空", trigger: "blur" }
        ],
        dataFormat: [
          { required: true, message: "数据格式不能为空", trigger: "blur" }
        ],
        modifyDate: [
          { required: true, message: "拍摄时间不能为空", trigger: "blur" }
        ],
        isCheck:[
          { required: true, message: "是否自检不能为空", trigger: "blur" }
        ],
        catalogCode:[
          { required: true, message: "数据编码不能为空", trigger: "blur"}
        ]
      }
    };
  },
  created() {
    this.getList();
      // 定时器
    this.start()
  },
  methods: {
      clearTimer() {
      if(this.timer) {
        clearInterval(this.timer);
        this.timer = '';
      }
    },
        // 随机数
        get_random_v(size) {
          var min = size - 10
          var max = size + 10
          var result = Math.round(Math.random() * (max - min) + min)
          return result
        },
    // 每隔两秒增加上传进度
    start() {
      this.timer = setInterval(this.getProgress, 2000) // 注意: 第一个参数为方法名的时候不要加括号;
    },
    getProgress() {
     this.queryParams.pageNum = 1;
     this.getList();
     var downList = this.localimageList
     var downNum = 0;
     for(var i=0;i<downList.length;i++){
       if(downList[i].promotionId==3 && downList[i].wrsRow == 0){
          if(downList[i].wrsPath == 100){
            this.queryParams.imageId = downList[i].imageId
            this.queryParams.promotionId = 2
            editP2PLocalimage(this.queryParams).then(response => {
          })
         }else{
            downNum++
            this.random = this.get_random_v(1000)
            // 随机更新每次新增多少
            var filit = this.random / 100
            var percentage = downList[i].wrsPath + filit > 100 ? 100 : downList[i].wrsPath + filit
            this.queryParams.imageId = downList[i].imageId
            this.queryParams.promotionId = 3
            this.queryParams.wrsPath = parseFloat(percentage).toFixed(0) 
            editP2PLocalimage(this.queryParams).then(response => {
          })
         }
       }
     }
    //  console.log(downNum)
     // 无下载中任务，清除定时器
     if(downNum == 0){
         this.clearTimer();
     }
    },
    /** 查询本地资源管理列表 */
    getList() {
      this.loading = true;
      getUploadLocalimageList(this.queryParams).then(response => {
        this.localimageList = response.rows;
        this.total = response.total;
        this.loading = false;
      });
    },
    // 取消按钮
    cancel() {
      this.open = false;
      this.reset();
    },
    // 表单重置
    reset() {
      this.form = {
        imageId: null,
        catalogCode: null,
        promotionId: null,
        productName: null,
        productPrice: null,
        productDescription: null,
        satelliteId: null,
        sensorId: null,
        dataFormat: null,
        mapProjection: null,
        earthModel: null,
        spatialResolution: null,
        version: null,
        standardCode: null,
        imageCode: null,
        md5: null,
        diskId: null,
        serverId: null,
        wrsPath: null,
        wrsRow: null,
        thumbView: null,
        quickView: null,
        cloudPercent: null,
        filePath: null,
        dataSize: null,
        centerLat: null,
        centerLong: null,
        upperLeftLat: null,
        upperLeftLong: null,
        lowerRightLat: null,
        lowerRightLong: null,
        geom: null,
        dataUnitId: null,
        sourceProductId: null,
        area: null,
        sunElevation: null,
        sensingStartDate: null,
        sensingEndDate: null,
        datasetName: null,
        productType: null,
        productDate: null,
        productLevel: null,
        bands: null,
        upperRightLat: null,
        upperRightLong: null,
        lowerLeftLat: null,
        lowerLeftLong: null,
        recStationId: null,
        algorithm: null,
        temporalResolution: null,
        creator: null,
        createDate: null,
        modifier: null,
        modifyDate: null
      };
      this.resetForm("form");
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
      this.getProgress()
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.resetForm("queryForm");
      this.handleQuery();
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.imageId)
      this.single = selection.length!==1
      this.multiple = !selection.length
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset();
      this.datainfos = []
      var context = require('../../../static/dataCatlog2.json')
      this.datainfos = context
      this.open = true;
      this.title = "上传本地遥感数据资源";
    },
        /** 修改按钮操作 */
    changeSwitch(row) {
        this.clearTimer()
        updateLocalimage(row).then(response => {
          this.$modal.msgSuccess("更新状态成功");
          this.open = false;
          this.getList();
          this.start();
        });
      },
    /** 提交按钮 */
    submitForm() {
      this.$refs["form"].validate(valid => {
        if (valid) {
          if (this.form.imageId != null) {
            updateLocalimage(this.form).then(response => {
              this.$modal.msgSuccess("修改成功");
              this.open = false;
              this.getList();
            });
          } else {
            addLocalimage(this.form).then(response => {
              this.$modal.msgSuccess("上传任务新增成功");
              this.open = false;
              this.getList();
            });
          }
        }
      });
    },
    // 上传本地文件
    submitUpload(){
      this.form.promotionId = 3
      this.form.catalogCode = this.form.catalogCode[1]
      this.form.wrsPath = 0
      this.form.dataSize = this.get_random_v(856)
      console.log(this.form)
      addLocalimage(this.form).then(response => {
        this.$modal.msgSuccess("上传任务新增成功");
        this.open = false;
        this.getList();
      });
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      const imageIds = row.imageId || this.ids;
      this.$modal.confirm('是否确认删除本地资源管理编号为"' + imageIds + '"的数据项？').then(function() {
        return delLocalimage(imageIds);
      }).then(() => {
        this.getList();
        this.$modal.msgSuccess("删除成功");
      }).catch(() => {});
    },
    /** 导出按钮操作 */
    handleExport() {
      this.download('system/localimage/export', {
        ...this.queryParams
      }, `localimage_${new Date().getTime()}.xlsx`)
    }
  }
};
</script>
<style scoped>
.map {
  display: block;
  width: 100%;
}
#map {
  /* width: 100%; */
  height: 100%;
}
.el-header {
  background-color: #142336;
  color: rgb(240, 191, 30);
  text-align: center;
  line-height: 60px;
  
}
.el-form-item__label{
  color: rgb(28, 223, 191);
}
.el-tag {
  background-color: #18bc9c;
  padding: 0 10px;
  height: 32px;
  line-height: 30px;
  font-size: 12px;
  color: #ffffff;
  border-radius: 15px;
  -webkit-box-sizing: border-box;
  box-sizing: border-box;
  border: 1px solid #18bc9c;
  white-space: nowrap;
  margin-right: 10px;
  margin-bottom: 5px;
}
.el-tag .el-icon-close {
  border-radius: 50%;
  text-align: center;
  position: relative;
  cursor: pointer;
  font-size: 12px;
  height: 16px;
  width: 16px;
  line-height: 16px;
  top: -1px;
  right: -5px;
  color: #ffffff;
}
.el-tag .el-icon-close:hover {
  color: #ffffff;
  background-color: #18bc9c;
  font-weight: 800;
}

.el-tab-pane {
  flex-wrap: wrap;
  display: flex;
  margin-left: 20px;
  justify-content: flex-start;
  margin-right: 16px;
  margin-top: 5px;
}
.el-dialog__body {
  padding-top: 0px;
  padding-left: 15px;
  padding-right: 15px;
  padding-bottom: 10px;
  color: #606266;
  font-size: 14px;
}
.el-tabs--left.el-tabs--card .el-tabs__item.is-left:hover {
  color: #18bc9c;
}
.el-tabs--left.el-tabs--card .el-tabs__item.is-left {
  border-left: none;
  border: none;
  border-bottom: none;
  text-align: center;
  border-right: 1px solid #e4e7ed;
}
.el-tabs--left.el-tabs--card .el-tabs__nav {
  border-radius: 4px 0 0 4px;
  border-bottom: none;
  border-right: none;
}
.el-tabs--card > .el-tabs__header {
  border-bottom: none;
}
.el-tabs--card > .el-tabs__header .el-tabs__nav {
  border: none;
  background-color: #eeeeee;
}
.el-tabs--left .el-tabs__header.is-left {
  float: none;
  margin-bottom: 0;
  margin-right: 0px;
  width: 20%;
}
.el-dialog__header {
  padding: 15px 20px 15px;
  text-align: left;
  line-height: 1;
}
.el-dialog__headerbtn .el-dialog__close {
  color: #ffffff;
}
.el-dialog__headerbtn .el-dialog__close:hover {
  color: rgb(28, 203, 150);
}
.tabs {
  border: 1px solid #e4e7ed;
  display: flex;
}
.el-checkbox-group {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
}
.el-tabs--left.el-tabs--card .el-tabs__item.is-left.is-active {
  border-right-color: #fff;
  border-top: 1px solid #e4e7ed;
  border-bottom: 1px solid #e4e7ed;
  background-color: #ffffff;
  color: #18bc9c;
}
.topDialog{
  position: relative;
  margin: 0 auto 50px;
  border-radius: 8px;
  -webkit-box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  -webkit-box-sizing: border-box;
  box-sizing: border-box;
  width: 66%;
}
.el-tabs__content {
  width: 80%;
}
.el-dialog__title {
  font-size: 16px;
  color: #ffffff;
}
.el-dialog__headerbtn {
  position: absolute;
  top: 15px;
  right: 15px;
  padding: 0;
  background: 0 0;
  border: none;
  outline: 0;
  cursor: pointer;
  font-size: 16px;
  color: #ffffff;
}
.el-dialog__header {
  background: #1d202f;
  border-radius: 8px 8px 0 0;
  height: 3.25rem;
}
.dialog__header span {
  color: white;
}
.satitle {
  display: flex;
  justify-content: flex-start;
  line-height: 1;
  width: 100%;
}
.kongzhi {
  width: 65%;
}
.kongjian {
  padding: 10px;
  background-color: #ffffff;
  width: 45%;
}
.inputinfo {
  display: flex;
  line-height: 5;
  letter-spacing: 20px;
  margin-left: 40px;
}
.inputdiv {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  width: 100%;
}
</style>