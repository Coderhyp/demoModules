<template>
  <div>
    <el-input
      v-show="showSearch"
      v-model="filterText"
      :placeholder="searchPlaceholder"
      class="gt-tree__search"
      clearable
      suffix-icon="el-icon-search"
    ></el-input>
    <div
      ref="scroller"
      :style="{ height: option.height - 28 + 'px', width: option.width + 'px' }"
      class="gt-tree"
      @scroll="handleScroll"
    >
      <div
        v-if="visibleData.length"
        :style="{ transform: `translateY(${offset}px)` }"
        class="gt-tree__content"
      >
        <div
          v-for="item in visibleData"
          :key="item[nodeKey.id]"
          :style="{
            paddingLeft: 23 * (item.level - 1) + 'px',
            height: option.itemHeight + 'px',
          }"
          class="gt-tree__list-container"
          @click="handerClick(item)"
        >
          <i
            :class="[
              item.expand ? 'gt-tree__expand' : 'gt-tree__close',
              {
                hidden: !(
                  item[nodeKey.children] && item[nodeKey.children].length
                ),
              },
            ]"
            @click.stop="toggleExpand(item)"
          />
          <!-- <span v-show="!(item[nodeKey.children] && item[nodeKey.children].length)" :style="`margin-left: ${-5 + 5 * item.level}px`"></span> -->
          <div
            :class="[
              'checkbox',
              {
                'checkbox-checked': item.checked,
                'checkbox-disabled': item.disabled,
              },
            ]"
          >
            <i v-show="item.checked" :class="['el-icon-check']"></i>
          </div>
          <i
            :class="[
              'gt-tree__content__icon',
              {
                'gt-tree__content__icon__color': item.checked,
                'label-disabled': item.disabled,
              },
            ]"
            :name="`${item.type}`"
          ></i>
          <span
            :class="[
              'gt-tree__content__label',
              {
                'gt-tree__content__icon__color': item.checked,
                'label-disabled': item.disabled,
              },
            ]"
            :title="item[nodeKey.label]"
            >{{ item[nodeKey.label] }}</span
          >
        </div>
      </div>
      <div v-else class="gt-tree__noData">暂无数据</div>
    </div>
  </div>
</template>
<script>
import debounce from "lodash/debounce.js";

