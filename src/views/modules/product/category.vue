<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox="true"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      draggable="true"
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
            @click="() => append(data)"
            >Append</el-button
          >
          <el-button type="text" size="mini" @click="() => edit(data)"
            >Edit</el-button
          >
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
            >Delete</el-button
          >
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">Cancel</el-button>
        <el-button type="primary" @click="submitData">Confirm</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      updateNodes: [],
      maxLevel: 0,
      title: "",
      dialogType: "",
      data: [],
      menus: [],
      expandedKey: [],
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level;
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            parentCid: pCid,
            catLevel: catLevel,
          });
        }
        this.updateNodes.push({ catId: siblings[i].data.catId });
      }
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: (cNode.catLevel = node.childNodes[i].level),
          });

          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      var level = this.countNodeLevel(draggingNode.data);
      let depth = this.maxLevel - draggingNode.data.catLevel + 1;
      if (type == "inner") {
        return depth + dropNode.level <= 3;
      } else {
        return depth + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //find the max level
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = { catId, name, icon, productUnit };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        this.$notify({
          title: "Success",
          message: "菜单修改成功",
          type: "success",
        });
        //close dialog
        this.dialogVisible = false;
        this.getMenus();
        //set default expand menu
        this.expandedKey = [this.category.parentCid];
      });
    },
    edit(data) {
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.title = "修改分类";
      //send request to get the newest data
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("edit", data);
        this.category.name = data.category.name;
        this.category.catId = data.category.catId;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
        this.category.parentCid = data.category.parentCid;
      });
    },

    handleNodeClick(data) {
      console.log(data);
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据", data.data);
        this.menus = data.data;
      });
    },
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
    },
    addCategory() {
      console.log("data", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$notify({
          title: "Success",
          message: "菜单保存成功",
          type: "success",
        });
        //close dialog
        this.dialogVisible = false;
        this.getMenus();
        //set default expand menu
        this.expandedKey = [this.category.parentCid];
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除${data.name}菜单`, "Warning", {
        confirmButtonText: "OK",
        cancelButtonText: "Cancel",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$notify({
              title: "Success",
              message: "菜单删除成功",
              type: "success",
            });
            //refresh new menu
            this.getMenus();
            //set default expand menu
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
  },
  created() {
    this.getMenus();
  },
};
</script>

<style>
</style>