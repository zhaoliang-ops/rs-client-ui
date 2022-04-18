<template>
<el-container>
  <el-header>
      <el-form :model="queryParams" ref="queryForm" size="small" :inline="true" v-show="showSearch">
      <el-form-item label="数据选择" prop="roleName" class="greenItem">
        <el-button type="info" style="width:240px;margin-top:15px" icon="el-icon-search" @click="showDataCate">请点击选择数据资源</el-button>
      </el-form-item>
      <el-form-item label="经纬度" prop="roleKey" class="greenItem">
          <el-button type="info" style="width:240px;margin-top:15px" icon="el-icon-search" @click="showWeftDialog">请点击选择输入</el-button>
      </el-form-item>
      <el-form-item label="行政区划" prop="status" class="greenItem">
        <el-cascader  :options="options" v-model="selectedOptions" @change="changeArea" style="line-height: 60px;"/>
      </el-form-item>
      <el-form-item label="时间范围" class="greenItem"> 
        <el-date-picker
          v-model="dateRange"
          style="width: 240px;top:14px"
          value-format="yyyy-MM-dd"
          type="daterange"
          range-separator="-"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
        ></el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button style="margin-top:15px" type="primary" icon="el-icon-search" size="medium" @click="handleQuery">搜索</el-button>
        <el-button style="margin-top:15px" icon="el-icon-refresh" size="medium" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>
  </el-header>
  <el-main>
    <div id='container' style="width:100%;height:760px">
        <baidu-map class="map"  mapType="BMAP_SATELLITE_MAP" :center="map.center" :zoom="map.zoom" :scroll-wheel-zoom="true" @ready="handler">
        </baidu-map>
    </div>
  <el-dialog
    width="58%"
    title="遥感数据选择"
    style="color:white"
    :visible.sync="dialogFormVisible"
    append-to-body
  >
    <el-tabs
      type="card"
      class="tabs"
      tab-position="left"
    >
      <el-tab-pane
        :label="item.appliedfieldname"
        v-for="(item, index) in datainfos"
        :key="index"
        :name="item.appliedfieldname"
      >
        <div
          class="kongjian"
          v-for="(itemi,indexi) in item.children"
          :key="indexi"
        >
          <div class="satitle">
            <div
              style="padding-right: 65px;"
              class="kongzhi"
            >
              <el-checkbox
                @change="handleCheckAllChange(itemi.caralog_code)"
              >{{itemi.name}}</el-checkbox>
            </div>
          </div>
        </div>
        <div>
        <el-table :data="gridData" stripe border height="400" style="width: 800px" :key="Math.random()" @selection-change="handleSelectionChange">
          <el-table-column type="selection" width="55px" align="center" />
          <el-table-column prop="imageId" v-if="false"></el-table-column>
          <el-table-column prop="productName" align="center" label="数据名称" width="700px"></el-table-column>
        </el-table>
            <pagination
              v-show="total>0"
              :total="total"
              :page.sync="queryParams.pageNum"
              :limit.sync="queryParams.pageSize"
              @pagination="getDataList"
            />
      </div>
      </el-tab-pane>
    </el-tabs>
    <div style="margin-top: 20px;text-align: center;">
      <el-button
        class="buttongtey"
        type="primary"
        @click="handleUpdate"
      >开始下载</el-button>
      <el-button
        style="margin-left: 60px;"
        class="buttongteys"
        @click="dialogFormVisible = false"
      >取 消</el-button>
    </div>
  </el-dialog>
  <!--经纬度上传组件 -->
  <el-dialog
    title="经纬度输入"
    style="color:white"
    :visible.sync="dialogFormWeftVisible"
    append-to-body
    custom-class="mainDialog"
  >
    <div class="samlltitle">经纬度</div>
    <div class="infobody">
      <div class="inputdiv">
        <div class="inputinfo">左 <el-input
            v-model="leftvalue"
            placeholder="73 - 136"
            max='136'
            min="73"
          ></el-input>
        </div>
        <div class="inputinfo">右<el-input
            v-model="rightvalue"
            placeholder="73 - 136"
          ></el-input>
        </div>
      </div>
      <div class="inputdiv">
        <div class="inputinfo">上<el-input
            v-model="topvalue"
            placeholder="4 - 54"
          ></el-input>
        </div>
        <div class="inputinfo">下<el-input
            v-model="downvalue"
            placeholder="4 - 54"
          ></el-input>
        </div>
      </div>
    </div>
    <div style="margin-top: 20px;text-align: center;">
      <el-button
        class="buttongtey"
        type="primary"
        @click="dialogFormWeftVisible = false"
      >确 定</el-button>
      <el-button
        style="margin-left: 60px;"
        class="buttongteys"
        @click="dialogFormWeftVisible = false"
      >取 消</el-button>
    </div>
  </el-dialog>
  </el-main>
