<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDel">批量删除</el-button>
    <el-tree
      :data="menuData"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <!-- 新增按钮 -->
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >Append</el-button>
          <!-- 修改按钮 -->
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
          <!-- 删除按钮 -->
          <el-button
            v-if="node.childNodes ==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
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
        <el-form-item label="三级分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
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
  props: {},
  data() {
    return {
      category: {
        name: "",
        parentCid: 0,
        catLevel: 1,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        catId: null
      },
      pCid: [], //定义全局父节点数组，展开对应修改过的层级结构
      draggable: false, //默认不可拖拽
      updateNodes: [], //所有要更新的节点的集合
      maxLevel: 0, //定义最大的等级数,也就是子节点的等级数
      title: "", //弹框提示信息
      dialogType: "", //弹框提示，依据新增/修改 按钮 进行提示
      dialogVisible: false, //默认不弹框
      menuData: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name"
      }
    };
  },

  //计算属性
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    //从后台获取属性结构数据，进行展示
    getTreeMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/treeMenu"),
        method: "get"
      }).then(({ data }) => {
        // console.log("数据", data.listTree);
        this.menuData = data.listTree;
      });
    },
    //根据新增/修改，向后台发送请求
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory(); //新增三级分类
      }
      if (this.dialogType == "edit") {
        this.editCategory(); //修改三级分类
      }
    },
    ////批量删除选中的节点
    batchDel() {
      let delNodes = []; //选中节点的集合
      let childrenNodes = this.$refs.menuTree.getCheckedNodes();
      // console.log("选中的节点", childrenNodes);
      for (let i = 0; i < childrenNodes.length; i++) {
        delNodes.push(childrenNodes[i].catId);
      }
      this.$confirm(`确定批量删除【${delNodes}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(delNodes, false)
          }).then(({ data }) => {
            this.$message({
              message: "菜单批量删除成功",
              type: "success"
            });
            this.getTreeMenu(); //重新加载
          });
        })
        .catch(() => {});
    },
    //批量保存修改的节点内容
    batchSave() {
      //将收集到的要更新的节点向后台发送请求
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单拖拽顺序修改成功",
          type: "success"
        });
        this.getTreeMenu(); //重新加载
        this.expandedKey = this.pCid; //展开拖拽目标节点的三级分类的父节点
        this.updateNodes = []; //每次拖拽时，所有要更新的节点的集合需要置空
        this.maxLevel = 0;
      });
    },
    //新增分类
    append(data) {
      // console.log("新增", data);
      this.dialogVisible = true; //打开对话框
      this.dialogType = "add"; //弹框提示信息为：新增
      this.title = "新增三级分类";
      this.category.parentCid = data.catId; //当前分类的父id
      this.category.catLevel = data.catLevel * 1 + 1; //层级为父层级+1
      this.category.name = "";
      this.category.icon = ""; //图标
      this.category.productUnit = ""; //计量单位
      this.category.catId = null;
      this.category.showStatus = 1;
      this.category.sort = 0;
    },
    //新增分类后台请求发送
    addCategory() {
      // console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单新增成功",
          type: "success"
        });
        this.dialogVisible = false; //关闭模态框
        this.getTreeMenu(); //重新加载
        this.expandedKey = [this.category.parentCid]; //展开删除的三级分类的父节点
      });
    },
    //修改分类
    edit(data) {
      this.dialogVisible = true; //打开模态框
      this.dialogType = "edit"; //弹框提示信息为：修改
      this.title = "修改三级分类";
      // console.log("要编辑的catId",data.catId)
      //回显内容从数据库查询
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get"
      }).then(({ data }) => {
        // console.log("1111",data.data)
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catId = data.data.catId;
      });
    },
    //修改分类后台请求发送
    editCategory() {
      // console.log(this.category);
      var { catId, icon, productUnit, name } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, icon, productUnit, name }, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success"
        });
        this.dialogVisible = false; //关闭模态框
        this.getTreeMenu(); //重新加载
        this.expandedKey = [this.category.parentCid]; //展开删除的三级分类的父节点
      });
    },
    //删除分类
    remove(node, data) {
      // console.log("删除", node, data);
      var ids = [data.catId];
      this.$confirm(`确定删除【${data.name}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success"
            });
            //重新加载
            this.getTreeMenu();
            //展开删除的三级分类的父节点
            this.expandedKey = [node.parent.data.catId];
            // console.log("父节点", this.expandedKey);
          });
        })
        .catch(() => {});
    },
    //拖拽成功完成时触发的事件
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // console.log("事件: ", draggingNode, dropNode, dropType);
      //1.当前节点的最新父节点
      let pCid = 0;
      let allChildrenNodes = null; //目标节点的所有子节点
      //根据拖动的位置，父节点的取值也不同
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        allChildrenNodes = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        allChildrenNodes = dropNode.childNodes;
      }
      this.pCid.push(pCid);
      // console.log("最新父节点：", pCid);

      //2.当前节点的最新顺序
      for (let i = 0; i < allChildrenNodes.length; i++) {
        if (allChildrenNodes[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前拖拽的节点，则需要更新当前拖拽节点的父节点id
          let catLevel = draggingNode.level;
          if (allChildrenNodes[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = allChildrenNodes[i].level;
            //修改当前节点的子节点的层级
            this.updateChildrenNodeLevel(allChildrenNodes[i]);
          }
          this.updateNodes.push({
            catId: allChildrenNodes[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          });
        } else {
          this.updateNodes.push({
            catId: allChildrenNodes[i].data.catId,
            sort: i
          });
        }
      }
      // console.log("更新节点的集合：", this.updateNodes);
    },
    //3.当前节点的最新层次
    updateChildrenNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          });
          this.updateChildrenNodeLevel(node.childNodes[i]);
        }
      }
    },
    //节点可拖拽的方法
    allowDrop(draggingNode, dropNode, type) {
      //1、当前拖拽节点的总层数和所在父节点的总层数的和不能大于3，只有三个等级的分类

      //当前节点的总层数（循环递归）
      // console.log("thisNode:", draggingNode, dropNode, type);
      this.thisNodeLevel(draggingNode);

      //当前拖动的节点+所在父节点的和<=3就可以拖动
      var deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      // console.log("深度：", deep);
      //判断type的类型，进行拖拽的效果
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    thisNodeLevel(node) {
      //找到所有子节点，统计节点深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        //循环遍历
        for (let i = 0; i < node.childNodes.length; i++) {
          //找出比定义的最大等级数大的等级数
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          //如果有子节点继续找出最大的节点数
          this.thisNodeLevel(node.childNodes[i]);
        }
      }
    }
  },
  //生命周期，创建完成
  created() {
    this.getTreeMenu();
  },
  //生命周期，挂载完成
  mounted() {},
  //生命周期，创建之前
  beforeCreate() {},
  //生命周期，挂载之前
  beforeMount() {},
  //生命周期，更新之前
  beforeUpdate() {},
  //更新之后
  updated() {},
  //生命周期，销毁之前
  beforeDestroy() {},
  //销毁完成
  destroyed() {},
  //如果页面有keep-alive缓存功能，会触发该函数
  activated() {}
};
</script>