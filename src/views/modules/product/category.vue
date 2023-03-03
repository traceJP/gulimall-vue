<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-show="draggable" size="mini" @click="batchSave">保存修改</el-button>
    <el-button size="mini" type="danger" @click="batchDelete">批量删除</el-button>

    <el-tree
      ref="menuTree"
      :data="menus"
      node-key="catId"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      :draggable="draggable"
      :default-expanded-keys="expandedKey"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
    >
      <!-- node element-ui封装的当前节点对象 包含对象的一些信息   data为当前节点的对象 -->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <!-- 三级菜单 第三层不允许添加子元素 -->
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <!-- 判断是否有子元素 否则不允许删除 推荐：使用node对象获取数据，因为data对象的变量名可能会被修改 -->
          <el-button
            v-if="data.children.length === 0"
            type="text"
            size="mini"
            @click="remove(node, data)"
          >
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <!-- 添加记录对话框 -->
    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
      width="30%"
    >
      <!-- 记录表单 -->
      <el-form ref="categoryForm" :model="category" label-width="80px">
        <el-form-item label="分类名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="图表">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="submitData">确 定</el-button>
          <el-button @click="dialogVisible = false">取 消</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // data
      menus: [],
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
      },

      // list config
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [],
      
      // dialog config
      dialogVisible: false,
      dialogTitle: "Unknow",
      dialogType: "", // 用于判断对话框的类型 - 是由于修改状态打开的还是由于添加状态打开的 edit | add

      // 拖拽配置
      draggable: false, // 是否允许拖拽
      maxLevel: 0, // 最大层级 判断是否允许拖拽
      updateNodes: [], // 收集所有变化过顺序的节点
      pCid: [], // 所有变化过顺序的节点的父级id，用于默认展示
    };
  },
  created() {
    this.getMenuList();
  },
  methods: {
    // =================================== BUTTON ===================================

    append(data) {
      this.dialogVisible = true;
      this.dialogType = "add";
      this.dialogTitle = "添加";

      // 刷新表单
      Object.assign(this.category, this.$options.data().category);

      // 默认值
      // 需要添加的记录的父级id 就是当前节点的id
      this.category.parentCid = data.catId;
      // 需要添加的记录的层级 就是当前节点的层级 + 1
      this.category.catLevel = data.catLevel + 1;
    },
    edit(data) {
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.dialogTitle = "修改";

      // 发送请求获取 当前节点 的 最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        // 默认值 - 回显 最新数据
        this.category = data.data;
      });
    },
    // 判断提交类型
    submitData() {
      const type = this.dialogType;
      switch (type) {
        case "add":
          this.addCategory();
          break;
        case "edit":
          this.editCategory();
          break;
      }
    },

    // ================================ 列表节点拖拽 =================================
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // draggingNode 当前拖拽的节点 ； dropNode 当前拖拽的节点的 *相对节点* ； type 拖拽的类型（inner内部 next下一个 prev上一个）
      // 如果是inner，则dropNode为父节点 ；  如果是next 则 dropNode为下一个节点 ； 如果是prev 则 dropNode为上一个节点
      // *** 注意： 拖动节点必须操作 element-ui 封装的对象，不能操作后端传过来的 data 数据 ， 因为会有批量操作的节点多次变化，data不是最新数据 ***

      // 当前节点的最新父节点id
      let pCid = 0;
      // 当前节点的父节点下的所有子节点
      let siblings = [];

      switch (dropType) {
        // inner时，父节点直接取 dropNode
        case "inner":
          pCid = dropNode.data.catId;
          siblings = dropNode.childNodes;
          break;
        // next | after时，父节点为 dropNode的父节点
        case "before":
        case "after":
          pCid = dropNode.parent.data.catId || 0; // 根节点的父节点默认为0
          siblings = dropNode.parent.childNodes;
      }

      // 修改全局的pCid
      this.pCid.push(pCid);

      // 获取当前拖拽节点的所有兄弟节点 的 最新顺序 - （动了一个节点，相当于周围所有节点的顺序都变了）
      for (let i in siblings) {
        // 判断是否是当前拖拽的节点 还需要修改自己的父节点id 因为就自己的层级变化了
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 修改当前节点的层级
          let catLevel = 0;
          if (siblings[i].level !== draggingNode.level) {
            // 当前层级是否发生变化 =》 siblings使用的是data的静态数据 ； draggingNode是element-ui的动态数据 ；判断两则是否一致则可以知道层级是否发生变化
            catLevel = siblings[i].level;
            // 将子节点的层级也一并修改
            this.updateChildNodeLevel(siblings[i]); // 当前节点
          }

          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
          });
        } else {
          // 其他兄弟节点 只是顺序变化了
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i, // 新的顺序值
          });
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode);
      console.log("maxLevel ======", this.maxLevel);
      // 拖拽的当前节点的深度 = 当前节点的深度 - 当前节点层级 + 1
      // eg. 当前节点的深度为 3（countNodeLevel方法遍历得出） ， 当前节点层级为 1 ， 拖拽到层级为 2 的节点； 则当前节点的深度为 3 - 1 + 1 = 3
      const deep = Math.abs(this.maxLevel - draggingNode.level) + 1;

      // 判断正在拖动的节点深度 + 父节点深度 不大于 3 即可
      switch (type) {
        // inner时，父节点直接取 dropNode
        case "inner":
          return dropNode.level + deep <= 3; // eg. 3 + 2 = 5 返回 false
        // next | prev时，父节点为 dropNode的父节点
        case "next":
        case "prev":
          return dropNode.parent.level + deep <= 3;
      }
    },

    // 递归找到所有 后代 节点（不包括自己） 求出最大深度
    countNodeLevel(node) {
      // 找当前节点下的子节点的最大 catLevel
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i in node.childNodes) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          // 递归找
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },

    // 递归修改所有 后代 节点（不包括自己） 的层级
    updateChildNodeLevel(node) {
      for (let i in node.childNodes) {
        const cNode = node.childNodes[i].data;
        this.updateNodes.push({
          catId: cNode.catId,
          catLevel: cNode.catLevel,
        });
        // 递归修改
        this.updateChildNodeLevel(node.childNodes[i]);
      }
    },

    // ================================ HTTP CRUD ================================

    getMenuList() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.data;
        console.log(this.menus);
      });
    },
    remove(node, data) {
      const ids = [data.catId];

      // check
      this.$confirm(`是否删除【 ${data.name} 】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          // request delete
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            console.log(data);
            if (data.code === 0) {
              // tips
              this.$message({
                message: "删除成功",
                type: "success",
                duration: 1500,
              });
              // 刷新菜单
              this.getMenuList();
              // 更新默认选中的菜单为删除后的菜单 - 通过 element-ui 的 node 对象 获取 父节点属性
              // 默认节点值是根据 node-key 进行判断的
              this.expandedKey = [node.parent.data.catId];
            }
          });
        })
        .catch();
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category),
      }).then(({ data }) => {
        if (data.code === 0) {
          this.$message({
            message: "添加成功",
            type: "success",
            duration: 1500,
          });
          this.dialogVisible = false;
          this.getMenuList();
          // 当前的 expandedKey 被 add 默认值
          this.expandedKey = [this.category.parentCid];
        }
      });
    },
    editCategory() {
      // 解构出需要修改的参数 - 有些参数不允许修改 需要排除
      const { catId, name, icon, productUnit } = this.category;
      console.log("即将进行修改请求", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        if (data.code === 0) {
          this.$message({
            message: "修改成功",
            type: "success",
            duration: 1500,
          });
          this.dialogVisible = false;
          this.getMenuList();
          // 当前的 expandedKey 被 edit 修改成了当前节点的值 所以可以通过 expandedKey 定位到 当前所选择的节点
          this.expandedKey = [this.category.parentCid];
        }
      });
    },

    batchSave() {
      // 发送请求将变动的节点提交修改
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        if (data.code === 0) {
          this.$message({
            message: "修改顺序成功",
            type: "success",
            duration: 1500,
          });
          this.getMenuList();
          this.expandedKey = this.pCid;
          // 恢复拖拽计算的默认值
          this.maxLevel = 0;
          this.updateNodes = [];
          this.pCid = [];
        }
      });
    },

    batchDelete() {
      const checkedNodes = this.$refs['menuTree'].getCheckedNodes();
      let ids = [];
      let name = [];
      checkedNodes.map(item => {
        ids.push(item.catId);
        name.push(item.name);
      });
      console.log(ids);
      this.$confirm(`是否删除【 ${name} 】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          // request delete
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            console.log(data);
            if (data.code === 0) {
              // tips
              this.$message({
                message: "菜单批量删除成功",
                type: "success",
                duration: 1500,
              });
              this.getMenuList();
            }
          });
        })
        .catch();

    },

  },
  
};
</script>

<style scoped>
</style>
