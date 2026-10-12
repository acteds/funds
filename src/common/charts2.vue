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
      if (useLog) {
        for (let i = 0; i < values.length; i++) {
          const raw = values[i];
          if (raw === null || raw === undefined || raw === "") {
            continue;
          }
          const num = +raw;
          const base = isPercent ? 1 + num / 100 : num;
          if (!isFinite(num) || !(base > 0)) {
            useLog = false;
            break;
          }
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
    // 对数坐标的刻度：数字取等差的「整」值（净值 0.1/0.5/1/5…，收益 10%/50%/100%…），
    // 位置按对数落点，所以越往上越挤。返回 [{ coord, text }]。
    logTicks(min, max, isPercent) {
      const steps = isPercent
        ? [0.1, 0.2, 0.25, 0.5, 1, 2, 2.5, 5, 10, 20, 25, 50, 100, 200, 250, 500, 1000, 2000, 2500, 5000]
        : [0.001, 0.002, 0.0025, 0.005, 0.01, 0.02, 0.025, 0.05, 0.1, 0.2, 0.25, 0.5, 1, 2, 2.5, 5, 10, 20, 25, 50, 100, 200, 250, 500, 1000, 2000, 2500, 5000];
      const lo = min;
      const hi = max;
      let step = steps[steps.length - 1];
      for (let i = 0; i < steps.length; i++) {
        if ((hi - lo) / steps[i] <= 9) {
          step = steps[i];
          break;
        }
      }
      const ticks = [];
      for (let k = Math.ceil(lo / step); k * step <= hi; k++) {
        const val = k * step;
        const factor = isPercent ? 1 + val / 100 : val;
        ticks.push({
          coord: Math.log(factor) / Math.LN10,
          text: isPercent ? val.toFixed(1) + "%" : val.toFixed(3),
        });
      }
      return ticks;
    },
    // 对数模式下：隐藏原生刻度/网格，改用等差刻度（markLine 画线+标注）；
    // 关掉开关时恢复原生刻度。
    applyAxisMode(values, isPercent) {
      const onLog = this.useLog;
      const axis = this.option.yAxis;
      axis.axisLabel.show = !onLog;
      axis.axisTick = axis.axisTick || {};
      axis.axisTick.show = !onLog;
      axis.splitLine.show = !onLog;
      const series = this.option.series || [];
      if (!onLog || !series.length) {
        return;
      }
      let min = Infinity;
      let max = -Infinity;
      values.forEach((raw) => {
        const num = +raw;
        if (raw === null || raw === undefined || raw === "" || !isFinite(num)) {
          return;
        }
        min = Math.min(min, num);
        max = Math.max(max, num);
      });
      if (!isFinite(min) || !isFinite(max)) {
        return;
      }
      series[0].markLine = {
        silent: true,
        symbol: "none",
        animation: false,
        lineStyle: { type: "dashed", color: this.defaultColor },
        label: {
          show: true,
          position: "start",
          color: this.defaultColor,
          fontSize: 12,
          formatter: (p) => (p.data && p.data.text) || "",
        },
        data: this.logTicks(min, max, isPercent).map((tick) => ({
          yAxis: tick.coord,
          text: tick.text,
        })),
      };
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
            this.applyAxisMode(
              dataList
                .map((item) => +item.YIELD)
                .concat(dataList.map((item) => +item.INDEXYIED)),
              true
            );
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
          this.applyAxisMode(
            dataList
              .map((item) => +item.DWJZ)
              .concat(dataList.map((item) => +item.LJJZ))
          );
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