</el-container>
</template>
<script type="text/javascript">
import { provinceAndCityDataPlus, CodeToText } from 'element-china-area-data'// 2.省市带‘全部’的二级联动
import { listLocalimage,editP2PLocalimage,listAllP2PImage} from "@/api/system/localimage";
import { listAlgorithm, getAlgorithm, delAlgorithm, addAlgorithm, updateAlgorithm } from "@/api/system/algorithm";
import {BaiduMap,BmNavigation,BmView,BmGeolocation,BmCityList} from 'vue-baidu-map'
    export default{
        components: {
            BaiduMap,
            BmNavigation,
            BmView,
            BmGeolocation,
            BmCityList,
        },
        data(){
            return {  
              // 选中数组
              ids: [],
              intAlgorithm: 0,
              sysAlgorithm: '无',
              regionName:'黑龙江',
              random: 0,
                  // 总条数
              total: 0,
              percentage: 0, // 进度条的占比
              timer: '',
              gridData: [],
              district: null,
              map: '',
              BMap:null,
              bdary:null,
              polygons: [],
              options: provinceAndCityDataPlus, // 2.省市带‘全部’的二级联动
              selectedOptions: [],
                    // 查询参数
              queryParams: {
                pageNum: 1,
                pageSize: 10,
                roleName: undefined,
                roleKey: undefined,
                status: undefined,
                catalogCode: null,
                productName: null,
                productDescription: null,
                dataFormat: null,
                createDate: null,
                promotionId:0,
                imageId:null,
                algorithm:''
              },
                    // 表单参数
              form: {},
              showSearch: true,
              dialogFormVisible:false,
              dialogFormWeftVisible : false,
              leftvalue: '',
              rightvalue: '',
              topvalue: '',
              wradio: '2',
              downvalue: '',
              dateRange:[],
              datainfos:[],
              boundariesValue:[],
              map:{
                center: {lng: 105.929912,lat: 39.683191},
                // 105.3, 39.9
                zoom: 5,
                show: true,
                dragging: true,
                
                // mapType:BMAP_HYBRID_MAP
            },
          }
        },
        created(){
          // this.initMap()
          this.getAlgName()
        },
        destroyed(){
        },
        methods: {
        changeArea () {
            // console.log(this.selectedOptions)
                var loc = this.selectedOptions[0]
                this.keyword = loc
                var nameZh = CodeToText[loc]
                // 输出区域码所对应的属性值即中文地址
                this.drawmap(nameZh)
            },
          // // 行政区区域划分
          // drawBounds(newValue) {
          //   this.drawmap(newValue)
          // },
          handler ({BMap, map}) {
            this.map = map
            this.BMap = BMap
            // console.log(1+this.BMap)
            this.bdary = new BMap.Boundary();
          },
          // 行政区区域划分
          drawmap(area){
            var BMap = this.BMap
            var map = this.map
            map.clearOverlays();

            this.bdary.get(area, function (rs) {
                //轮廓的总数 —— 有的区由多个封闭区域组成
                let count = rs.boundaries.length;
                if (!count) {
                    console.log('未能获取到当前输入的行政区域');
                    return;
                }
                for (let i = 0; i < count; i++) {
                    //创建多边形覆盖物
                    let ply =new BMap.Polygon(rs.boundaries[i],
                        {
                            strokeWeight: 2,
                            strokeColor: "red",
                            fillOpacity: 0.5,
                            strokeStyle: "dashed",
                        }
                    );
                    // 给覆盖物添加点击事件
                    ply.addEventListener('click', function () {
                        // 改变自己的透明度
                        // this.setFillOpacity(0.6)
                    });
                    //添加覆盖物
                    map.addOverlay(ply);
                }
            })

          },
          getAlgName(){
            var params = {
              pageNum: 1,
              pageSize: 1,
              isActive: 1
            }
            listAlgorithm(params).then(response => {
              // console.log(response.rows)
              if(response.rows.length != 0){
                this.intAlgorithm = response.rows[0].id
                this.sysAlgorithm = response.rows[0].nameZh
              }
              
            });
          },
          getAlgorithmNameZh(context) {
          var result = '无'
          if (context === 1) {
            result = 'MGA-PSO算法'
          } else if (context === 2) {
            result = 'GA-PSO算法'
          } else if (context === 3) {
            result = 'GA算法'
          } else if (context === 4) {
            result = 'PSO算法'
          }
          return result
        },
          // 多选框选中数据
          handleSelectionChange(selection) {
            this.ids = selection.map(item => item.imageId)
            this.single = selection.length!==1
            this.multiple = !selection.length
          },
          // 开始下载
          handleUpdate() {
            var algName = this.sysAlgorithm

                // console.log('算法名称—'+algName)
            
            if (this.ids.length!=0) {
                  for(var i =0;i<this.ids.length;i++){
                      this.queryParams.imageId = this.ids[i]
                      this.queryParams.promotionId = 1
                      this.queryParams.algorithm = algName
                    editP2PLocalimage(this.queryParams).then(response => {
                    })
                  }
                    this.getDataList();
                    this.$modal.msgSuccess("下载任务提交成功");
                
            }
            // console.log("开始下载："+this.ids)
            this.dialogFormVisible = false
          },
          handleCheckAllChange(value){
              this.queryParams.catalogCode = value
              this.getDataList()
          },
          showDataCate(){
              this.datainfos = []
              var context = require('../static/dataCatlog.json')
              // console.log(context)
              this.datainfos = context
              this.gridData = []
              this.dialogFormVisible = true
          },
          showWeftDialog(){
                this.dialogFormWeftVisible = true
          },
        /** 查询本地资源管理列表 */
        getDataList() {
          this.loading = true;
          listAllP2PImage(this.queryParams).then(response => {
            this.gridData = response.rows;
            this.total = response.total;
            this.loading = false;
          });
        },
        /** 搜索按钮操作 */
        handleQuery() {
          this.queryParams.pageNum = 1;
          this.getList();
        },
        /** 重置按钮操作 */
        resetQuery() {
          this.dateRange = [];
          this.resetForm("queryForm");
          this.handleQuery();
        },
},
}
</script>
<style lang="scss" type="text/css" rel="stylesheet/scss">
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
.map {
    width: 100%;
    height: 750px;
}
</style>
