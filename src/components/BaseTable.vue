<template>
  <div class="base-table">
    <el-table :data="tableData" style="width: 100%">
      <el-table-column
        show-overflow-tooltip
        v-for="(item, index) in columnList"
        :width="item.width"
        :key="index"
        :prop="item.prop"
        :label="item.label"
        :align="item.align ? item.align : 'center'"
      >
        <template slot-scope="scope">
          <!--            todo 操作-->
          <span v-if="!item.prop">
            <el-button
              v-if="isShowReport"
              @click="getReport(scope.row.id, $event)"
              size="mini"
              type="text"
              icon="el-icon-tickets"
              >智慧消防报告</el-button
            >
            <el-button
              @click="getDetail(scope.row.id, $event)"
              size="mini"
              type="text"
              icon="el-icon-search"
              >详情</el-button
            >
            <el-button
              v-if="isShowDelete"
              @click="deleteInfo(scope.row)"
              size="mini"
              type="text"
              icon="el-icon-delete"
              >删除</el-button
            >
          </span>
          <!--          todo 需要自定义内容slot]-->
          <span v-else-if="item.slot && scope.row[item.prop]">
            <el-button type="text" @click="getSlotDetail(scope.row, item.slot)"
              >{{ scope.row[item.prop] }}
              <span>{{ item.unit }}</span>
              <i class="el-icon-zoom-in"></i>
            </el-button>
          </span>
          <!--          todo 中文映射表map]-->
          <span
            v-else-if="item.map"
            :class="getName(scope.row[item.prop], item, 'className')"
          >
            <span v-if="scope.row[item.prop] != '部分离线'">
              {{ getName(scope.row[item.prop], item, "label") }}
            </span>
            <el-button
              @click="getOffline(scope.row)"
              style="color: #e6a23c !important;"
              v-else
              type="text"
              >{{ getName(scope.row[item.prop], item, "label") }}</el-button
            >
          </span>
          <!--          todo 默认数据-->
          <span
            v-else
            :class="classObject(item.badClass, scope.row[item.badKey])"
            >{{ scope.row[item.prop] | capitalize(item) }}</span
          >
        </template>
      </el-table-column>

      <!--      todo 无数据的展示页面-->
      <template slot="empty">
        暂无数据<i class="el-icon-brush"></i>
      </template>
    </el-table>
  </div>
</template>

<script>
// Todo VUE代码风格、拒绝硬编码！
/**
 *  作者：                                             时间：
 *  1,常量从js文件引入，不要定义魔术变量
 *  2,超过两次使用的方法定义为公共方法，import导入
 *  3,不涉及修改vuex中state中的值时候，不要在action中写方法
 *  4,css字体公用颜色，样式从文件中导入，个性化样式写在组件内
 *  5,定义全局标量来保存很多地方都要使用的值
 *  6,事先定义好基础的公共组件，全局注册
 */

