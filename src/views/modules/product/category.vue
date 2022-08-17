<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" show-checkbox node-key="catId" :expand-on-click-node="false">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)">
            Append
          </el-button>
          <el-button
            v-if="node.isLeaf"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
  </div>
</template>

<script>
export default {
  name: 'category',
  data() {
    return {
      menus: [],
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
    append(data) {

    },
    remove(node, data) {

    }
  },
  created() {
    this.getMenus()
  }
}
</script>

<style scoped>

</style>
