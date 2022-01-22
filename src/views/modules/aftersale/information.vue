<template>
  <div class="laboratory-standards-list">
    <Row>
      <Col span="12" type="flex">
        <DatePicker type="date" placeholder="操作日期过滤" style="width: 200px"></DatePicker>
        <Select v-model="search.DATETYPE" clearable filterable placeholder="搜索全部"
                style="width:90px">
          <Option v-for="(item,index) in searchDateTypes " :key="item.value" :value="item.value">{{
              item.label
            }}
          </Option>
        </Select>
        <Input v-model="seach"  clearable style="width: 200px" />
        <Button icon="md-search" type="primary" @click="refreshData">搜索</Button>
        <Button @click="reset">重置</Button>
      </Col>
    </Row>
    <Row  style="margin-top: 10px":gutter="10">
      <Col>
        <Button  icon="md-add" @click="toAddComponent">添加</Button>
      </Col>
      <Col>
        <Button  icon="md-create" @click="editSelectRow">修改</Button>
      </Col>
      <Col>
      <Poptip
        confirm
        content="content"
        title="确定要删除选择的项吗?"
        transfer
        @on-ok="deleteByIds">
        <Button  icon="ios-trash">删除</Button>
      </Poptip>
    </Col>
      <Col>
        <Button  icon="md-create" @click="select">查看</Button>
      </Col>
      <Col>
        <Button  icon="ios-copy" @click="copyData">复制
        </Button>
      </Col>
      <Col>
        <Button   icon="md-return-left" @click="returnPiece">返件
        </Button>
      </Col>
      <Col>
        <Dropdown v-if="permsBtn.EXPORT_AFTERSALE_INFORMATION">
          <Button>
            导出列表
            <Icon type="ios-arrow-down"/>
          </Button>
          <DropdownMenu slot="list">
            <DropdownItem @click.native="exportList()">当前页</DropdownItem>
            <DropdownItem @click.native="exportList(1,pager.total)">全部数据</DropdownItem>
          </DropdownMenu>
        </Dropdown>
      </Col>
      <Col>
        <Dropdown v-if="permsBtn.IMPORT_AFTERSALE_INFORMATION">
          <Button>基础信息导入及更新
            <Icon type="ios-arrow-down"/>
          </Button>
          <DropdownMenu slot="list">
            <DropdownItem @click.native="downLoadFile1">下载模板</DropdownItem>
            <DropdownItem>
              <Upload :action="`${API_HOST}/afterSale/batch/import`"
                      :format="['xls']"
                      :headers="{'x-access-token':$SessionStorage.get('token')}"
                      :on-format-error="handleFormatError"
                      :on-progress="progress"
                      :on-success="uploadOk"
                      :show-upload-list="false"
              >
                导入数据
              </Upload>
            </DropdownItem>
          </DropdownMenu>
        </Dropdown>
      </Col>
    </Row>
    <!-- 列表 -->
    <Table ref="selection"  :columns="table.selectColumns" :data="table.data" :loading="false"
           border style="height: calc(100vh - 285px);width: calc(100vw - 254px)" @on-row-dblclick="dblClick"
    >
    </Table>
    <div style="padding: 20px 0 20px 0px;">
      <Page id="pageId" :current="pager.current" :page-size="pager.limit"
            :page-size-opts="[10,20,50,100]"
            :total="pager.total"
            show-elevator show-sizer
            show-total size="small" @on-change="pageChange" @on-page-size-change="pageSizeChange"></Page>
    </div>
  </div>

</template>

