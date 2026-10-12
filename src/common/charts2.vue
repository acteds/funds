<template>
  <div
    class="box"
    v-loading="loading"
    :element-loading-background="
      darkMode ? 'rgba(0, 0, 0, 0.9)' : 'rgba(255, 255, 255, 0.9)'
    "
  >
    <div class="main-echarts" ref="mainCharts"></div>
    <div>
      <el-radio-group v-model="sltTimeRange" @change="changeTimeRange">
        <el-radio-button label="y">月</el-radio-button>
        <el-radio-button label="3y">季</el-radio-button>
        <el-radio-button label="6y">半年</el-radio-button>
        <el-radio-button label="n">一年</el-radio-button>
        <el-radio-button label="3n">三年</el-radio-button>
        <el-radio-button label="5n">五年</el-radio-button>
      </el-radio-group>
    </div>
  </div>
</template>

<script>
let echarts = require("echarts/lib/echarts");

import "./js/customed.js";
import "./js/dark.js";

require("echarts/lib/chart/line");

require("echarts/lib/component/tooltip");
require("echarts/lib/component/legend");

export default {
  name: "chatrs",
  props: {
    darkMode: {
      type: Boolean,
      default: false,
    },
    fund: {
      type: Object,
      required: true,
    },
    chartType: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      chartEL: null,
      myChart: null,
      minVal: null,
      maxVal: null,
      interVal: null,
      sltTimeRange: "y",
      chartTypeList: {
        DWJZ: {
          name: "单位净值",
        },
        LJJZ: {
          name: "累计净值",
        },
      },
      option: {},
      loading: false,
      logYAxis: false, // 设置项：Y 轴是否使用对数坐标
      useLog: false, // 当前图表实际是否按对数坐标绘制
    };
  },

  watch: {},
  computed: {
    defaultColor() {
      return this.darkMode ? "rgba(255,255,255,0.6)" : "#ccc";
    },
  },
  mounted() {
    this.init();
  },
  beforeDestroy() {
    this.myChart.clear();
  },
  methods: {
    init() {
      this.chartEL = this.$refs.mainCharts;
      this.myChart = echarts.init(
        this.chartEL,
        this.darkMode ? "dark" : "customed"
      );
      this.option = {
        tooltip: {
          trigger: "axis",
          formatter: (p) => {
            return `时间：${p[0].name}<br />${
              this.chartTypeList[this.chartType].name
            }：${p[0].value}`;
          },
        },
        grid: {
          top: 30,
          bottom: 30,
          left: 60
        },
        xAxis: {
          type: "category",
          data: [],
          axisLabel: {},
        },
        yAxis: {
          type: "value",
          scale: true,
          axisLabel: {
            color: this.defaultColor,
            formatter: (val) => this.fmtAxis(val),
          },
          splitLine: {
            show: true,
            lineStyle: {
              type: "dashed",
              color: this.defaultColor,
            },
          },
          data: [],
        },
        series: [
          {
            type: "line",
            data: [],
          },
        ],
      };
      // 先取设置再拉数据，保证首帧的坐标系就是用户选择的
      chrome.storage.sync.get("logYAxis", (res) => {
        this.logYAxis = !!res.logYAxis;
        this.getData();
      });
    },
    changeTimeRange(val) {
      this.getData();
    },
    handle_num_range(data) {
      var _aa = Math.max.apply(null, data);
      var _bb = Math.min.apply(null, data);
      return [_aa, _bb];
    },
    // ---------------- 对数坐标（设置中可开关） ----------------
    // 历史净值：坐标取 log10(净值)，标签与提示还原成净值。
    // 累计收益是百分比曲线（起点 0%、经常为负），对其净值倍数 1+涨幅/100 取对数，
    // 标签与提示仍显示百分比。
    // 数据中出现无法取对数的值（<=0）时，该图自动保持原来的线性坐标。
    setLogAxis(values, isPercent) {
      let useLog = !!this.logYAxis;
      for (let i = 0; useLog && i < values.length; i++) {
        const raw = values[i];
        if (raw === null || raw === undefined || raw === "") {
          continue;
        }
        const num = +raw;
        // 缺失/非数字的值按断点处理，不影响坐标类型
        if (!isFinite(num)) {
          continue;
        }
        const base = isPercent ? 1 + num / 100 : num;
        if (!(base > 0)) {
          useLog = false;
        }
      }
      this.useLog = useLog;
    },
    axisVal(value, isPercent) {
      const num = +value;
      if (!this.useLog) {
        return num;
      }
      if (
        value === null ||
        value === undefined ||
        value === "" ||
        !isFinite(num)
      ) {
        return null;
      }
      const base = isPercent ? 1 + num / 100 : num;
      return base > 0 ? Math.log(base) / Math.LN10 : null;
    },
    valOf(value) {
      return this.useLog
        ? Math.round(Math.pow(10, +value) * 1e6) / 1e6
        : +value;
    },
    pctOf(value) {
      return this.useLog
        ? Math.round((Math.pow(10, +value) - 1) * 1e4) / 100
        : +value;
    },
    fmtAxis(value) {
      return this.chartType == "LJSY"
        ? this.pctOf(value).toFixed(1) + "%"
        : this.valOf(value).toFixed(3);
    },
    getData() {
      this.loading = true;
      if (this.chartType == "LJSY") {
        let url = `https://fundmobapi.eastmoney.com/FundMApi/FundYieldDiagramNew.ashx?FCODE=${
          this.fund.fundcode
        }&RANGE=${
          this.sltTimeRange
        }&deviceid=Wap&plat=Wap&product=EFund&version=2.0.0&_=${new Date().getTime()}`;
        this.$axios.get(url).then((res) => {
          this.loading = false;
          let dataList = res.data.Datas;
          if (dataList) {
            this.setLogAxis(
              dataList
                .map((item) => +item.YIELD)
                .concat(dataList.map((item) => +item.INDEXYIED)),
              true
            );
            this.option.legend = {
              show: true,
            };
            this.option.tooltip.formatter = (p) => {
              let str =
                p.length > 1
                  ? `<br />${p[1].seriesName}：${this.pctOf(p[1].value)}%`
                  : "";
              return `时间：${p[0].name}<br />${p[0].seriesName}：${this.pctOf(
                p[0].value
              )}%${str}`;
            };
            this.option.series = [
              {
                type: "line",
                name: "涨幅",
                data: dataList.map((item) => this.axisVal(item.YIELD, true)),
              },
              {
                type: "line",
                name: res.data.Expansion.INDEXNAME,
                data: dataList.map((item) => this.axisVal(item.INDEXYIED, true)),
              },
            ];
            this.option.xAxis.data = dataList.map((item) => item.PDATE);
            this.myChart.setOption(this.option);
          }
        });
      } else {
        let url = `https://fundmobapi.eastmoney.com/FundMApi/FundNetDiagram.ashx?FCODE=${
          this.fund.fundcode
        }&RANGE=${
          this.sltTimeRange
        }&deviceid=Wap&plat=Wap&product=EFund&version=2.0.0&_=${new Date().getTime()}`;
        this.$axios.get(url).then((res) => {
          this.loading = false;
          let dataList = res.data.Datas;
          this.setLogAxis(
            dataList
              .map((item) => +item.DWJZ)
              .concat(dataList.map((item) => +item.LJJZ))
          );
          this.option.series = [
            {
              type: "line",
              name: "单位净值",
              data: dataList.map((item) => this.axisVal(item.DWJZ)),
            },
            {
              type: "line",
              name: "累计净值",
              data: dataList.map((item) => this.axisVal(item.LJJZ)),
            },
          ];
          this.option.tooltip.formatter = (p) => {
            let str =
              p.length > 1
                ? `<br />${p[1].seriesName}：${this.valOf(p[1].value)}`
                : "";
            return `时间：${p[0].name}<br />${p[0].seriesName}：${this.valOf(
              p[0].value
            )}${str}<br />日增长率：${dataList[p[0].dataIndex].JZZZL}%`;
          };
          this.option.legend = {
            show: true,
          };
          this.option.xAxis.data = dataList.map((item) => item.FSRQ);
          this.myChart.setOption(this.option);
        });
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.box {
  width: 100%;
  height: 100%;
}
.main-echarts {
  width: 100%;
  height: 232px;
}
</style>
