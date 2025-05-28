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

    <hr />

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">不合格品数量控制图 (np Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>np 图</h4>
            <div ref="npChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">不合格品率控制图 (p Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>p 图</h4>
            <div ref="pChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">不合格品率趋势图 (pt Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>pt 图</h4>
            <div ref="ptChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">不合格数控制图 (c Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>c 图</h4>
            <div ref="cChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">单位不合格数控制图 (u Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>u 图</h4>
            <div ref="uChart" class="chart" />
          </div>
        </div>
      </el-card>
    </div>

    <div class="chart-section">
      <el-card shadow="hover" class="mb-6">
        <template #header>
          <div class="card-header">
            <span class="chart-title">单位不合格数趋势图 (ut Chart)</span>
          </div>
        </template>
        <div class="chart-container">
          <div class="sub-chart">
            <h4>ut 图</h4>
            <div ref="utChart" class="chart" />
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

// Chart DOM refs (existing)
const xBarChart = ref<HTMLElement>();
const sChart = ref<HTMLElement>();
const xBarChart2 = ref<HTMLElement>();
const rChart = ref<HTMLElement>();
const iChart = ref<HTMLElement>();
const mrChart = ref<HTMLElement>();

// New Chart DOM refs
const npChart = ref<HTMLElement>();
const pChart = ref<HTMLElement>();
const ptChart = ref<HTMLElement>();
const cChart = ref<HTMLElement>();
const uChart = ref<HTMLElement>();
const utChart = ref<HTMLElement>();

// Chart instances (existing)
let xBarChartInstance: ECharts | null = null;
let sChartInstance: ECharts | null = null;
let xBarChart2Instance: ECharts | null = null;
let rChartInstance: ECharts | null = null;
let iChartInstance: ECharts | null = null;
let mrChartInstance: ECharts | null = null;

// New Chart instances
let npChartInstance: ECharts | null = null;
let pChartInstance: ECharts | null = null;
let ptChartInstance: ECharts | null = null;
let cChartInstance: ECharts | null = null;
let uChartInstance: ECharts | null = null;
let utChartInstance: ECharts | null = null;

// Data storage (existing)
const rawData = ref<number[][]>([]); // For Xbar-S, Xbar-R
const individualData = ref<number[]>([]); // For I-MR

// New Data storage for attribute charts
const sampleSize_np_p = 100; // Sample size for np and p charts
const sampleSize_u = 10; // Sample size for u chart (e.g., number of inspection units)

const defectiveData = ref<number[]>([]); // Number of defectives (for np, p)
const defectsData = ref<number[]>([]); // Number of defects (for c, u)

// --- Data Generation Functions ---

// Generate random data (existing)
const generateSampleData = (count: number, sampleSize: number = 5) => {
  const data: number[][] = [];
  for (let i = 0; i < count; i++) {
    const sample: number[] = [];
    for (let j = 0; j < sampleSize; j++) {
      // Generate normal distribution data (mean=100, stdDev=5)
      sample.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
    }
    data.push(sample);
  }
  return data;
};

// Generate individual data (existing)
const generateIndividualData = (count: number) => {
  const data: number[] = [];
  for (let i = 0; i < count; i++) {
    data.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
  }
  return data;
};

// Generate Defective Data (for np, p charts)
const generateDefectiveData = (count: number, sampleSize: number) => {
  const data: number[] = [];
  for (let i = 0; i < count; i++) {
    // Simulate number of defectives, e.g., 2% defective rate
    const defectives = Math.floor(
      Math.random() * sampleSize * 0.04 + sampleSize * 0.01
    ); // Between 1% and 5% defectives
    data.push(Math.min(defectives, sampleSize)); // Ensure it doesn't exceed sample size
  }
  return data;
};

// Generate Defects Data (for c, u charts)
const generateDefectsData = (count: number) => {
  const data: number[] = [];
  for (let i = 0; i < count; i++) {
    // Simulate number of defects, e.g., Poisson distribution approximation
    const defects = Math.floor(Math.random() * 6 + 1); // Between 1 and 6 defects
    data.push(defects);
  }
  return data;
};

// --- Statistical Calculation Functions (existing) ---

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

// --- New Statistical Calculation Functions for Attribute Charts ---

// For np chart
const calculateNpChartLimits = (
  defectiveCounts: number[],
  sampleSize: number
) => {
  const totalDefectives = defectiveCounts.reduce((sum, val) => sum + val, 0);
  const totalSamples = defectiveCounts.length;
  const pBar = totalDefectives / (totalSamples * sampleSize); // Average proportion defective

  const CL = pBar * sampleSize;
  const UCL = CL + 3 * Math.sqrt(CL * (1 - pBar));
  const LCL = Math.max(0, CL - 3 * Math.sqrt(CL * (1 - pBar)));
  return { CL, UCL, LCL };
};

// For p chart
const calculatePChartLimits = (
  defectiveCounts: number[],
  sampleSize: number
) => {
  const totalDefectives = defectiveCounts.reduce((sum, val) => sum + val, 0);
  const totalSamples = defectiveCounts.length;
  const pBar = totalDefectives / (totalSamples * sampleSize); // Average proportion defective

  const CL = pBar;
  const UCL = CL + 3 * Math.sqrt((CL * (1 - CL)) / sampleSize);
  const LCL = Math.max(0, CL - 3 * Math.sqrt((CL * (1 - CL)) / sampleSize));
  return { CL, UCL, LCL };
};

// For c chart
const calculateCChartLimits = (defects: number[]) => {
  const cBar = defects.reduce((sum, val) => sum + val, 0) / defects.length;
  const CL = cBar;
  const UCL = CL + 3 * Math.sqrt(CL);
  const LCL = Math.max(0, CL - 3 * Math.sqrt(CL));
  return { CL, UCL, LCL };
};

// For u chart
const calculateUChartLimits = (defects: number[], sampleSize: number) => {
  const totalDefects = defects.reduce((sum, val) => sum + val, 0);
  const totalSamples = defects.length;
  const uBar = totalDefects / (totalSamples * sampleSize); // Average defects per unit

  const CL = uBar;
  const UCL = CL + 3 * Math.sqrt(CL / sampleSize);
  const LCL = Math.max(0, CL - 3 * Math.sqrt(CL / sampleSize));
  return { CL, UCL, LCL };
};

// Create control chart common option (existing)
const createControlChartOption = (
  data: number[],
  centerLine: number,
  ucl: number,
  lcl: number,
  title: string,
  yAxisName: string,
  minY?: number, // Optional min Y-axis value
  maxY?: number // Optional max Y-axis value
) => {
  const xAxisData = data.map((_, index) => index + 1);

  const option = {
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
      min: minY, // Apply min Y-axis if provided
      max: maxY, // Apply max Y-axis if provided
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
  return option;
};

// Update chart data (existing and new)
const updateCharts = () => {
  // Keep recent data only
  const currentRawData = rawData.value.slice(-dataPointsCount.value);
  const currentIndividualData = individualData.value.slice(
    -dataPointsCount.value
  );
  const currentDefectiveData = defectiveData.value.slice(
    -dataPointsCount.value
  );
  const currentDefectsData = defectsData.value.slice(-dataPointsCount.value);

  // Calculate statistics (existing)
  const xBarData = calculateXBar(currentRawData);
  const sData = calculateS(currentRawData);
  const rData = calculateR(currentRawData);
  const mrData = calculateMR(currentIndividualData);

  // Calculate control limits (simplified calculation, more precise statistical constants needed for actual application)
  const xBarMean =
    xBarData.reduce((sum, val) => sum + val, 0) / xBarData.length;
  const sMean = sData.reduce((sum, val) => sum + val, 0) / sData.length;
  const rMean = rData.reduce((sum, val) => sum + val, 0) / rData.length;
  const iMean =
    currentIndividualData.reduce((sum, val) => sum + val, 0) /
    currentIndividualData.length;
  const mrMean = mrData.reduce((sum, val) => sum + val, 0) / mrData.length;

  // Update X̄-S Chart
  if (xBarChartInstance) {
    xBarChartInstance.setOption(
      createControlChartOption(
        xBarData,
        xBarMean,
        xBarMean + (3 * sMean) / Math.sqrt(5), // Simplified control limit calculation
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
        sMean * 2.089, // Simplified B4 coefficient for n=5
        sMean * 0, // Simplified B3 coefficient for n=5 (B3 is 0 for n<=5)
        "",
        "标准偏差"
      )
    );
  }

  // Update X̄-R Chart
  if (xBarChart2Instance) {
    xBarChart2Instance.setOption(
      createControlChartOption(
        xBarData,
        xBarMean,
        xBarMean + 0.577 * rMean, // A2 coefficient for n=5
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
        rMean * 2.114, // D4 coefficient for n=5
        rMean * 0, // D3 coefficient for n=5 (D3 is 0 for n<=6)
        "",
        "极差"
      )
    );
  }

  // Update I-MR Chart
  if (iChartInstance) {
    iChartInstance.setOption(
      createControlChartOption(
        currentIndividualData,
        iMean,
        iMean + 2.66 * mrMean, // E2 coefficient for n=2 (for MR chart, n=2)
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
        mrMean * 3.267, // D4 coefficient for n=2
        mrMean * 0, // D3 coefficient for n=2
        "",
        "移动极差"
      )
    );
  }

  // --- Update New Attribute Charts ---

  // np chart
  if (npChartInstance) {
    const { CL, UCL, LCL } = calculateNpChartLimits(
      currentDefectiveData,
      sampleSize_np_p
    );
    npChartInstance.setOption(
      createControlChartOption(
        currentDefectiveData,
        CL,
        UCL,
        LCL,
        "",
        "不合格品数量",
        0 // np chart's LCL can be 0 or greater
      )
    );
  }

  // p chart
  if (pChartInstance) {
    const pData = currentDefectiveData.map(d => d / sampleSize_np_p);
    const { CL, UCL, LCL } = calculatePChartLimits(
      currentDefectiveData,
      sampleSize_np_p
    );
    pChartInstance.setOption(
      createControlChartOption(
        pData,
        CL,
        UCL,
        LCL,
        "",
        "不合格品率",
        0, // p chart's LCL can be 0 or greater
        1 // p chart's max value is 1
      )
    );
  }

  // pt chart (using p chart logic)
  if (ptChartInstance) {
    const pData = currentDefectiveData.map(d => d / sampleSize_np_p);
    const { CL, UCL, LCL } = calculatePChartLimits(
      currentDefectiveData,
      sampleSize_np_p
    );
    ptChartInstance.setOption(
      createControlChartOption(pData, CL, UCL, LCL, "", "不合格品率", 0, 1)
    );
  }

  // c chart
  if (cChartInstance) {
    const { CL, UCL, LCL } = calculateCChartLimits(currentDefectsData);
    cChartInstance.setOption(
      createControlChartOption(
        currentDefectsData,
        CL,
        UCL,
        LCL,
        "",
        "不合格数",
        0 // c chart's LCL can be 0 or greater
      )
    );
  }

  // u chart
  if (uChartInstance) {
    const uData = currentDefectsData.map(d => d / sampleSize_u);
    const { CL, UCL, LCL } = calculateUChartLimits(
      currentDefectsData,
      sampleSize_u
    );
    uChartInstance.setOption(
      createControlChartOption(
        uData,
        CL,
        UCL,
        LCL,
        "",
        "单位不合格数",
        0 // u chart's LCL can be 0 or greater
      )
    );
  }

  // ut chart (using u chart logic)
  if (utChartInstance) {
    const uData = currentDefectsData.map(d => d / sampleSize_u);
    const { CL, UCL, LCL } = calculateUChartLimits(
      currentDefectsData,
      sampleSize_u
    );
    utChartInstance.setOption(
      createControlChartOption(uData, CL, UCL, LCL, "", "单位不合格数", 0)
    );
  }
};

// Generate new data (existing and new)
const generateNewData = () => {
  // Existing data generation
  rawData.value = generateSampleData(dataPointsCount.value * 2);
  individualData.value = generateIndividualData(dataPointsCount.value * 2);

  // New data generation for attribute charts
  defectiveData.value = generateDefectiveData(
    dataPointsCount.value * 2,
    sampleSize_np_p
  );
  defectsData.value = generateDefectsData(dataPointsCount.value * 2);

  updateCharts();
};

// Add a single new data point (existing and new)
const addNewDataPoint = () => {
  // Add new sample for raw data
  const newSample: number[] = [];
  for (let j = 0; j < 5; j++) {
    newSample.push(100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5);
  }
  rawData.value.push(newSample);

  // Add new individual value
  const newIndividualValue =
    100 + (Math.random() - 0.5) * 20 + Math.random() * 10 - 5;
  individualData.value.push(newIndividualValue);

  // Add new defective data
  const newDefectives = Math.floor(
    Math.random() * sampleSize_np_p * 0.04 + sampleSize_np_p * 0.01
  );
  defectiveData.value.push(Math.min(newDefectives, sampleSize_np_p));

  // Add new defects data
  const newDefects = Math.floor(Math.random() * 6 + 1);
  defectsData.value.push(newDefects);

  // Keep data within a reasonable range, discard older data
  if (rawData.value.length > dataPointsCount.value * 3) {
    rawData.value = rawData.value.slice(-dataPointsCount.value * 2);
  }
  if (individualData.value.length > dataPointsCount.value * 3) {
    individualData.value = individualData.value.slice(
      -dataPointsCount.value * 2
    );
  }
  if (defectiveData.value.length > dataPointsCount.value * 3) {
    defectiveData.value = defectiveData.value.slice(-dataPointsCount.value * 2);
  }
  if (defectsData.value.length > dataPointsCount.value * 3) {
    defectsData.value = defectsData.value.slice(-dataPointsCount.value * 2);
  }

  updateCharts();
};

// Start/Stop Simulation (existing)
const toggleSimulation = () => {
  if (isSimulating.value) {
    // Stop simulation
    if (simulationTimer) {
      clearInterval(simulationTimer);
      simulationTimer = null;
    }
    isSimulating.value = false;
  } else {
    // Start simulation
    isSimulating.value = true;
    simulationTimer = setInterval(() => {
      addNewDataPoint();
    }, simulationSpeed.value);
  }
};

// Initialize charts (existing and new)
const initCharts = async () => {
  await nextTick();

  // Existing chart initializations
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

  // New chart initializations
  if (npChart.value) {
    npChartInstance = echarts.init(npChart.value);
  }
  if (pChart.value) {
    pChartInstance = echarts.init(pChart.value);
  }
  if (ptChart.value) {
    ptChartInstance = echarts.init(ptChart.value);
  }
  if (cChart.value) {
    cChartInstance = echarts.init(cChart.value);
  }
  if (uChart.value) {
    uChartInstance = echarts.init(uChart.value);
  }
  if (utChart.value) {
    utChartInstance = echarts.init(utChart.value);
  }

  // Resize charts on window resize
  window.addEventListener("resize", () => {
    xBarChartInstance?.resize();
    sChartInstance?.resize();
    xBarChart2Instance?.resize();
    rChartInstance?.resize();
    iChartInstance?.resize();
    mrChartInstance?.resize();
    npChartInstance?.resize();
    pChartInstance?.resize();
    ptChartInstance?.resize();
    cChartInstance?.resize();
    uChartInstance?.resize();
    utChartInstance?.resize();
  });

  // Generate initial data
  generateNewData();
};

onMounted(() => {
  initCharts();
});

// Clear timer on component unmount
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