<script>
import {Col,Row} from "iview";
export default {
  name: "after-information-list",
  components: {
    Col,
    Row
  },
  created() {
    //查询列表
    this.refreshData();
    //查询权限
    let permsAll = this.$store.state.permissions;

  },
  mounted() {
    this.selection = this.$refs.selection;

  },
  data() {
    return {
      searh:'',
      afterSeach:'',
      errorMsg: {
        //成功条数
        successCount: 0,
        //失败条数
        failCount: 0,
        show: false,
        columns: [
          {
            width: 130,
            title: '错误行号',
            key: 'errorRow',
            align: 'center'
          },
          {
            title: '错误信息',
            key: 'errorMessage',
            align: 'left'
          }
        ],
        dataList: []
      },
      showCustomColumns: false,
      API_HOST: process.env.API_HOST,
      deleteDisable: true,
      updateDisable: true,
      headers: {"headers": {"Content-Type": "application/json; charset=UTF-8"}},
      disabled: true,
      search: {
        ISBATCH:null,
        //是否超过3年
        THREEYEAR:null,
        //是否超过6公里
        SIXKILOMETRE:null,
        //是否超过10公里
        TENKILOMETRE:null,
        //是否申诉
        COMPLAIN:null,
        //状态
        INFOSTATUS: null,
        //搜索时间类型
        DATETYPE: null,
        //操作过滤日期
        operationDate: [],
        //搜索全部还是只按车系,车型搜索
        key: null,
        //搜索的关键字
        value: ''
      },
      isexceedList:[
        {label: '是', value: "是"},
        {label: '否', value: "否"}
      ],isComplainList:[
        {label: '是', value: 1},
        {label: '否', value: 2}
      ],
      statusList: [
        {label: '正常', value: 0},
        {label: '返件中', value: 1},
        {label: '待认领', value: 2},
        {label: '分析中', value: 3},
        {label: '待处置', value: 4},
        {label: '处置完成', value: 5},
        {label: '已结题', value: 6}
      ],
      searchTypes: [ /*搜索下拉框 数据*/
        {label: '车系', value: 'COMBINE_CAR_SERIES'},
        {label: '车型', value: 'MODEL_CODE'},
        {label: '最小符合范围', value: 'MINIMUM_'},
        {label: 'IPTV36考核月', value: 'IPTV_THIRTY_SIX'},
        {label: 'IPTV24考核月', value: 'IPTV_TWENTY_FOUR'},
        {label: 'IPTV12考核月', value: 'IPTV_TWELVE'},
        {label: 'IPTV6考核月', value: 'IPTV_SIX'},
        {label: 'IPTV3考核月', value: 'IPTV_THREE'},
        {label: '底盘号', value: 'VIN'},
        {label: '维修次数', value: 'MAINTAIN'},
        {label: '行驶里程', value: 'MILEAGE'},
        {label: '里程区间', value: 'MILEAGE_SECTION'},
        {label: '故障描述', value: 'BREAKDOWN_DESCRIBE'},
        {label: '服务站号', value: 'STORE_CODE'},
        {label: '服务站名称', value: 'STORE_NAME'},
        {label: '维修省份', value: 'STORE_PROVINCE'},
        {label: '索赔单号', value: 'APPLICATION_NUMBER'},
        {label: '发动机编号', value: 'ENGINE_CODE'},
        {label: '故障分类', value: 'FAULT_TYPE'},
        {label: '故障备注', value: 'FAULT_REMARK'},
        {label: '材料号', value: 'MATERIAL_NUMBER'},
        {label: '材料名称', value: 'MATERIAL_NAME'},
        {label: '大总成', value: 'BIG_ASSEMBLY'},
        {label: '总成分类', value: 'ASSEMBLY_TYPE'},
        {label: '材料分类', value: 'MATERIAL_TYPE'},
        {label: '索赔承担方', value: 'CLAIM_BEARER'},
        {label: '合计', value: 'SUPPLIER_CLAIM_AMOUNT'},
        {label: '单位', value: 'SUPPLIER_FULL_NAME'},
        {label: '提前返件费', value: 'COST_ADVANCED'},
        {label: '计算', value: 'CAICULTE'},
        {label: '问题描述', value: 'PROBLEM_DESCRIPTION'},
        {label: '申诉指标', value: 'ISCOMPLAIN_INDICATOR'},
        {label: '故障件代码', value: 'FAULTY_PARTS_CODE'},
        {label: '工厂代码', value: 'FACTORY_DEMO'},
        {label: '厂牌', value: 'BRAND'},
        {label: '索赔类别', value: 'CLAIM_CATEGORY'},
      ],
      searchDateTypes: [ /*搜索下拉框 数据*/
        {label: '生产日期', value: 'PRODUCED_DATE'},
        {label: '购车日期', value: 'GET_LICENSE_DATE'},
        {label: '修理日期', value: 'REPAIR_DATE'},
        {label: '上传日期', value: 'UPLOAD_DATE'},
        {label: '审件日期', value: 'AUDIT_PARTS_DATE'},
      ],
      //列表参数
      table: {
        loading: true,
        data: [],
        selectColumns: [
          {
            type: 'selection',
            width: 50,
            align: 'center',
          },
          {
            title: 'IPTV最小符合范围',
            key: 'minimum',
            width: 80,
            select: true,
            resizable: true,
          }, {
            title: 'IPTV36考核月',
            slot: 'iptvThirtySix',
            width: 70,
            select: true,
            resizable: true,
          }, {
            title: 'IPTV24考核月',
            select: true,
            slot: 'iptvTwentyFour',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV12考核月',
            select: true,
            slot: 'iptvTwelve',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV6考核月',
            select: true,
            slot: 'iptvSix',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV3考核月',
            select: true,
            slot: 'iptvThree',
            width: 70,
            resizable: true,
          },
          {
            title: '车系',
            select: true,
            key: 'combineCarSeries',
            width: 60,
            resizable: true,
          },
          {
            title: '车型',
            select: true,
            key: 'modelCode',
            width: 100,
            resizable: true,
          },
          {
            title: '底盘号',
            select: true,
            key: 'vin',
            width: 130,
            resizable: true,
          },
          {
            title: '维修次数',
            select: true,
            key: 'maintain',
            width: 100,
            resizable: true,
          },
          {
            title: '生产日期',
            select: true,
            slot: 'producedDate',
            width: 100,
            resizable: true,
          },
          {
            title: '购车日期',
            select: true,
            slot: 'getLicenseDate',
            width: 100,
            resizable: true,
          },
          {
            title: '维修日期',
            select: true,
            slot: 'repairDate',
            width: 100,
            resizable: true,
          }, {
            title: '上传日期',
            select: true,
            slot: 'uploadDate',
            width: 100,
            resizable: true,
          },
          {
            title: '跟踪状态',
            select: true,
            slot: 'traceStatus',
            width: 100,
            resizable: true,
          }, {
            title: '行驶里程',
            key: 'mileage',
            width: 100,
            resizable: true,
          }, {
            title: '里程区间',
            key: 'mileageSection',
            width: 100,
            resizable: true,
          },
          {
            title: '故障描述',
            key: 'breakdownDescribe',
            width: 100,
            resizable: true,
          }, {
            title: '服务站号',
            key: 'storeCode',
            width: 100,
            resizable: true,
          }, {
            title: '服务站名称',
            key: 'storeName',
            width: 100,
            resizable: true,
          }, {
            title: '维修省份',
            key: 'storeProvince',
            width: 100,
            resizable: true,
          }, {
            title: '索赔单号',
            key: 'applicationNumber',
            width: 100,
            resizable: true,
          }, {
            title: '发动机编号',
            key: 'engineCode',
            width: 100,
            resizable: true,
          }, {
            title: '故障分类',
            key: 'faultType',
            width: 100,
            resizable: true,
          }, {
            title: '故障备注',
            key: 'faultRemark',
            width: 100,
            resizable: true,
          }, {
            title: '材料号',
            key: 'materialNumber',
            width: 100,
            resizable: true,
          }, {
            title: '材料名称',
            key: 'materialName',
            width: 100,
            resizable: true,
          }, {
            title: '大总成',
            key: 'bigAssembly',
            width: 100,
            resizable: true,
          }, {
            title: '总成分类',
            key: 'assemblyType',
            width: 100,
            resizable: true,
          }, {
            title: '材料分类',
            key: 'materialType',
            width: 100,
            resizable: true,
          }, {
            title: '索赔承担方',
            key: 'claimBearer',
            width: 100,
            resizable: true,
          }, {
            title: '台次',
            key: 'were',
            width: 100,
            resizable: true,
          }, {
            title: '工时费',
            key: 'supplierWorkingHoursFee',
            width: 100,
            resizable: true,
          }, {
            title: '材料费',
            key: 'supplierMaterialFee',
            width: 100,
            resizable: true,
          }, {
            title: '管理费',
            key: 'supplierManagementFee',
            width: 100,
            resizable: true,
          }, {
            title: '辅料费',
            key: 'supplierSupplementaryFee',
            width: 100,
            resizable: true,
          }, {
            title: '外出救援费',
            key: 'supplierServiceFee',
            width: 100,
            resizable: true,
          }, {
            title: '运费',
            key: 'supplierTransportationFee',
            width: 100,
            resizable: true,
          }, {
            title: '合计',
            key: 'supplierClaimAmount',
            width: 100,
            resizable: true,
          }, {
            title: '提前返件费',
            key: 'costAdvanced',
            width: 100,
            resizable: true,
          }, {
            title: '计算',
            key: 'caiculte',
            width: 100,
            resizable: true,
          }, {
            title: '结算编号',
            key: 'supPlantSettlementNo',
            width: 100,
            resizable: true,
          }, {
            title: '单位',
            key: 'supplierFullName',
            width: 100,
            resizable: true,
          }, {
            title: '问题描述',
            key: 'problemDescription',
            width: 100,
            resizable: true,
          }, {
            title: '故障部位代码',
            key: 'faultPositionCode',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出3年',
            key: 'overstepThreeyear',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出6万公里',
            key: 'overstepSixKilometers',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出10万公里',
            key: 'overstepTenKilometers',
            width: 100,
            resizable: true,
          }, {
            title: '是否返件',
            key: 'isreturn',
            width: 100,
            resizable: true,
          },{
            title: '是否申诉',
            key: 'iscomplain',
            width: 100,
            resizable: true,
          }, {
            title: '申诉指标',
            key: 'iscomplainIndicator',
            width: 100,
            resizable: true,
          }, {
            title: '申诉理由',
            key: 'iscomplainReason',
            width: 100,
            resizable: true,
          }, {
            title: '原因分析',
            key: 'analyzeReason',
            width: 100,
            resizable: true,
          }, {
            title: '处理结果',
            key: 'treatmentResult',
            width: 100,
            resizable: true,
          }, {
            title: '故障件代码',
            key: 'faultyPartsCode',
            width: 100,
            resizable: true,
          }, {
            title: '工厂代码',
            key: 'factoryDemo',
            width: 100,
            resizable: true,
          }, {
            title: '厂牌',
            key: 'brand',
            width: 100,
            resizable: true,
          }, {
            title: '索赔类别',
            key: 'claimCategory',
            width: 100,
            resizable: true,
          },
          {
            title: '是否批量',
            key: 'isBatch',
            width: 100,
            resizable: true,
          }

        ],
        columns: [
          {
            type: 'selection',
            width: 50,
            align: 'center',
          },
          {
            title: 'IPTV最小符合范围',
            key: 'minimum',
            width: 80,
            select: true,
            resizable: true,
          }, {
            title: 'IPTV36考核月',
            slot: 'iptvThirtySix',
            width: 70,
            select: true,
            resizable: true,
          }, {
            title: 'IPTV24考核月',
            select: true,
            slot: 'iptvTwentyFour',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV12考核月',
            select: true,
            slot: 'iptvTwelve',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV6考核月',
            select: true,
            slot: 'iptvSix',
            width: 70,
            resizable: true,
          }, {
            title: 'IPTV3考核月',
            select: true,
            slot: 'iptvThree',
            width: 70,
            resizable: true,
          },
          {
            title: '车系',
            select: true,
            key: 'combineCarSeries',
            width: 60,
            resizable: true,
          },
          {
            title: '车型',
            select: true,
            key: 'modelCode',
            width: 100,
            resizable: true,
          },
          {
            title: '底盘号',
            select: true,
            key: 'vin',
            width: 130,
            resizable: true,
          },
          {
            title: '维修次数',
            select: true,
            key: 'maintain',
            width: 100,
            resizable: true,
          },
          {
            title: '生产日期',
            select: true,
            slot: 'producedDate',
            width: 100,
            resizable: true,
          },
          {
            title: '购车日期',
            select: true,
            slot: 'getLicenseDate',
            width: 100,
            resizable: true,
          },
          {
            title: '维修日期',
            select: true,
            slot: 'repairDate',
            width: 100,
            resizable: true,
          }, {
            title: '上传日期',
            select: true,
            slot: 'uploadDate',
            width: 100,
            resizable: true,
          },
          {
            title: '跟踪状态',
            select: true,
            slot: 'traceStatus',
            width: 100,
            resizable: true,
          }, {
            title: '行驶里程',
            key: 'mileage',
            width: 100,
            resizable: true,
          }, {
            title: '里程区间',
            key: 'mileageSection',
            width: 100,
            resizable: true,
          },
          {
            title: '故障描述',
            key: 'breakdownDescribe',
            width: 100,
            resizable: true,
          }, {
            title: '服务站号',
            key: 'storeCode',
            width: 100,
            resizable: true,
          }, {
            title: '服务站名称',
            key: 'storeName',
            width: 100,
            resizable: true,
          }, {
            title: '维修省份',
            key: 'storeProvince',
            width: 100,
            resizable: true,
          }, {
            title: '索赔单号',
            key: 'applicationNumber',
            width: 100,
            resizable: true,
          }, {
            title: '发动机编号',
            key: 'engineCode',
            width: 100,
            resizable: true,
          }, {
            title: '故障分类',
            key: 'faultType',
            width: 100,
            resizable: true,
          }, {
            title: '故障备注',
            key: 'faultRemark',
            width: 100,
            resizable: true,
          }, {
            title: '材料号',
            key: 'materialNumber',
            width: 100,
            resizable: true,
          }, {
            title: '材料名称',
            key: 'materialName',
            width: 100,
            resizable: true,
          }, {
            title: '大总成',
            key: 'bigAssembly',
            width: 100,
            resizable: true,
          }, {
            title: '总成分类',
            key: 'assemblyType',
            width: 100,
            resizable: true,
          }, {
            title: '材料分类',
            key: 'materialType',
            width: 100,
            resizable: true,
          }, {
            title: '索赔承担方',
            key: 'claimBearer',
            width: 100,
            resizable: true,
          }, {
            title: '台次',
            key: 'were',
            width: 100,
            resizable: true,
          }, {
            title: '工时费',
            key: 'supplierWorkingHoursFee',
            width: 100,
            resizable: true,
          }, {
            title: '材料费',
            key: 'supplierMaterialFee',
            width: 100,
            resizable: true,
          }, {
            title: '管理费',
            key: 'supplierManagementFee',
            width: 100,
            resizable: true,
          }, {
            title: '辅料费',
            key: 'supplierSupplementaryFee',
            width: 100,
            resizable: true,
          }, {
            title: '外出救援费',
            key: 'supplierServiceFee',
            width: 100,
            resizable: true,
          }, {
            title: '运费',
            key: 'supplierTransportationFee',
            width: 100,
            resizable: true,
          }, {
            title: '合计',
            key: 'supplierClaimAmount',
            width: 100,
            resizable: true,
          }, {
            title: '提前返件费',
            key: 'costAdvanced',
            width: 100,
            resizable: true,
          }, {
            title: '计算',
            key: 'caiculte',
            width: 100,
            resizable: true,
          }, {
            title: '结算编号',
            key: 'supPlantSettlementNo',
            width: 100,
            resizable: true,
          }, {
            title: '单位',
            key: 'supplierFullName',
            width: 100,
            resizable: true,
          }, {
            title: '问题描述',
            key: 'problemDescription',
            width: 100,
            resizable: true,
          }, {
            title: '故障部位代码',
            key: 'faultPositionCode',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出3年',
            key: 'overstepThreeyear',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出6万公里',
            key: 'overstepSixKilometers',
            width: 100,
            resizable: true,
          }, {
            title: '是否超出10万公里',
            key: 'overstepTenKilometers',
            width: 100,
            resizable: true,
          }, {
            title: '是否返件',
            key: 'isreturn',
            width: 100,
            resizable: true,
          },{
            title: '是否申诉',
            key: 'iscomplain',
            width: 100,
            resizable: true,
          }, {
            title: '申诉指标',
            key: 'iscomplainIndicator',
            width: 100,
            resizable: true,
          }, {
            title: '申诉理由',
            key: 'iscomplainReason',
            width: 100,
            resizable: true,
          }, {
            title: '原因分析',
            key: 'analyzeReason',
            width: 100,
            resizable: true,
          }, {
            title: '处理结果',
            key: 'treatmentResult',
            width: 100,
            resizable: true,
          }, {
            title: '故障件代码',
            key: 'faultyPartsCode',
            width: 100,
            resizable: true,
          }, {
            title: '工厂代码',
            key: 'factoryDemo',
            width: 100,
            resizable: true,
          }, {
            title: '厂牌',
            key: 'brand',
            width: 100,
            resizable: true,
          }, {
            title: '索赔类别',
            key: 'claimCategory',
            width: 100,
            resizable: true,
          },
          {
            title: '是否批量',
            key: 'isBatch',
            width: 100,
            resizable: true,
          },


        ]
      },
      pager: {
        total: 0,
        limit: 10,
        current: 1,
      },
      selection: null,
      permsCur: [
        "QUERY_AFTERSALE_INFORMATION",
        "INSERT_AFTERSALE_INFORMATION",
        "DELETE_AFTERSALE_INFORMATION",
        "UPDATE_AFTERSALE_INFORMATION",
        "COPY_AFTERSALE_INFORMATION",
        "RETURN_AFTERSALE_INFORMATION",
        "CLAIM_AFTERSALE_INFORMATION",
        "APPEAL_AFTERSALE_INFORMATION",
        "SYNCH_AFTERSALE_INFORMATION",
        "EXPORT_AFTERSALE_INFORMATION",
        "IMPORT_AFTERSALE_INFORMATION",
        "RETURNFEE_AFTERSALE_INFORMATION",
      ],
      permsBtn: {
        "QUERY_AFTERSALE_INFORMATION": false,
        "INSERT_AFTERSALE_INFORMATION": false,
        "DELETE_AFTERSALE_INFORMATION": false,
        "UPDATE_AFTERSALE_INFORMATION": false,
        "COPY_AFTERSALE_INFORMATION": false,
        "RETURN_AFTERSALE_INFORMATION": false,
        "CLAIM_AFTERSALE_INFORMATION": false,
        "APPEAL_AFTERSALE_INFORMATION": false,
        "SYNCH_AFTERSALE_INFORMATION": false,
        "EXPORT_AFTERSALE_INFORMATION": false,
        "IMPORT_AFTERSALE_INFORMATION": false,
        "RETURNFEE_AFTERSALE_INFORMATION": false,
      }
    }
  },
  computed: {

    delDisable() { //禁用删除
      if (this.selection) {
        return !(this.selection.getSelection().length > 0);
      }
      return true;
    },
    set_copy_Disable() { //禁用修改
      if (this.selection) {
        //计算属性在mounted之前就会执行 一开始this.selection为null undefined
        return !(this.selection.getSelection().length === 1)
      }
      return true;
    },
    isTimeOut() {
      return function (time) {
        if (time <= new Date().getTime()) {
          return {color: 'red'}
        }
      }
    },
    labRefresh() {
      return this.$store.state.labRefresh;
    }
  },
  methods: {
    appeal() {
      // console.log('申诉')
    },
    settleALawsuit() {
      // console.log('结案')
    },
    downLoadFile1() {
      this.$iqis.download("/afterSale/download/afterSaleinfo", {}, "get", () => {
      })

    }, downLoadFile2() {
      this.$iqis.download("/afterSale/download/returnCose", {}, "get", () => {
      })

    },
    handleFormatError(file) {
      this.$Notice.warning({
        title: '文件格式不正确',
        desc: '文件格式' + file.name + '不正确,请选择xls文件'
      });
    },
    cancel() {
      this.errorMsg.data = []
      this.errorMsg.show = false
    },
    uploadOk(resp, file, fileList) {
      //成功条数
      this.errorMsg.successCount = resp.data["affectRows"];
      //失败条数
      this.errorMsg.failCount = resp.data["errorTotal"];
      //失败信息
      this.errorMsg.dataList = resp.data["errorMesList"];
      this.errorMsg.show = true;

      this.table.loading = false;
      //刷新表格
      this.refreshData();

    },
    exportList(pageNum, pageSize = this.pager.limit) {
      //方法参数2:pageSize=this.pager.limit 如果方法参数2为空没有传值过来,就是用pageSize=this.pager.limit赋值
      //如果方法参数2传了值过来,就使用传过来的值给pageSize赋值
      let queryParams = this.getQueryParams()
      let currentPage = queryParams.page
      queryParams.pageNum = pageNum || this.pager.current
      queryParams.pageSize = pageSize || queryParams.limit
      //如果查询结果小于1
      // if(this.pager.total<1) 怎么判断点击的是导出全部数据 pageNum为1
      if ((currentPage === queryParams.page && this.table.data.length < 1) || (pageSize < 1)) {
        this.$Notice.warning({title: '暂无数据'})
        return;
      }
      let keys = []
      let titles = []
      for (let i in this.table.columns) {
        if (i >= 1) {
          titles.push(this.table.columns[i].title)
          keys.push(this.table.columns[i].key || this.table.columns[i].slot)
        }

      }

      this.$iqis.download("/afterSale/export/list", {
        queryParams: JSON.stringify(queryParams),
        titles,
        keys

      }, "get", () => {

      })

    },
    getQueryParams() {

      /**
       * 查询参数
       */
      const {ISBATCH,THREEYEAR,SIXKILOMETRE,TENKILOMETRE,COMPLAIN,INFOSTATUS, DATETYPE, operationDate, key, value} = this.search

      //分页条件 当前页 每页显示多少条数据
      const {current, limit} = this.pager;

      const params = {
        pageNum: current,
        pageSize: limit,
      };

      if (operationDate && operationDate[0]) {
        params.startDate = Date.parse(operationDate[0]);
        params.endDate = Date.parse(operationDate[1]);
        if (params.startDate === params.endDate) {
          params.endDate += 24 * 60 * 60 * 1000;
        }
      }
      if (value) {
        if (key) {
          params.field = key;
        }
        params.keyword = value;

      }
      if (THREEYEAR) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "THREEYEAR", THREEYEAR);
      }
      if (SIXKILOMETRE) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "SIXKILOMETRE", SIXKILOMETRE);
      }
      if (ISBATCH) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "ISBATCH", ISBATCH);
      }
      if (TENKILOMETRE) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "TENKILOMETRE", TENKILOMETRE);
      }
      if (COMPLAIN) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "COMPLAIN", COMPLAIN);
      }
      if (DATETYPE) {
        params.specialQuery = this.$iqis.utils.query.join(params.specialQuery, "DATETYPE", DATETYPE);
      }

      if (INFOSTATUS||INFOSTATUS==0) {
        params.stateQuery = this.$iqis.utils.query.join(params.stateQuery, "INFOSTATUS", INFOSTATUS);
      }
      //返回查询条件对象
      return params;
    },
    async deleteByIds() {
      let selectedRow = this.$refs.selection.getSelection();
      let ids = selectedRow.map(item => item.afterSaleId).join(",");
      this.$http.delete("/afterSale/batch/delete/" + ids).then(
        () => {
          this.$Message.success("删除成功!")
          this.refreshData()
        }
      )

    },
    selectedChange(selectedRow) {
      if (selectedRow.length === 1) {
        this.updateDisable = false
      } else {
        this.updateDisable = true
      }
      this.deleteDisable = selectedRow.length < 1;
    },
    editSelectRow() {//修改
      let selectedRow = this.$refs.selection.getSelection()
      // console.log("selectedRow", selectedRow)
      //选中的长度不等于1则提示
      if (selectedRow.length !== 1) {
        this.$Notice.warning({
          title: i18n.t('system.cbm'),
          desc: ''
        });
        return;
      }

      let obj = {
        type: "修改",
      }
      obj = selectedRow[0];
      obj.type = "修改";

      this.$emit("open-tab", "after-information-set", "售后信息修改",
        "page-after-sales-message-add",
        {
          obj,
          refresh: this.refreshData
        }
      )
    },
    //复制
    copyData() {
      let selectedRow = this.$refs.selection.getSelection()
      // console.log("selectedRow", selectedRow)
      //选中的长度不等于1则提示
      if (selectedRow.length !== 1) {
        this.$Notice.warning({
          title: i18n.t('system.cbm'),
          desc: ''
        });
        return;
      }
      let obj = {
        type: "复制",
      }
      obj = selectedRow[0];
      obj.type = "复制";

      this.$emit("open-tab", "after-information-set", "复制",
        "page-after-sales-message-add",
        {
          obj,
          refresh: this.refreshData
        })
    },
    //批量导入数据
    batchImportData() {

    },
    //返件
    returnPiece() {
      let selectedRow = this.$refs.selection.getSelection()


      let error = selectedRow.filter(item => (item.traceStatus !== "0")).length;
      //选中的长度不等于1则提示
      if (error > 0) {
        this.$Message.warning(`请确保勾选的数据都是未返件的！`)
        return;
      }
      let resp = save(selectedRow);
      resp.then(() => {
        this.refreshData()
      })

      if (resp) {
        this.$Message.success(`返件成功`)

      } else {
        this.$Message.error(`返件失败`)
      }


    },
    //索赔分配
    claimDistribution() {
      let selectedRow = this.$refs.selection.getSelection()
      //选中的长度不等于1则提示
      if (selectedRow.length !== 1) {
        this.$Notice.warning({
          title: i18n.t('system.cbm'),
          desc: ''
        });
        return;
      }
      //选中的长度不等于1则提示
      if (selectedRow[0].claimBearer) {
        if (this.$store.user.jobId == -10086) {
          this.$Message.warning(`索赔信息已分配，您是售后信息员，可再次修改信息`)
          let obj = {
            type: "索赔",
          }
          obj = selectedRow[0];
          obj.type = "索赔";

          this.$emit("open-tab", "after-information-set", "索赔分配",
            "page-after-sales-message-add",
            {
              obj,
              refresh: this.refreshData
            })


        } else {
          this.$Message.warning(`已分配，您不是售后信息员，不可再次修改`)

        }

      } else {
        this.$Message.warning({
          content: "索赔分配仅能填写一次，请谨慎填写!",
          top: 250,
          duration: 15,
          closable: true
        });
        let obj = {
          type: "索赔",
        }
        obj = selectedRow[0];
        obj.type = "索赔";

        this.$emit("open-tab", "after-information-set", "索赔分配",
          "page-after-sales-message-add",
          {
            obj,
            refresh: this.refreshData
          })
      }


    },
    //申诉
    complaint() {
      let selectedRow = this.$refs.selection.getSelection()
      // console.log("selectedRow", selectedRow)
      //选中的长度不等于1则提示
      if (selectedRow.length !== 1) {
        this.$Notice.warning({
          title: i18n.t('system.cbm'),
          desc: ''
        });
        return;
      }
      let obj = {
        type: "申诉",
      }
      obj = selectedRow[0];
      obj.type = "申诉";

      this.$emit("open-tab", "after-information-set", "申诉",
        "page-after-sales-message-add",
        {
          obj,
          refresh: this.refreshData
        })

    },
    dblClick(selectedRow) {//修改

      let obj = {
        type: "修改",
      }
      obj = selectedRow;
      obj.type = "修改";

      //先关闭自己修改页面
      this.$emit("close-tab", "after-information-set", "售后信息修改",
        "page-after-sales-message-add",)

      this.$emit("open-tab", "after-information-set", "售后信息修改",
        "page-after-sales-message-add",
        {
          obj,
          refresh: this.refreshData
        })
    },
    toAddComponent() {

      let obj = {
        id: '',
        type: "添加",
        supplierWorkingHoursFee: 0,
        supplierMaterialFee: 0,
        supplierManagementFee: 0,
        supplierSupplementaryFee: 0,
        supplierServiceFee: 0,
        accidentCompensation: 0,
        supplierTransportationFee: 0,
        supplierClaimAmount: 0,
        costAdvanced: 0,
        caiculte: 0,
      }
      obj.type = "添加";
      this.$emit("open-tab", "after-information-add", "售后信息添加",
        "page-after-sales-message-add",
        {
          obj,
          refresh: this.refreshData
        })
    }, progress() {
      this.table.loading = true;
    },
    async refreshData() {
      /**这个搜索请求只需要修改请求地址即可*/
      this.table.loading = true;

      //当查询参数发生改变或者当前页面发生改变
      let params = this.getQueryParams();
      //调用查询列表
      let resp = await getList(params);

      this.table.data = resp.data || [];
      //总记录数
      this.pager.total = resp.pagerInfo.total;
      this.pager.current = resp.pagerInfo.page;
      this.pager.limit = resp.pagerInfo.limit;


      //关闭加载状态
      this.table.loading = false;


    }, async synchData() {//同步数据
      //开启加载状态
      this.table.loading = true;
      this.$Message.info({
        content: '同步数据加载比较慢，请耐心等待',
        duration: 10
      })

      let res = await synchData();


      if (res.data) {
        //关闭加载状态
        this.table.loading = false;
        this.$Message.success(`同步成功`)
        this.refreshData();
      } else {
        this.table.loading = false;
        this.$Message.error(`无数据需要更新`)

      }

    },
    //页面初始化,数据全部用一开始的数据
    reset() {
      this.search = this.$options.data().search
      this.refreshData();
    },
    //当前页面改变
    pageChange(current) {
      this.pager.current = current;
      this.refreshData()
    },
    //当前页大小改变,默认把当前页修改为1
    pageSizeChange(limit) {
      this.pager.current = 1
      this.pager.limit = limit
      this.refreshData()
    }
  },
  watch: {
    labRefresh() {
      this.refreshData();
    }
  }
}
</script>

<style lang="less"  scoped>


.laboratory-standards-list {
  /deep/ .ivu-row-flex {
    margin-top: 8px;
  }

  .myfirstRow {
    margin-top: 0px;
  }

  /deep/ .ivu-modal-body {
    padding: 12px;

  }

  /deep/ .ivu-table-wrapper {
    margin-top: 10px;

    .ivu-table-body {
      height: calc(100% - 40px);
      overflow-y: auto;
    }

  }


  .ivu-dropdown-menu .ivu-dropdown-item {
    padding: 7px 16px !important;
  }

  /deep/ .ivu-upload-select {
    color: #515a6e;
    font-size: 14px !important;
  }


}
</style>
