<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :default-expanded-keys="expandedKey" :data="menus" :props="defaultProps" @node-click="handleNodeClick"
             @node-drop="handleDrop"
             show-checkbox node-key="catId"
             :draggable="draggable"
             :allow-drop="allowDrop"
             ref="menuTree"
             :expand-on-click-node="false">
        <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level<=2"
            type="text"
            size="mini"
            @click="() => append(data)">
           添加
          </el-button>
             <el-button
               type="text"
               size="mini"
               @click="() => edit(data)">
           编辑
          </el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :close-on-click-modal="false"
      :title=title
      :visible.sync="dialogVisible"
      width="30%"
      :before-close="handleClose">
      <el-form :model="categroy">
        <el-form-item label="分类名称" :label-width="formLabelWidth">
          <el-input v-model="categroy.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标" :label-width="formLabelWidth">
          <el-input v-model="categroy.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位" :label-width="formLabelWidth">
          <el-input v-model="categroy.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
    <el-button @click="dialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="submitData">确 定</el-button>
  </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  components: {},
  mounted() {
  },
  beforeCreate() {
  },
  beforeDestroy() {
  },
  destroyed() {
  },
  //如果有keepaLive缓存功能，这个函数触发
  activated() {
    this.getMenus();
  },
  beforeUpdate() {
  },
  created() {
  },
  data() {
    return {
      draggable: false,
      title: "",
      maxLevel: 0,
      formLabelWidth: '120px',
      categroy: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: '',
        productUnit: '',
        catId: null
      },
      dialogType: '',
      dialogVisible: false,
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    };
  },
  methods: {
    batchDelete() {
      let catIds = [];
      let checkNodes = this.$refs.menuTree.getCheckedNodes();
      for (let i = 0; i < checkNodes.length; i++) {
        catIds.push(checkNodes[i].catId);
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl("/product/category/delete"),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: '菜单删除成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getMenus();
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => {
      })
    },
    batchSave() {

    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('tree drop: ', dropNode.label, dropType);
    },
    edit(data) {
      this.title = "修改分类";
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get',
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.categroy.name = data.data.name;
          this.categroy.catId = data.data.catId;
          this.categroy.icon = data.data.icon;
          this.categroy.productUnit = data.data.productUnit;
          this.categroy.parentCid = data.data.parentCid;
        } else {
          this.$message.error(data.msg)
        }
      })

    },
    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode.data);
      let deep = (this.maxLevel - draggingNode.data.catLevel) + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
      //   console.log(draggingNode.data,dropNode,type)
      return false;

    },
    countNodeLevel(node) {
      //找出所有子节点，求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel
          }
          this.countNodeLevel(node.children[i])

        }


      }


    },
    editCategroy() {
      let {catId, name, icon, productUnit} = this.categroy;
      this.$http({
        url: this.$http.adornUrl(`/product/category/update`),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        this.$message({
          message: "菜单修改成功",
          type: "success"
        })
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.categroy.parentCid];
      })

    },
    submitData(data) {
      if (this.dialogType == "edit") {
        this.editCategroy()

      } else if (this.dialogType = "add") {

        this.addCategroy()
      }

    },
    addCategroy() {

      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: 'post',
        data: this.$http.adornData(this.categroy, false)
      }).then(({data}) => {
        this.dialogVisible = false;
        if (data && data.code === 0) {
          this.$message({
            message: '菜单添加成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              this.getMenus();
              this.expandedKey = [this.categroy.parentCid];
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })


    },
    handleClose() {

    },
    append(data) {
      this.title = "添加分类";
      this.dialogType = "add";
      this.dialogVisible = true;
      this.categroy.parentCid = data.catId;
      this.categroy.catLevel = data.catLevel * 1 + 1;

    },
    remove(node, data) {
      // 删除
      let ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl("/product/category/delete"),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: '菜单删除成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getMenus();
                this.expandedKey = [node.parent.data.catId];
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => {
      })

    },
    handleNodeClick(data,node,component) {
      this.$emit("treeNodeCli",data,node,component)
      console.log(data);
    },
    // 获取数据列表
    getDataList() {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get',
      }).then(({data}) => {
        this.menus = data.list
        this.dataListLoading = false
      })
    },
    getMenus() {

      this.getDataList();

    },
  },
  props: {},
  computed: {},
  watch: {}

}
</script>

<style scoped>

</style>
