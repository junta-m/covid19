<template>
  <data-view
    :title="title"
    :title-id="titleId"
    :date="date"
    :head-title="title + infoTitles.join(',')"
  >
    <ul
      :class="$style.GraphLegend"
      :style="{ display: canvas ? 'block' : 'none' }"
    >
      <li v-for="(item, i) in dataLabels" :key="i" @click="onClickLegend(i)">
        <button>
          <div
            v-if="i === 1"
            :style="{
              backgroundColor: colors[i].fillColor,
              border: 0,
              height: '3px',
            }"
          />
          <div
            v-else
            :style="{
              backgroundColor: colors[i].fillColor,
              borderColor: colors[i].strokeColor,
            }"
          />
          <span
            :style="{
              textDecoration: displayLegends[i] ? 'none' : 'line-through',
            }"
            >{{ item }}</span
          >
        </button>
      </li>
    </ul>
    <h4 :id="`${titleId}-graph`" class="visually-hidden">
      {{ $t(`{title}のグラフ`, { title }) }}
    </h4>
    <scrollable-chart v-show="canvas" :display-data="displayData">
      <template v-slot:chart="{ chartWidth }">
        <bar
          :ref="'barChart'"
          :chart-id="chartId"
          :chart-data="displayData"
          :options="displayOption"
          :display-legends="displayLegends"
          :height="240"
          :width="chartWidth"
        />
      </template>
      <template v-slot:sticky-chart>
        <bar
          class="sticky-legend"
          :chart-id="`${chartId}-header-right`"
          :chart-data="displayDataHeader"
          :options="displayOptionHeader"
          :plugins="yAxesBgPlugin"
          :display-legends="displayLegends"
          :height="240"
        />
      </template>
    </scrollable-chart>
    <template v-slot:dataTable>
      <client-only>
        <data-view-table :headers="tableHeaders" :items="tableData" />
      </client-only>
    </template>
    <template v-slot:additionalDescription>
      <slot name="additionalDescription" />
    </template>
    <template v-slot:dataSetPanel>
      <data-view-data-set-panel
        :title="infoTitles[0]"
        :l-text="displayInfo[0].lText"
        :s-text="displayInfo[0].sText"
        :unit="displayInfo[0].unit"
      />
    </template>
    <template v-slot:footer>
      <open-data-link v-show="url" :url="url" />
    </template>
  </data-view>
</template>

<script lang="ts">
import Vue from 'vue'
import { ThisTypedComponentOptionsWithRecordProps } from 'vue/types/options'
import { TranslateResult } from 'vue-i18n'
import { Chart } from 'chart.js'
import dayjs from 'dayjs'
import DataView from '@/components/DataView.vue'
import DataViewTable, {
  TableHeader,
  TableItem,
} from '@/components/DataViewTable.vue'
import DataViewDataSetPanel from '@/components/DataViewDataSetPanel.vue'
import ScrollableChart from '@/components/ScrollableChart.vue'
import OpenDataLink from '@/components/OpenDataLink.vue'
import { DisplayData, yAxesBgPlugin } from '@/plugins/vue-chart'
import { getGraphSeriesColor, SurfaceStyle } from '@/utils/colors'
import { calcDayBeforeRatio } from '@/utils/formatDayBeforeRatio'
import { getNumberToFixedFunction } from '@/utils/monitoringStatusValueFormatters'

type Data = {
  canvas: boolean
  displayLegends: boolean[]
  colors: SurfaceStyle[]
}
type Methods = {
  makeLineData: (value: number) => number[]
  onClickLegend: (i: number) => void
}

type Computed = {
  displayInfo: [
    {
      lText: string
      sText: string
      unit: string
    }
  ]
  displayData: DisplayData
  displayOption: Chart.ChartOptions
  displayDataHeader: DisplayData
  displayOptionHeader: Chart.ChartOptions
  scaledTicksYAxisMax: number
  tableHeaders: TableHeader[]
  tableData: TableItem[]
}

