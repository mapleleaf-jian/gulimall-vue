<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandNodes"
      draggable
      :allow-drop="allowDrop"
    >
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

    <el-dialog
      title="提示"
      :visible.sync="dialogVisible"
      width="30%">
      <el-form :model="category">
        <el-form-item label="菜单名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCategory">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'category',
  data() {
    return {
      menus: [],
      category: {name: '', showStatus: 1, sort: 0},
      dialogVisible: false,
      expandNodes: [], // 默认展开的节点
      defaultProps: {
        children: 'chilNodes',
        label: 'name'
      },
      appendName: '',
      maxLevel: 0
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
    allowDrop(draggingNode, dropNode, type) {
      // console.log(draggingNode, dropNode, type)
      // 计算当前拖动的节点 (draggingNode) 的最大深度(即最大level)
      this.countDeep(draggingNode.data)
      // 计算当前拖动的节点往下总共有多少层
      let draggingLevel = this.maxLevel - draggingNode.data.catLevel + 1
      // console.log('当前节点往下有' + + draggingLevel + '层')
      if (type === 'inner') {
        return draggingLevel + dropNode.data.catLevel <= 3
      } else {
        return draggingLevel + dropNode.parent.data.catLevel <= 3
      }
    },
    countDeep(node) {
      if (node.chilNodes !== null && node.chilNodes.length !== 0) {
        for (let child of node.chilNodes) {
          if (child.catLevel > this.maxLevel) {
            this.maxLevel = child.catLevel
          }
          // 递归找每个子节点的最大深度
          this.countDeep(child)
        }
      }
    },
    append(data) {
      this.dialogVisible = true
      this.category.catLevel = data.catLevel * 1 + 1 // 保证是数字
      this.category.parentCid = data.catId
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '添加成功!'
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandNodes = [this.category.parentCid]
      })
    },
    remove(node, data) {
      const ids = [data.catId]
      this.$confirm(`确认删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          this.$message({
            type: 'success',
            message: '删除成功!'
          })
          this.expandNodes = [node.parent.data.catId]
          this.getMenus()
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    }
  },
  created() {
    this.getMenus()
  }
}
</script>

<style scoped>

</style>
