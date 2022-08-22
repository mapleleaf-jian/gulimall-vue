<template>
  <el-tree
    :data="menus"
    :props="defaultProps"
    node-key="catId"
    ref="menuTree"
    @node-click="nodeClick"
  >
  </el-tree>
</template>

<script>
export default {
  // 公共的三级菜单组件
  data() {
    return {
      menus: [],
      expandNodes: [],
      defaultProps: {
        children: 'chilNodes',
        label: 'name'
      }
    }
  },
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        this.menus = data.data
      })
    },
    nodeClick(data, node, component) {
      // 子组件触发父组件为其绑定的自定义事件
      this.$emit('tree-node-click', data, node, component)
    }
  },
  created() {
    this.getMenus()
  }
}
</script>

<style scoped>

</style>
