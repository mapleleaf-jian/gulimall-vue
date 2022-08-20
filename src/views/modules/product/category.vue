<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>

    <el-button v-if="draggable" @click="handleBatchUpdate">批量保存</el-button>

    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandNodes"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
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
      draggable: false, // 是否开启拖拽
      appendName: '',
      maxLevel: 0, // 存储拖拽节点的最大的深度
      updateNodes: [], // 存储需要更新的节点信息，一起发送到数据库
      pCid: [] // 当前拖动节点的父节点id
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
    // 被拖拽节点对应的 Node、结束拖拽时最后进入的节点、被拖拽节点的放置位置(before、after、inner)、event
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('tree drop: ', draggingNode, dropNode, dropType, ev);
      // 1. 当前节点最新的父节点id
      let pCid = 0
      let broNodes = null // 拖拽后的兄弟节点集合

      if (dropType === 'inner') {
        pCid = dropNode.data.catId
        broNodes = dropNode.childNodes
      } else {
        pCid = dropNode.parent.data.catId || 0
        broNodes = dropNode.parent.childNodes
      }
      this.pCid.push(pCid)

      // 2. 当前拖拽节点的最新顺序
      for (let i = 0; i < broNodes.length; i++) {
        // 如果当前遍历的是正在 拖拽 的节点，还需要修改它的 父级id
        if (broNodes[i].data.catId === draggingNode.data.catId) {
          let cLevel = draggingNode.level
          // 如果当前节点的层级发生变化
          if (broNodes[i].level !== draggingNode.level) {
            cLevel = draggingNode.level
            // 递归修改子节点的层级
            this.updateChildNodeLevel(broNodes[i])
          }
          this.updateNodes.push({catId: broNodes[i].data.catId, sort: i, parentCid: pCid, catLevel: cLevel})
        } else {
          this.updateNodes.push({catId: broNodes[i].data.catId, sort: i})
        }
      }
      console.log(this.updateNodes)
    },
    handleBatchUpdate() {
      // 发送请求，修改数据库
      this.$http({
        url: this.$http.adornUrl('/product/category/update/batch'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '更新成功!'
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandNodes = this.pCid

        // 重置
        this.maxLevel = 0
        this.updateNodes = []
        this.pCid = []
      })
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let cNode = node.childNodes[i].data
          this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // console.log(draggingNode, dropNode, type)
      // 计算当前拖动的节点 (draggingNode) 的最大深度(即最大level)
      this.countDeep(draggingNode)
      // 计算当前拖动的节点往下总共有多少层
      let draggingLevel = Math.abs(this.maxLevel - draggingNode.level) + 1
      // console.log('当前节点往下有' + + draggingLevel + '层')
      if (type === 'inner') {
        return draggingLevel + dropNode.data.catLevel <= 3
      } else {
        return draggingLevel + dropNode.parent.level <= 3
      }
    },
    countDeep(node) {
      if (node.childNodes !== null && node.childNodes.length !== 0) {
        for (let child of node.childNodes) {
          if (child.level > this.maxLevel) {
            this.maxLevel = child.level
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
      this.category.sort = 0;
      this.category.showStatus = 1
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