type Props = {
  title: string
  titleId: string
  infoTitles: string[]
  chartId: string
  chartData: number[][]
  getFormatter: Function
  date: string
  labels: string[]
  dataLabels: string[] | TranslateResult[]
  unit: string
  url: string
}

const options: ThisTypedComponentOptionsWithRecordProps<
  Vue,
  Data,
  Methods,
  Computed,
  Props
> = {
  created() {
    this.canvas = process.browser
  },
  components: {
    DataView,
    DataViewTable,
    DataViewDataSetPanel,
    ScrollableChart,
    OpenDataLink,
  },
  props: {
    title: {
      type: String,
      default: '',
    },
    titleId: {
      type: String,
      required: false,
      default: '',
    },
    infoTitles: {
      type: Array,
      required: false,
      default: () => [],
    },
    chartId: {
      type: String,
      default: 'mixed-bar-and-line-chart',
    },
    chartData: {
      type: Array,
      required: false,
      default: () => [],
    },
    getFormatter: {
      type: Function,
      required: false,
      default: (_: number) => getNumberToFixedFunction(),
    },
    date: {
      type: String,
      required: true,
      default: '',
    },
    url: {
      type: String,
      default: '',
    },
    labels: {
      type: Array,
      default: () => [],
    },
    dataLabels: {
      type: Array,
      default: () => [],
    },
    unit: {
      type: String,
      default: '',
    },
  },
  data() {
    const colors: SurfaceStyle[] = [
      getGraphSeriesColor('B'),
      getGraphSeriesColor('A'),
    ]
    return {
      displayLegends: [true, true],
      colors,
      canvas: true,
      yAxesBgPlugin,
    }
  },
  computed: {
    displayInfo() {
      const { lastDay, lastDayData, dayBeforeRatio } = calcDayBeforeRatio({
        displayData: this.displayData,
        dataIndex: 1,
        digit: 1,
      })
      return [
        {
          lText: lastDayData,
          sText: `${this.$t('{date} の数値', {
            date: lastDay,
          })}（${this.$t('前日比')}: ${dayBeforeRatio} ${this.unit}）`,
          unit: this.unit,
        },
      ]
    },
    displayData() {
      return {
        labels: this.labels,
        datasets: [
          {
            type: 'bar',
            label: this.dataLabels[0],
            data: this.chartData[0],
            backgroundColor: this.colors[0].fillColor,
            borderColor: this.colors[0].strokeColor,
            borderWidth: 1,
            order: 3,
          },
          {
            type: 'line',
            label: this.dataLabels[1],
            data: this.chartData[1],
            pointBackgroundColor: 'rgba(0,0,0,0)',
            pointBorderColor: 'rgba(0,0,0,0)',
            borderColor: this.colors[1].fillColor,
            borderWidth: 3,
            fill: false,
            order: 2,
            lineTension: 0,
          },
        ],
      }
    },
    tableHeaders() {
      return [
        { text: this.$t('日付'), value: 'text' },
        ...(this.dataLabels as string[]).map((text, i) => {
          return { text, value: String(i), align: 'end' }
        }),
      ]
    },
    tableData() {
      return this.labels
        .map((label, i) => {
          return Object.assign(
            { text: label },
            ...(this.dataLabels as string[]).map((_, j) => {
              if (this.chartData[j][i] === null) {
                return {
                  [j]: '',
                }
              }
              return {
                [j]: this.getFormatter(j)(
                  parseFloat(this.chartData[j][i]?.toLocaleString())
                ),
              }
            })
          )
        })
        .sort((a, b) => dayjs(a.text).unix() - dayjs(b.text).unix())
        .reverse()
    },
    displayOption() {
      const unit = this.unit
      const options: Chart.ChartOptions = {
        tooltips: {
          displayColors: false,
          callbacks: {
            label: (tooltipItem) => {
              const cases = this.getFormatter(tooltipItem.datasetIndex!)(
                parseFloat(tooltipItem.value!.toLocaleString())
              )
              return `${
                this.dataLabels[tooltipItem.datasetIndex!]
              } : ${cases} ${unit}`
            },
            title(tooltipItem, data) {
              const date = data.labels![tooltipItem[0].index!].toString()
              return dayjs(date).format('M/D')
            },
          },
        },
        maintainAspectRatio: false,
        legend: {
          display: false,
        },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false,
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: '#808080',
                maxRotation: 0,
                callback: (label: string) => {
                  return dayjs(label).format('D')
                },
              },
              // #2384: If you set "type" to "time", make sure that the bars at both ends are not hidden.
              // #2384: typeをtimeに設定する時はグラフの両端が見切れないか確認してください
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: true,
                drawBorder: false,
                tickMarkLength: 10,
              },
              ticks: {
                fontSize: 11,
                fontColor: '#808080',
                padding: 3,
                fontStyle: 'bold',
              },
              type: 'time',
              time: {
                unit: 'month',
                displayFormats: {
                  month: 'MMM',
                },
              },
            },
          ],
          yAxes: [
            {
              position: 'left',
              gridLines: {
                display: true,
                drawOnChartArea: true,
                color: '#E5E5E5', // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax,
              },
            },
          ],
        },
      }
      if (this.$route.query.ogp === 'true') {
        Object.assign(options, { animation: { duration: 0 } })
      }
      return options
    },
    displayDataHeader() {
      return {
        labels: ['2020-01-01'],
        datasets: (this.dataLabels as string[]).map((_) => ({
          data: [],
          backgroundColor: 'transparent',
          borderWidth: 0,
        })),
      }
    },
    displayOptionHeader() {
      const options: Chart.ChartOptions = {
        maintainAspectRatio: false,
        legend: {
          display: false,
        },
        tooltips: { enabled: false },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false,
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: 'transparent',
                maxRotation: 0,
                minRotation: 0,
                callback: (label: string) => {
                  return dayjs(label).format('D')
                },
              },
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: false, // true -> false
                drawBorder: false,
                tickMarkLength: 10,
              },
              ticks: {
                fontSize: 11,
                fontColor: 'transparent', // #808080
                padding: 13, // 3 + 10(tickMarkLength)
                fontStyle: 'bold',
              },
              type: 'time',
              time: {
                unit: 'month',
              },
            },
          ],
          yAxes: [
            {
              type: 'linear',
              position: 'left',
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5', // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax,
              },
            },
          ],
        },
        animation: { duration: 0 },
      }
      return options
    },
    scaledTicksYAxisMax() {
      return this.chartData.reduce((max, data) => Math.max(max, ...data), 0)
    },
  },
  methods: {
    onClickLegend(i) {
      this.displayLegends[i] = !this.displayLegends[i]
      this.displayLegends = this.displayLegends.slice()
    },
    makeLineData(value: number): number[] {
      return this.chartData[0].map((_) => value)
    },
  },
  mounted() {
    const barChart = this.$refs.barChart as Vue
    const barElement = barChart.$el
    const canvas = barElement.querySelector('canvas')
    const labelledbyId = `${this.titleId}-graph`

    if (canvas) {
      canvas.setAttribute('role', 'img')
      canvas.setAttribute('aria-labelledby', labelledbyId)
    }
  },
}

export default Vue.extend(options)
</script>

<style module lang="scss">
.Graph {
  &Legend {
    text-align: center;
    list-style: none;
    padding: 0 !important;
    li {
      display: inline-block;
      margin: 0 3px;
      div {
        height: 12px;
        margin: 2px 4px;
        width: 40px;
        display: inline-block;
        vertical-align: middle;
        border-width: 1px;
        border-style: solid;
      }
      button {
        color: $gray-3;
        @include font-size(12);
      }
    }
  }
}
</style>
