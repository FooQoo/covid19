<template>
  <data-view :title="title" :title-id="titleId" :date="date" :url="url">
    <template v-slot:button>
      <p class="Graph-Desc">
        {{ $t('(注)') }} {{ $t('福井県から情報が提供されていないため、あくまでも参考値となります') }}<br />
        {{ $t('(注)') }} {{ $t('現在患者数 = 累計陽性患者数 - 退院済患者数 - 死亡者数') }}<br />
        {{ $t('(注)') }} {{ $t('段階は４段階に分かれています') }}<br />
        　　　{{ $t('「青：良好」／「黄：注意報」／「赤：警報」／「黒：緊急事態」') }}
      </p>
    </template>
    <pie-chart
      :style="{ display: canvas ? 'block' : 'none' }"
      :chart-id="chartId"
      :chart-data="displayData"
      :options="displayOption"
      :height="240"
    />
    <template v-slot:infoPanel>
      <data-view-basic-info-panel
        :l-text="displayInfo.lText"
        :s-text="displayInfo.sText"
        :unit="displayInfo.unit"
      />
    </template>
  </data-view>
</template>

<script lang="ts">
import Vue from 'vue'
import { ThisTypedComponentOptionsWithRecordProps } from 'vue/types/options'
import { GraphDataType } from '@/utils/formatGraph'
import DataView from '@/components/DataView.vue'
import DataViewBasicInfoPanel from '@/components/DataViewBasicInfoPanel.vue'
// import { single as color } from '@/utils/colors'

type Data = {
  dataKind: 'transition' | 'cumulative'
  canvas: boolean
}
type Methods = {
  formatDayBeforeRatio: (dayBeforeRatio: number) => string
}
type Computed = {
  displayCumulativeRatio: string
  displayTransitionRatio: string
  displayInfo: {
    lText: string
    sText: string
    unit: string
  }
  displayData: {
    labels: string[]
    datasets: {
      label: string[]
      data: number[]
      backgroundColor: (string | string[])[]
      borderWidth: number
    }[]
  }
  displayOption: {
    tooltips: {
      displayColors: boolean
      callbacks: {
        label(tooltipItem: any): string
        title(tooltipItem: any[], data: any): string | undefined
      }
    }
    responsive: boolean
    maintainAspectRatio: boolean
    legend: {
      display: boolean
    }
  }
  scaledTicksYAxisMax: number
  tableHeaders: {
    text: string
    value: string
  }[]
  tableData: {
    [key: number]: number
  }[]
}
type Props = {
  title: string
  titleId: string
  chartId: string
  chartData: GraphDataType[]
  date: string
  unit: string
  info: string
  labels: string[]
  url: string
  description: string
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
  components: { DataView, DataViewBasicInfoPanel },
  props: {
    title: {
      type: String,
      default: ''
    },
    titleId: {
      type: String,
      default: ''
    },
    chartId: {
      type: String,
      default: 'time-bar-chart'
    },
    chartData: {
      type: Array,
      default: () => []
    },
    date: {
      type: String,
      required: true
    },
    unit: {
      type: String,
      default: ''
    },
    info: {
      type: String,
      default: ''
    },
    labels: {
      type: Array,
      default: () => []
    },
    url: {
      type: String,
      default: ''
    },
    description: {
      type: String,
      default: ''
    }
  },
  data: () => ({
    dataKind: 'transition',
    canvas: true
  }),
  computed: {
    displayCumulativeRatio() {
      const lastDay = this.chartData.slice(-1)[0].cumulative
      const lastDayBefore = this.chartData.slice(-2)[0].cumulative
      return this.formatDayBeforeRatio(lastDay - lastDayBefore)
    },
    displayTransitionRatio() {
      const lastDay = this.chartData.slice(-1)[0].transition
      const lastDayBefore = this.chartData.slice(-2)[0].transition
      return this.formatDayBeforeRatio(lastDay - lastDayBefore)
    },
    displayInfo() {
      const totalBeds = this.chartData[this.chartData.length - 1].cumulative
      const usedBeds = this.chartData[0].cumulative
      const rateOfUsed = ((usedBeds / totalBeds) * 100).toFixed(2)
      return {
        lText: rateOfUsed.toLocaleString(),
        sText: this.info,
        unit: this.unit
      }
    },
    displayData() {
      // const colorArray = ['#006400', '#ccc']
      const colorArray = [
        [
          '#003580', // normal
          '#FFD700', // warning
          '#FC143C', // critical
          "#000", // emergency
        ],
        '#ccc'
      ]
      return {
        labels: this.chartData.map((_d: any, index) => {
          return this.labels[index]
        }),
        datasets: [
          {
            label: this.chartData.map((_d: any, index) => {
              return this.labels[index]
            }),
            data: this.chartData.map((d: any) => {
              if (d.label === 'unused') {
                if (d.transition < 0) {
                  d.transition = 0
                }
              }
              return d.transition
            }),
            backgroundColor: this.chartData.map((d, index) => {
              if (d.label === 'used') {
                const rateOfUsed = parseFloat(this.displayInfo.lText)
                let status = 0
                if(rateOfUsed >= 10)
                  status = 1
                if(rateOfUsed >= 20)
                  status = 2
                if(rateOfUsed >= 25)
                  status = 3
                return colorArray[index][status]
              } else {
                return colorArray[index]
              }
            }),
            borderWidth: 0
          }
        ]
      }
    },
    displayOption() {
      // const unit = this.unit
      const chartData = this.chartData
      const options = {
        tooltips: {
          displayColors: false,
          callbacks: {
            label(tooltipItem: any) {
              return `${
                chartData[tooltipItem.index].transition
              } 人 (総病床数: ${chartData[1].cumulative} 人)`
            },
            title(tooltipItem: any, data: any) {
              return data.labels[tooltipItem[0].index]
            }
          }
        },
        responsive: true,
        maintainAspectRatio: false,
        legend: {
          display: true
        }
      }
      return options
    },
    scaledTicksYAxisMax() {
      const yAxisMax = 1.2
      const dataKind =
        this.dataKind === 'transition' ? 'transition' : 'cumulative'
      const values = this.chartData.map(d => d[dataKind])
      return Math.max(...values) * yAxisMax
    },
    tableHeaders() {
      return [
        { text: '', value: 'text' },
        { text: this.title, value: '0' }
      ]
    },
    tableData() {
      return this.displayData.datasets[0].data.map((_, i) => {
        return Object.assign(
          { text: this.displayData.labels[i] },
          { '0': this.displayData.datasets[0].data[i] }
        )
      })
    }
  },
  methods: {
    formatDayBeforeRatio(dayBeforeRatio: number): string {
      const dayBeforeRatioLocaleString = dayBeforeRatio.toLocaleString()
      switch (Math.sign(dayBeforeRatio)) {
        case 1:
          return `+${dayBeforeRatioLocaleString}`
        case -1:
          return `${dayBeforeRatioLocaleString}`
        default:
          return `${dayBeforeRatioLocaleString}`
      }
    }
  }
}

export default Vue.extend(options)
</script>

<style lang="scss" scoped>
.Graph-Desc {
  margin: 10px 0;
  font-size: 12px;
  color: $gray-3;
}

.link {
  text-decoration: none;
}
</style>
