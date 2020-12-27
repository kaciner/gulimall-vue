<template>
  <div>
    <el-switch
      v-model="draggable"
      style="margin-bottom: 10px; margin-left: 5px"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button @click="batchSave" v-if="draggable" style="margin: 0px 0px 10px 5px" size="mini">批量保存</el-button>
    <el-button type="danger" @click="batchDelete" style="margin: 0px 0px 10px 5px" size="mini">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false"
             show-checkbox node-key="catId" :default-expanded-keys="expandedKey" :draggable="draggable"
             :allow-drop="allowDrop" @node-drop="handleDrop" ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
          <span>{{ node.label }}</span>
          <span>
            <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">Append</el-button>
            <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
            <el-button v-if="node.childNodes.length == 0" type="text" size="mini" @click="() => remove(node, data)">Delete</el-button>
          </span>
        </span>
    </el-tree>
    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
      width="30%">
      <el-form :model="category" :rules="rules" ref="categoryForm">
        <el-form-item label="分类名称" prop="name">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标" prop="icon">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位" prop="productUnit">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
          <el-button @click="dialogVisible = false">取 消</el-button>
          <el-button type="primary" @click=submitData>确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    name: 'category',
    data() {
      return {
        pCid: [],
        maxLevel: 0,
        draggable: false,
        updateNodes: [],
        category: {name: "", parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: "", productUnit: "", catId: null},
        dialogType: "",
        dialogTitle: "",
        dialogVisible: false,
        menus: [],
        expandedKey: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        },
        rules: {
          name: [{required: true, message: '请输入分类名称', trigger: 'change'}]
        }
      }
    },
    methods: {
      getMenus() {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get',
        }).then(({data}) => {
          this.menus = data.data;
        })
      },

      allowDrop(draggingNode, dropNode, type) {
        // 被拖动的当前节点以及所在的父节点总层数不能大于3
        this.countNodeLevel(draggingNode);
        // 当前正在拖动的节点+父节点所在的深度不大于3即可
        let deep = Math.abs((this.maxLevel - draggingNode.level) + 1);
        if(type == "inner") {
            return (deep + dropNode.level) <= 3;
        }else {
            return (deep + dropNode.parent.level) <= 3;
        }
      },

      handleDrop(draggingNode, dropNode, dropType, ev) {
        //1. 当前节点的最新父节点id
        console.log("handleDrop: ", draggingNode, dropNode, dropType)
        let pCid = 0;
        let siblings = null;
        if(dropType == "before" || "after") {
            pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId;
            siblings = dropNode.parent.childNodes;
        }else {
            pCid = dropNode.data.catId;
            siblings = dropNode.childNodes;
        }
        this.pCid.push(pCid);
        //2. 当前拖拽节点的最新顺序
        for(let i = 0; i < siblings.length; i++) {
            if(siblings[i].data.catId == draggingNode.data.catId) {
                // 如果遍历的是当前正在拖拽的节点
              let catLevel = draggingNode.level;
                if(siblings[i].level == draggingNode.level) {
                    // 当前节点的层级发生变化
                    catLevel = siblings[i].level;
                    // 修改子节点的层级
                    this.updateChildNodeLevel(siblings[i]);
                }
                this.updateNodes.push({catId: siblings[i].data.catId, sort:i, parentCid: pCid});
            }else {
                this.updateNodes.push({catId: siblings[i].data.catId, sort:i});
            }
        }
        //3. 当前拖拽节点的最新层级
      },

      batchSave() {
        this.$http({
          url: this.$http.adornUrl('/product/category/update/sort'),
          method: 'post',
          data: this.$http.adornData(this.updateNodes, false)
        }).then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单顺序修改成功!'
          });
          this.getMenus();
          // this.pCid = 0;
          this.maxLevel = 0;
          this.updateNodes = [];
          this.expandedKey = this.pCid;
        })
      },

      batchDelete() {
          let catIds = [];
          let checkedNodes = this.$refs.menuTree.getCheckedNodes();
          for(let i = 0; i < checkedNodes.length; i++) {
            catIds.push(checkedNodes[i].catId);
          }
          this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }).then(() => {
            this.$http({
              url: this.$http.adornUrl('/product/category/delete'),
              method: 'post',
              data: this.$http.adornData(catIds, false)
            }).then(({data}) => {
              this.$message({
                type: 'success',
                message: '菜单删除成功!'
              });
              this.getMenus();
            })
          }).catch(() => {
            this.$message({
              type: 'info',
              message: '已取消删除'
            });
          });
      },

      updateChildNodeLevel(node) {
          if(node.childNodes.length > 0) {
              for(let i = 0; i < node.childNodes.length; i++) {
                  var cNode = node.childNodes[i].data;
                  this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level});
                  this.updateChildNodeLevel(node.childNodes[i]);
              }
          }
      },

      countNodeLevel(node) {
        // 找到所有子节点，求出最大深度
          if(node.childNodes != null && node.childNodes.length > 0) {
              for(let i = 0; i < node.childNodes.length; i++) {
                  if(node.childNodes[i].level > this.maxLevel) {
                      this.maxLevel = node.childNodes[i].level;
                  } else {
                    this.maxLevel = node.level;
                  }
                  this.countNodeLevel(node.childNodes[i]);
              }
          }
      },

      submitData() {
          if(this.dialogType === "add") {
              this.$refs['categoryForm'].validate((valid) => {
                  if (valid) {
                      this.addCategory();
                  }
              });
          }
          if(this.dialogType === "edit") {
              this.$refs['categoryForm'].validate((valid) => {
                  if (valid) {
                      this.editCategory();
                  }
              });
          }
      },

      append(data) {
          this.dialogType = "add";
          this.dialogTitle = "添加分类";
          this.dialogVisible = true;
          this.category.parentCid = data.catId;
          this.category.catLevel = data.catLevel * 1 + 1;
          this.category.catId = null;
          this.category.name = "";
          this.category.icon = "";
          this.category.productUnit = "";
          this.category.sort = 0;
          this.category.showStatus = 1;
      },

      //添加三级分类
      addCategory() {
          this.$http({
              url: this.$http.adornUrl('/product/category/save'),
              method: 'post',
              data: this.$http.adornData(this.category, false)
          }).then(({data}) => {
                this.$message({
                type: 'success',
                message: '菜单保存成功!'
            });
              this.dialogVisible = false;
              this.getMenus();
              this.expandedKey = [this.category.parentCid];
          })
      },

      edit(data) {
          this.dialogType = "edit";
          this.dialogTitle = "修改分类";
          this.dialogVisible = true;
          this.$http({
              url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
              method: 'get',
          }).then(({data}) => {
              this.category.name = data.data.name;
              this.category.catId = data.data.catId;
              this.category.icon = data.data.icon;
              this.category.productUnit = data.data.productUnit;
              this.category.parentCid = data.data.parentCid;
              this.category.catLevel = data.data.catLevel;
              this.category.sort = data.data.sort;
              this.category.showStatus = data.data.showStatus;
          })
      },

      //修改三级分类
      editCategory() {
        var {catId, name, icon, productUnit} = this.category;
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData({catId, name, icon, productUnit}, false)
        }).then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单修改成功!'
          });
          this.dialogVisible = false;
          this.getMenus();
          this.expandedKey = [this.category.parentCid];
        })
      },

      remove(node, data) {
        var ids = [data.catId]
        this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              type: 'success',
              message: '菜单删除成功!'
            });
            //刷新出新的菜单
            this.getMenus();
            //设置默认需要展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          });
        });
      },

    },
    created () {
      this.getMenus();
    }
  }
</script>

<style scoped>

</style>
