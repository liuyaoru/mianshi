<template>
  <div>
    <el-row :gutter="20">
      <el-col :span="6">
        <category @treeNodeCli="treeNodeClick"></category>
      </el-col>
      <el-col :span="18">
        <div class="mod-config">
          <el-form :inline="true" :model="dataForm" @keyup.native.native="getDataList()">
            <el-form-item>
              <el-input v-model="dataForm.key" placeholder="参数名" clearable></el-input>
            </el-form-item>
            <el-form-item>
              <el-button >查询</el-button>
              <el-button type="success"  @click="getAllDataList()" >查询全部</el-button>
              <el-button
                type="primary"
                @click="addOrUpdateHandle"
              >新增</el-button>
              <el-button
                type="danger"
                :disabled="dataListSelections.length <= 0"
              >批量删除</el-button>
            </el-form-item>
          </el-form>
          <el-table
            :data="dataList"
            border
            v-loading="dataListLoading"
            style="width: 100%;"
          >
            <el-table-column type="selection" header-align="center" align="center" width="50"></el-table-column>
            <el-table-column prop="attrGroupId" header-align="center" align="center" label="分组id"></el-table-column>
            <el-table-column prop="attrGroupName" header-align="center" align="center" label="组名"></el-table-column>
            <el-table-column prop="sort" header-align="center" align="center" label="排序"></el-table-column>
            <el-table-column prop="descript" header-align="center" align="center" label="描述"></el-table-column>
            <el-table-column prop="icon" header-align="center" align="center" label="组图标"></el-table-column>
            <el-table-column prop="catelogId" header-align="center" align="center" label="所属分类id"></el-table-column>
            <el-table-column
              fixed="right"
              header-align="center"
              align="center"
              width="150"
              label="操作"
            >
              <template slot-scope="scope">
                <el-button type="text" size="small"  @click="relationHandle(scope.row.attrGroupId)">关联</el-button>
                <el-button type="text" size="small" @click="addOrUpdateHandle(scope.row.attrGroupId)">修改</el-button>
                <el-button type="text" size="small" @click="deleteHandle(scope.row.attrGroupId)">删除</el-button>
              </template>
            </el-table-column>
          </el-table>
          <el-pagination
            :current-page="pageIndex"
            :page-sizes="[10, 20, 50, 100]"
            :page-size="pageSize"
            :total="totalPage"
            layout="total, sizes, prev, pager, next, jumper"
          ></el-pagination>
          <!-- 弹窗, 新增 / 修改 -->
          <attr-group-add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="getDataList"></attr-group-add-or-update>
          <!-- 修改关联关系 -->
          <relation-update v-if="relationVisible" ref="relationUpdate" @refreshData="getDataList"></relation-update>
        </div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import RelationUpdate from "./attr-group-relation";
import Category from "./categroy"
import AttrGroupAddOrUpdate from "./attrgroup-add-or-update";
export default {
  components: {
    Category:Category,
    attrGroupAddOrUpdate:AttrGroupAddOrUpdate,
    RelationUpdate
  },
  mounted() {
  },
  getAllDataList()
  {
    this.catId = 0;
    this.getDataList();
  },
  beforeCreate() {
  },
  beforeDestroy() {
  },
  destroyed() {
  },
  //如果有keepaLive缓存功能，这个函数触发
  activated() {
  this.getDataList();
  },
  beforeUpdate() {
  },
  created() {
  },
  data() {
      return {
        catId: 0,
        dataForm: {
          key: ""
        },
        dataList: [],
        pageIndex: 1,
        pageSize: 10,
        totalPage: 0,
        dataListLoading: false,
        dataListSelections: [],
        addOrUpdateVisible: false,
        relationVisible:true
      };
  },
  methods: {
    getAllDataList(){
      this.catId = 0;
      this.getDataList();
    },
    //处理分组与属性的关联
    relationHandle(groupId) {
      this.relationVisible = true;
      this.$nextTick(() => {
        this.$refs.relationUpdate.init(groupId);
      });
    },
    deleteHandle(id)
    {

      let ids = id ? [id] : this.dataListSelections.map(item => {
          return item.attrGroupId;
        });
      this.$confirm(
        `确定对[id=${ids.join(",")}]进行[${id ? "删除" : "批量删除"}]操作?`,
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning"
        }
      ).then(() => {
        this.$http({
          url: this.$http.adornUrl("/product/attrgroup/delete"),
          method: "post",
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: "操作成功",
              type: "success",
              duration: 1500,
              onClose: () => {
                this.getDataList();
              }
            });
          } else {
            this.$message.error(data.msg);
          }
        });
      });

    },
    addOrUpdateHandle(id)
    {
      this.addOrUpdateVisible = false;
      this.addOrUpdateVisible = true;
      this.$nextTick(() => {
      /*  this.$refs.addOrUpdate.init(id);*/
      });
    },

    treeNodeClick(data,node,component)
    {
      if(node.level==3)
      {
        this.catId=data.catId
        this.getDataList();
      }
    },

    getDataList()
    {
      // 获取数据列表
      this.dataListLoading = false;
      this.$http({
        url: this.$http.adornUrl(`/product/attrgroup/list/${this.catId}`),
        method: "get",
        params: this.$http.adornParams({
          page: this.pageIndex,
          limit: this.pageSize,
          key: this.dataForm.key
        })
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.dataList = data.page.list;
          this.totalPage = data.page.totalCount;
        } else {
          this.dataList = [];
          this.totalPage = 0;
        }
        /*     this.dataListLoading = false;*/
      });

    }
  },
  props: {},
  computed: {},
  watch: {}

}
</script>

<style scoped>

 .el-row {
  margin-bottom: 20px;
   &:last-child {
   margin-bottom: 0;
 }
}
.el-col {
  border-radius: 4px;
}
.bg-purple-dark {
  background: #99a9bf;
}
.bg-purple {
  background: #d3dce6;
}
.bg-purple-light {
  background: #e5e9f2;
}
.grid-content {
  border-radius: 4px;
  min-height: 36px;
}
.row-bg {
  padding: 10px 0;
  background-color: #f9fafc;
}
</style>