let lastTime = 0;
export default {
  name: "TreePart",
  props: {
    fetch: { type: Function, default: () => {} },
    defaultExpand: {
      // 是否默认展开
      type: Boolean,
      required: false,
      default: false,
    },
    timeout: {
      // 刷新频率
      type: Number,
      default: 5,
    },
    nodeKey: {
      type: Object,
      default: () => {
        return { children: "children", label: "label", id: "id" };
      },
    },
    option: {
      // tree配置对象
      type: Object,
      default: () => {
        return {
          height: 500, // 滚动容器的高度
          width: 268,
          itemHeight: 25, // 单个item的高度
        };
      },
    },
    treeData: {
      type: Array,
      default() {
        return [];
      },
    },
    multiple: { type: Boolean, default: false },
    maxChoose: { type: Number, default: 5 },
    showSearch: { type: Boolean, default: true },
    searchPlaceholder: { type: String, default: "请输入" },
    hasCrowd: { type: Boolean, default: true },
  },
  data() {
    return {
      loading: false,
      tree: [],
      contentHeight: "0px",
      offset: 0, // 偏移量
      visibleData: [], // 可视数组
      chooseData: [], // 已选择数组
      filterText: "",
    };
  },
  computed: {
    visibleCount() {
      // 获得滚动区域的item的数量
      return Math.floor(this.option.height / this.option.itemHeight);
    },
    flattenTree() {
      const children = this.nodeKey.children || "children";
      const choosedData = this.chooseData;
      const flatten = function (
        list,
        childKey = children,
        level = 1,
        parent = null,
        defaultExpand = true
      ) {
        let arr = [];
        list.forEach((item) => {
          item.level = level;
          item.parent = parent;
          if (item.expand === undefined) {
            item.expand = defaultExpand;
          }
          if (item.visible === undefined) {
            item.visible = true;
          }
          if (item.parent && !item.parent.expand) {
            item.visible = false;
          }
          item.checked = false;
          if (
            choosedData.length &&
            choosedData.findIndex((i) => i.id === item.id) > -1
          ) {
            // 单选  需递归
            list.forEach((itm)=>{
              itm.checked =false
            })
            item.checked = true;
          }

          arr.push(item);
          if (item[childKey]) {
            arr.push(
              ...flatten(
                item[childKey],
                childKey,
                level + 1,
                item,
                defaultExpand
              )
            );
          }
        });
        console.log(choosedData);

        return arr;
      };
      return flatten(this.tree, children, 1, null, this.defaultExpand);
    },
  },
  watch: {
    filterText(val) {
      this.onSearchTree(val);
    },
    treeData(val) {
      if (val.length) {
        this.filterTreeData(val);
        this.updateView();
      }
    },
    deep: true,
  },
  mounted() {
    if (this.treeData.length) {
      this.filterTreeData(this.treeData);
      this.updateView();
    } else {
      this.fetch && this.fetchTreeData();
    }
  },
  methods: {
    fetchTreeData(searchValue) {
      this.loading = true;
      this.fetch(searchValue)
        .then((res) => {
          this.loading = false;
          this.filterTreeData(res);
          this.updateView();
        })
        .catch(() => {
          this.loading = false;
          this.tree = [];
          this.updateView();
        });
    },
    filterTreeData(data) {
      const hasCrowd = this.hasCrowd;
      const flatten = function (data) {
        let list = [];
        data.forEach((item) => {
          if (!hasCrowd || item.children || item.crowd_library) {
            item.type = "group";
            item.disabled = item.disabled || hasCrowd;
            item.children = item.children
              ? [...item.children, ...(item.crowd_library || [])]
              : item.crowd_library;
          } else {
            item.type = "group-sort";
            item.disabled = false;
          }
          list.push({
            id: item.id,
            label: item.label,
            children: item.children ? flatten(item.children) : null,
            type: item.type,
            disabled: item.disabled,
          });
        });
        return list;
      };
      this.tree = flatten(data);
    },
    updateView() {
      // 初始化
      this.getContentHeight();
      this.$emit("update", this.tree);
      this.handleScroll();
    },
    getContentHeight() {
      // 获得真实内容高度
      this.contentHeight =
        (this.flatten || []).filter((item) => item.visible).length *
          this.option.itemHeight +
        "px";
    },
    handleScroll() {
      // 触发滚动
      let currentTime = +new Date();
      if (currentTime - lastTime > this.timeout) {
        this.updateVisibleData(this.$refs.scroller.scrollTop);
        lastTime = currentTime;
      }
    },
    updateVisibleData(scrollTop = 0) {
      // 根据滚动scrollTop计算出可视数据并更新
      let start =
        Math.floor(scrollTop / this.option.itemHeight) -
        Math.floor(this.visibleCount / 2);
      start = start < 0 ? 0 : start;
      const end = start + this.visibleCount * 2;
      const allVisibleData = (this.flattenTree || []).filter(
        (item) => item.visible
      );
      this.visibleData = allVisibleData.slice(start, end);
      this.offset = start * this.option.itemHeight;
    },
    toggleExpand(item) {
      // 点击节点
      const isExtend = item.expand;
      this[isExtend ? "collapse" : "expand"](item, true);
      this.updateView();
    },
    collapse(item) {
      // 折叠节点
      item.expand = false;
      this.recursionVisible(item[this.nodeKey.children], false);
    },
    expand(item) {
      // 展开节点
      item.expand = true;
      this.recursionVisible(item[this.nodeKey.children], true);
    },
    recursionVisible(children, status) {
      // 节点visible处理
      children.forEach((node) => {
        node.visible = status;
        node.expand = status;
        if (node[this.nodeKey.children]) {
          this.recursionVisible(node[this.nodeKey.children], status);
        }
        // node.checked = !node.checked
        // this.$emit('current-change', this.chooseData)
        // this.updateView()
      });
    },
    onSearchTree: debounce(function (val) {
      this.fetchTreeData(val);
    }, 300),
    handerClick(item) {
      if (this.hasCrowd && item.children) {
        this.toggleExpand(item);
        return;
      }
      // 临时特殊处理其他管控群体
      if (item.id === "custom_other_staffManage") {
        this.toggleExpand(item);
        return;
      }
      if (item.checked) {
        console.log(this.chooseData,item);
        this.chooseData.splice(
          this.chooseData.findIndex((i) => i.id === item.id),
          1
        );
      } else {
        if (this.maxChoose <= this.chooseData.length) {
          this.$message({
            type: "error",
            message: `群体选择数量不超过${this.maxChoose}个`,
          });
          item.checked = false;
          this.updateView();
          return;
        }
        if (this.multiple) {
          this.chooseData = [item]
        } else {
          this.chooseData = [...this.chooseData,item]
        }
      }
      // if (this.multiple) {
        // this.chooseData.forEach((e,i) => {
        //   this.$set(this.chooseData[i], 'checked', false)
        //   console.log(this.chooseData[i]);
        // });
        // this.chooseData[0].checked =false
        // console.log(this.chooseData[0].checked);
        // console.log(this.chooseData); 

        // item.checked = !item.checked;
        // this.updateView();
        // return
      // }
      // console.log(this.chooseData);
      item.checked = !item.checked;
      this.$emit("current-change", this.chooseData);
      this.updateView();
    },
    setCheckedNodes(item) {
      this.chooseData = item;
      this.updateView();
    },
  },
};
</script>
<style lang="less" scoped>
@color: #515a6e;
@iconColor: #5f82ff;

