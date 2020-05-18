<template>
  <div>
    <el-tree
      :data="menuData"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >Append</el-button>
          <el-button
            v-if="node.childNodes ==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog title="提示" :visible.sync="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="三级分类名称" >
          <el-input :v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCategory()">确 定</el-button>
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
        parentCid: "",
        catLevel: "",
        showStatus: "1",
        sort: "0"
        },
      //默认不弹框
      dialogVisible: false,
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
        console.log("数据", data.listTree);
        this.menuData = data.listTree;
      });
    },
    //新增分类
    append(data) {
      console.log("新增", data);
      this.dialogVisible = true;
      console.log("新增菜单",this.category)
    },
    addCategory(){
      console.log(this.category)
    },
    //删除分类
    remove(node, data) {
      console.log("删除", node, data);
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
            console.log("父节点", this.expandedKey);
          });
        })
        .catch(() => {});
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