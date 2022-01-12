<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="nodeClick"
      ref="menuTree"
      >
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

  },
  beforeUpdate() {
  },
  created() {
    this.getMenus()
  },
  data() {
    return {
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  methods: {
    nodeClick(data,node,component)
    {
      console.log(data,node,component)
        this.$emit("treeNodeCli",data,node,component)
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
