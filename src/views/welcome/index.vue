<template>
  <div class="spc-charts-container">
    <div class="controls-panel">
      <el-card shadow="never" class="mb-4">
        <div class="flex items-center gap-4">
          <span class="text-sm font-medium">数据点数量:</span>
          <el-input-number
            v-model="dataPointsCount"
            :min="10"
            :max="100"
            :step="5"
            size="small"
            :disabled="isSimulating"
            @change="updateCharts"
          />
          <span class="text-sm font-medium">模拟速度:</span>
          <el-input-number
            v-model="simulationSpeed"
            :min="500"
            :max="5000"
            :step="100"
            size="small"
            :disabled="isSimulating"
          />
          <span class="text-xs text-gray-400">毫秒</span>
          <el-button type="primary" size="small" @click="generateNewData">
            重新生成数据
          </el-button>
          <el-button
            :type="isSimulating ? 'danger' : 'success'"
            size="small"
            @click="toggleSimulation"
          >
            {{ isSimulating ? "停止模拟" : "开始模拟" }}
          </el-button>
          <span v-if="isSimulating" class="text-sm text-gray-500 ml-2">
            正在接收新数据... ({{ simulationSpeed }}ms间隔)
          </span>
        </div>
      </el-card>
    </div>

    <!-- 均值-标准偏差控制图 -->
    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">均值-标准偏差控制图 (X̄-S Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>均值图 (X̄ Chart)</h4>
            <div ref="xBarChart" class="chart" />
          </div>
          <div class="sub-chart">
            <h4>标准偏差图 (S Chart)</h4>
            <div ref="sChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <!-- 均值-极差控制图 -->
    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">均值-极差控制图 (X̄-R Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>均值图 (X̄ Chart)</h4>
            <div ref="xBarChart2" class="chart" />
          </div>
          <div class="sub-chart">
            <h4>极差图 (R Chart)</h4>
            <div ref="rChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <!-- 单值-移动极差控制图 -->
    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">单值-移动极差控制图 (I-MR Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>单值图 (Individual Chart)</h4>
            <div ref="iChart" class="chart" />
          </div>
          <div class="sub-chart">
            <h4>移动极差图 (Moving Range Chart)</h4>
            <div ref="mrChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick, onBeforeUnmount } from "vue";
import * as echarts from "echarts";
import type { ECharts } from "echarts";

// 响应式数据
const dataPointsCount = ref(30);
const simulationSpeed = ref(1000); // 毫秒
const isSimulating = ref(false);
let simulationTimer: NodeJS.Timeout | null = null;
const xBarChart = ref<HTMLElement>();
const sChart = ref<HTMLElement>();
const xBarChart2 = ref<HTMLElement>();
const rChart = ref<HTMLElement>();
const iChart = ref<HTMLElement>();
const mrChart = ref<HTMLElement>();

// 图表实例
let xBarChartInstance: ECharts | null = null;
let sChartInstance: ECharts | null = null;
let xBarChart2Instance: ECharts | null = null;
let rChartInstance: ECharts | null = null;
let iChartInstance: ECharts | null = null;
let mrChartInstance: ECharts | null = null;

// 数据存储
const rawData = ref<number[][]>([]);
const individualData = ref<number[]>([]);

// 生成随机数据
const generateSampleData = (count: number, sampleSize: number = 5) => {
  const data: number[][] = [];
  for (let i = 0; i < count; i++) {
    const sample: number[] = [];
    for (let j = 0; j < sampleSize; j++) {
      // 生成正态分布数据 (均值=100, 标准差=5)
      sample.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
    }
    data.push(sample);
  }
  return data;
};

// 生成单值数据
const generateIndividualData = (count: number) => {
  const data: number[] = [];
  for (let i = 0; i < count; i++) {
    data.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
  }
  return data;
};

// 计算统计值
const calculateXBar = (samples: number[][]) => {
  return samples.map(
    sample => sample.reduce((sum, val) => sum + val, 0) / sample.length
  );
};

const calculateS = (samples: number[][]) => {
  return samples.map(sample => {
    const mean = sample.reduce((sum, val) => sum + val, 0) / sample.length;
    const variance =
      sample.reduce((sum, val) => sum + Math.pow(val - mean, 2), 0) /
      (sample.length - 1);
    return Math.sqrt(variance);
  });
};

const calculateR = (samples: number[][]) => {
  return samples.map(sample => Math.max(...sample) - Math.min(...sample));
};

const calculateMR = (data: number[]) => {
  const mr: number[] = [];
  for (let i = 1; i < data.length; i++) {
    mr.push(Math.abs(data[i] - data[i - 1]));
  }
  return mr;
};