.gt-tree {
  background-color: #fff;
  border: 1px solid #dadada;
  padding-top: 20px;
  margin-right: 5px;
  position: relative;
  overflow-y: scroll;

  &::-webkit-scrollbar {
    width: 0px;
    height: 100%;
  }

  &__contentHeight {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    z-index: -1;
  }

  &__search {
    padding: 10px;
    width: 100%;
    position: sticky;
    top: 0px;
    background: #fff;
    z-index: 100;
  }

  &__noData {
    text-align: center;
    padding: 20px;
  }

  &__content {
    position: absolute;
    top: 10px;
    min-height: 100px;

    .checkbox {
      display: flex;
      align-items: center;
      justify-content: center;
      width: 16px;
      height: 16px;
      background: #fff;
      border: 1px solid #d9d9d9;
      border-radius: 2px;
      margin: 0 6px;

      .el-icon-check {
        color: #fff;
        font-size: 12px;
        font-weight: border;
      }
    }

    .checkbox-checked {
      background: @iconColor;
      border: none;
    }

    .checkbox-disabled {
      background-color: #edf2fc;
      border-color: #dcdfe6;
      cursor: not-allowed;
    }

    .label-disabled {
      color: #c0c4cc;
      cursor: not-allowed;
    }

    &__icon {
      font-size: 14px;
      color: @color;
      margin: 0 8px;
    }

    &__icon__color {
      color: @iconColor;
    }

    &__label {
      flex: 1;
      color: @color;
      font-size: 14px;
      white-space: nowrap;
      text-overflow: ellipsis;
      overflow: hidden;
    }
  }

  &__list-container {
    display: flex;
    align-items: center;
    margin: 10px 0;
    cursor: pointer;
  }

  &__content__item {
    box-sizing: border-box;
    display: flex;
    position: relative;
    justify-content: center;
    align-items: center;
    cursor: pointer;

    &:hover,
    &__selected {
      background-color: #d7d7d7;
    }

    &__icon {
      position: absolute;
      left: 0;
      color: #c0c4cc;
      z-index: 10;
    }
  }

  &__icon__hidden {
    display: none;
  }

  &__close,
  &__expand {
    display: inline-block;
    width: 0;
    height: 0;
    overflow: hidden;
    font-size: 0;
    margin-right: 10px;
    border-width: 6px;
  }

  &__close {
    border-color: transparent transparent transparent #808695;
    border-style: dashed dashed dashed solid;
  }

  &__expand {
    border-color: #808695 transparent transparent transparent;
    border-style: solid dashed dashed dashed;
    margin-top: 5px;
  }
  .hidden {
    visibility: hidden;
  }
}
</style>