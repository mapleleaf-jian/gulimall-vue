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
            type="text"
            size="mini"
            @click="() => edit(node, data)">
            Edit
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
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="菜单名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
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
  name: 'category',
  data() {
    return {
      menus: [],
      dialogType: '',
      category: {
        name: '',
        showStatus: 1,
        sort: 0,
        catLevel: 0,
        parentCid: 0,
        catId: null,
        icon: '',
        productUnit: ''
      },
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
  computed: {
    title() {
      return (this.dialogType === 'add' ? '添加' : '修改') + '菜单'
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
    submitData() {
      if (this.dialogType === 'add') {
        this.addCategory()
      } else if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    append(data) {
      this.dialogVisible = true
      this.dialogType = 'add'
      this.category.catLevel = data.catLevel * 1 + 1 // 保证是数字
      this.category.parentCid = data.catId

      this.category.catId = null
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
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
    edit(node, data) {
      console.log(node, data)
      this.dialogVisible = true
      this.dialogType = 'edit'

      // 在数据库查询数据进行回显
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 按需回显数据
        this.category.catId = data.category.catId
        this.category.name = data.category.name
        this.category.icon = data.category.icon
        this.category.productUnit = data.category.productUnit
        this.category.parentCid = data.category.parentCid
      })
    },
    editCategory() {
      let {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '修改成功!'
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