// 创建控制图通用配置
const createControlChartOption = (
  data: number[],
  centerLine: number,
  ucl: number,
  lcl: number,
  title: string,
  yAxisName: string
) => {
  const xAxisData = data.map((_, index) => index + 1);

  return {
    title: {
      text: title,
      left: "center",
      textStyle: {
        fontSize: 14,
        fontWeight: "normal"
      }
    },
    tooltip: {
      trigger: "axis",
      formatter: (params: any) => {
        const point = params[0];
        return `点 ${point.axisValue}: ${point.value.toFixed(3)}`;
      }
    },
    legend: {
      show: false
    },
    grid: {
      left: "8%",
      right: "4%",
      bottom: "15%",
      top: "15%",
      containLabel: true
    },
    xAxis: {
      type: "category",
      data: xAxisData,
      name: "样本序号",
      nameLocation: "middle",
      nameGap: 25,
      axisLine: {
        lineStyle: { color: "#666" }
      }
    },
    yAxis: {
      type: "value",
      name: yAxisName,
      nameRotate: 90,
      nameLocation: "middle",
      nameGap: 40,
      axisLine: {
        lineStyle: { color: "#666" }
      }
    },
    series: [
      {
        name: "数据点",
        type: "line",
        data: data,
        symbol: "circle",
        symbolSize: 6,
        lineStyle: {
          color: "#1890ff",
          width: 2
        },
        itemStyle: {
          color: "#1890ff"
        }
      },
      {
        name: "中心线",
        type: "line",
        data: new Array(data.length).fill(centerLine),
        lineStyle: {
          color: "#52c41a",
          width: 2,
          type: "solid"
        },
        symbol: "none",
        emphasis: {
          disabled: true
        }
      },
      {
        name: "上控制限",
        type: "line",
        data: new Array(data.length).fill(ucl),
        lineStyle: {
          color: "#ff4d4f",
          width: 2,
          type: "dashed"
        },
        symbol: "none",
        emphasis: {
          disabled: true
        }
      },
      {
        name: "下控制限",
        type: "line",
        data: new Array(data.length).fill(lcl),
        lineStyle: {
          color: "#ff4d4f",
          width: 2,
          type: "dashed"
        },
        symbol: "none",
        emphasis: {
          disabled: true
        }
      }
    ]
  };
};

// 更新图表数据
const updateCharts = () => {
  // 保持后端数据，舍弃前端数据
  const currentRawData = rawData.value.slice(-dataPointsCount.value);
  const currentIndividualData = individualData.value.slice(
    -dataPointsCount.value
  );

  // 计算统计值
  const xBarData = calculateXBar(currentRawData);
  const sData = calculateS(currentRawData);
  const rData = calculateR(currentRawData);
  const mrData = calculateMR(currentIndividualData);

  // 计算控制限 (简化计算，实际应用中需要更精确的统计常数)
  const xBarMean =
    xBarData.reduce((sum, val) => sum + val, 0) / xBarData.length;
  const sMean = sData.reduce((sum, val) => sum + val, 0) / sData.length;
  const rMean = rData.reduce((sum, val) => sum + val, 0) / rData.length;
  const iMean =
    currentIndividualData.reduce((sum, val) => sum + val, 0) /
    currentIndividualData.length;
  const mrMean = mrData.reduce((sum, val) => sum + val, 0) / mrData.length;

  // 更新X̄-S图
  if (xBarChartInstance) {
    xBarChartInstance.setOption(
      createControlChartOption(
        xBarData,
        xBarMean,
        xBarMean + (3 * sMean) / Math.sqrt(5), // 简化的控制限计算
        xBarMean - (3 * sMean) / Math.sqrt(5),
        "",
        "均值"
      )
    );
  }

  if (sChartInstance) {
    sChartInstance.setOption(
      createControlChartOption(
        sData,
        sMean,
        sMean * 2.089, // 简化的B4系数
        sMean * 0, // 简化的B3系数
        "",
        "标准偏差"
      )
    );
  }

  // 更新X̄-R图
  if (xBarChart2Instance) {
    xBarChart2Instance.setOption(
      createControlChartOption(
        xBarData,
        xBarMean,
        xBarMean + 0.577 * rMean, // A2系数
        xBarMean - 0.577 * rMean,
        "",
        "均值"
      )
    );
  }

  if (rChartInstance) {
    rChartInstance.setOption(
      createControlChartOption(
        rData,
        rMean,
        rMean * 2.114, // D4系数
        rMean * 0, // D3系数
        "",
        "极差"
      )
    );
  }

  // 更新I-MR图
  if (iChartInstance) {
    iChartInstance.setOption(
      createControlChartOption(
        currentIndividualData,
        iMean,
        iMean + 2.66 * mrMean,
        iMean - 2.66 * mrMean,
        "",
        "单值"
      )
    );
  }

  if (mrChartInstance) {
    mrChartInstance.setOption(
      createControlChartOption(
        mrData,
        mrMean,
        mrMean * 3.267, // D4系数 for n=2
        mrMean * 0, // D3系数 for n=2
        "",
        "移动极差"
      )
    );
  }
};