import { isDelete } from "../plugins/commonJs";
export default {
  name: "BaseTable",
  // Todo: 组件注册
  components: {},
  // Todo: 特性
  props: {
    // todo 列表的每一列、
    columnList: {
      type: Array,
      require: true
    },
    tableData: {
      type: Array,
      require: true
    },
    isShowDelete: {
      type: Boolean,
      default: true
    },
    isShowReport: {
      type: Boolean,
      default: false
    }
  },
  // todo 过滤器
  filters: {
    // todo 对于没有值和为0的值显示为-、unit是单位名
    capitalize: function(val, item) {
      return val ? `${val}${item.unit ? item.unit : ""}` : "—";
    }
  },
  // Todo: 双向绑定的数据
  data() {
    return {
      // 检查结果映射
      checkResult: [
        {
          label: "未指定",
          value: 0
        },
        {
          label: "合格",
          value: 1,
          className: "normal"
        },
        {
          label: "现场整改",
          value: -1,
          className: "correctNow"
        },
        {
          label: "限期整改",
          value: -2,
          className: "correct"
        },
        {
          label: "停业整顿",
          value: -3,
          className: "shutDown"
        }
      ],
      // 网格事件列表状态的映射
      StreetStatus: [
        {
          label: "未指定",
          value: 0
        },
        {
          label: "待处理",
          value: 1,
          className: "shutDown"
        },
        {
          label: "处理中",
          value: 2,
          className: "correctNow"
        },
        {
          label: "已办结",
          value: 3,
          className: "normal"
        }
      ],
      // 网关状态状态的映射、因为有部分数据存在中文的状态而不是数字
      gatewayStatus: [
        {
          label: "未指定",
          value: 0
        },
        {
          label: "在线",
          value: 1,
          className: "normal"
        },
        {
          label: "离线",
          value: -1,
          className: "correctNow"
        },
        {
          label: "异常",
          value: -2,
          className: "shutDown"
        },
        {
          label: "部分离线",
          value: -3,
          className: "shutDown"
        }
      ],
      // 在线、离线样式表
      statusName: [
        {
          value: "离线",
          label: "离线",
          className: "correctNow"
        },
        {
          value: "在线",
          label: "在线",
          className: "normal"
        },
        {
          value: "部分离线",
          label: "部分离线",
          className: "correctNow"
        }
      ]
    };
  },
  // Todo: 计算属性
  computed: {},
  // Todo: 数据监听
  watch: {},
  // Todo: HTML 渲染前
  created: function() {},
  // Todo: HTML渲染后
  mounted: function() {},
  // Todo: 方法
  methods: {
    // todo
    /**
     *@fileOverview 获取中文 /获取样式 样式映射表，根据不同的类型和不同的值返回不同的class样式name
     * @param val 当前行对象的具体数字、可能是数字、中文、来赋上不同颜色
     * @param item columnList对象中的map匹配对象名称
     * @param nature 返回label中文名还是classname样式名
     */
    getName(val, item, nature) {
      let name = this[item.map].find(i => {
        return i.value === val;
      });
      return name[nature] ? name[nature] : "";
    },
    /**
     *@fileOverview 注册当前行某一列的事件
     * @param val 当前行数据对象
     * @param slotName 当前slot的名称
     */
    getSlotDetail(val, slotName) {
      this.$emit("slotDetail", val, slotName);
    },
    /**
     *@fileOverview 获取当前行对象详情
     * @param id 行对象id
     * @param event 原生点击事件
     */
    getDetail(id, event) {
      this.$emit("getDetail", id, event);
    },
    /**
     * @fileOverview 获取部分离线数据
     * @param item 行数据对象
     */
    getOffline(item) {
      console.log(item);
      this.$emit("getOffline", item);
    },
    /**
     * @fileOverview 获取部分数据的单独class、用法参考unitlist页面
     * @param item 返回的class名称
     * @param value 判断条件
     * @returns {*}
     */
    classObject(item, value) {
      if (value) {
        return item;
      }
    },
    /**
     * @fileOverview 获取报告详情
     * @param id 单位id
     * @param event 原生点击事件
     */
    getReport(id, event) {
      this.$emit("getReport", id, event);
    },
    /**
     * @fileOverview 删除当前行
     * @param val 行数据对象
     */
    deleteInfo(val) {
      isDelete()
        .then(() => {
          this.$emit("deleteInfo", val);
        })
        .catch(error => {
          console.log(error);
        });
    }
  }
};
</script>

<style lang="scss">
@import "../style/app-variables.scss";

.base-table {
  /* todo 背景色透明*/
  /*.el-table,*/
  /*.el-table__expanded-cell {*/
  /*  background-color: transparent ;*/
  /*}*/

  /*el-table th,*/
  /*.el-table tr {*/
  /*  background-color: transparent;*/
  /*}*/
  /*TODO 去掉行之间的线条*/
  .el-table th.is-leaf,
  .el-table td {
    border-bottom-width: 1px;
  }
  i {
    padding: 0 4px;
  }
  background-color: $table-body;
  /* TODO 设置表头样式*/
  .el-table th {
    background-color: $table-header;
    color: $font-white;
  }
  /*TODO table body*/
  tbody {
    /*todo 设置行高,因为有按钮把某些表行撑高了，为了统一所有表的行高*/
    .el-table,
    .cell {
      line-height: 40px;
    }
    color: $font-white;
    & > :nth-child(2n) {
      background-color: $table-body;
    }
    & > :nth-child(2n + 1) {
      background-color: $table-main;
    }
  }
  /*todo hover*/
  .el-table--enable-row-hover .el-table__body tr:hover > td {
    background-color: $table-header !important;
  }
  /*  todo 数据为空*/
  .el-table__empty-block {
    background-color: $table-body;
  }
}
/*  todo 中文样式映射表、为了最大的扩展性、虽然现在只存在颜色不同、但这种情况下可以添加各种不同的样式、比单独传一个颜色值要好的多*/
/* 单位列表部分名称显示为黄色*/
.badName {
  color: #e6a23c;
}
.normal {
  color: #67c23a;
}
.shutDown {
  color: #f56c6c;
}
.correct {
  color: chocolate;
}
.correctNow {
  color: #e6a23c;
}
.off-line {
  color: #f56c6c;
}
</style>