// 生成新数据
const generateNewData = () => {
  rawData.value = generateSampleData(dataPointsCount.value * 2); // 生成更多数据以支持切片
  individualData.value = generateIndividualData(dataPointsCount.value * 2);
  updateCharts();
};

// 添加单个新数据点
const addNewDataPoint = () => {
  // 为原始数据添加新样本
  const newSample: number[] = [];
  for (let j = 0; j < 5; j++) {
    newSample.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
  }
  rawData.value.push(newSample);

  // 为单值数据添加新数据点
  const newIndividualValue =
    100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5;
  individualData.value.push(newIndividualValue);

  // 保持数据在合理范围内，舍弃前端保留后端
  if (rawData.value.length > dataPointsCount.value * 3) {
    rawData.value = rawData.value.slice(-dataPointsCount.value * 2);
  }
  if (individualData.value.length > dataPointsCount.value * 3) {
    individualData.value = individualData.value.slice(
      -dataPointsCount.value * 2
    );
  }

  updateCharts();
};

// 开始/停止模拟
const toggleSimulation = () => {
  if (isSimulating.value) {
    // 停止模拟
    if (simulationTimer) {
      clearInterval(simulationTimer);
      simulationTimer = null;
    }
    isSimulating.value = false;
  } else {
    // 开始模拟
    isSimulating.value = true;
    simulationTimer = setInterval(() => {
      addNewDataPoint();
    }, simulationSpeed.value);
  }
};

// 初始化图表
const initCharts = async () => {
  await nextTick();

  if (xBarChart.value) {
    xBarChartInstance = echarts.init(xBarChart.value);
  }
  if (sChart.value) {
    sChartInstance = echarts.init(sChart.value);
  }
  if (xBarChart2.value) {
    xBarChart2Instance = echarts.init(xBarChart2.value);
  }
  if (rChart.value) {
    rChartInstance = echarts.init(rChart.value);
  }
  if (iChart.value) {
    iChartInstance = echarts.init(iChart.value);
  }
  if (mrChart.value) {
    mrChartInstance = echarts.init(mrChart.value);
  }

  // 窗口大小变化时重新调整图表大小
  window.addEventListener("resize", () => {
    xBarChartInstance?.resize();
    sChartInstance?.resize();
    xBarChart2Instance?.resize();
    rChartInstance?.resize();
    iChartInstance?.resize();
    mrChartInstance?.resize();
  });

  // 生成初始数据
  generateNewData();
};

onMounted(() => {
  initCharts();
});

// 组件卸载时清理定时器
onBeforeUnmount(() => {
  if (simulationTimer) {
    clearInterval(simulationTimer);
    simulationTimer = null;
  }
});
</script>

<style scoped>
.spc-charts-container {
  min-height: 100vh;
  padding: 20px;
  background-color: #f5f5f5;
}

.controls-panel {
  margin-bottom: 20px;
}

.controls-panel .flex {
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}

.controls-panel .flex > span {
  white-space: nowrap;
}

.chart-section {
  margin-bottom: 30px;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.chart-title {
  font-size: 18px;
  font-weight: 600;
  color: #1890ff;
}

.chart-container {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.sub-chart {
  width: 100%;
}

.sub-chart h4 {
  margin: 0 0 10px;
  font-size: 14px;
  font-weight: 500;
  color: #666;
  text-align: center;
}

.chart {
  width: 100%;
  height: 300px;
  background-color: white;
  border: 1px solid #e8e8e8;
  border-radius: 6px;
}

@media (width >= 768px) {
  .chart-container {
    flex-direction: row;
  }

  .sub-chart {
    flex: 1;
  }

  .chart {
    height: 350px;
  }
}

:deep(.el-card__header) {
  background: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%);
  border-bottom: 2px solid #1890ff;
}

:deep(.el-card__body) {
  padding: 24px;
}

:deep(.el-input-number) {
  width: 120px;
}
</style>
