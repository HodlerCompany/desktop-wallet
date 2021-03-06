<template>
  <section class="MarketChart w-full">
    <slot />
    <LineChart
      v-show="isReady"
      ref="chart"
      :chart-data="chartData"
      :options="options"
      :height="315"
      @ready="show"
    />
  </section>
</template>

<script>
import dayjs from 'dayjs'
import LineChart from '@/components/utils/LineChart'
import cryptoCompare from '@/services/crypto-compare'

export default {
  name: 'MarketChart',

  provide () {
    return {
      changePeriod: this.changePeriod,
      getPeriod: this.getPeriod
    }
  },

  components: {
    LineChart
  },

  props: {
    isActive: {
      type: Boolean,
      required: false,
      default: true
    }
  },

  data: () => ({
    isReady: false,
    period: 'day',
    chartData: {},
    options: {},
    gradient: null
  }),

  computed: {
    colours () {
      return {
        gradient: ['#666', '#528fe3', '#9c6dd8', '#e15362'],
        dark: {
          lines: '#787fa3',
          ticks: '#787fa3'
        },
        light: {
          lines: '#9ea7bc',
          ticks: '#9ea7bc'
        }
      }
    },

    currency () {
      return this.$store.getters['session/currency']
    },

    theme () {
      return this.$store.getters['session/theme']
    },

    token () {
      return this.session_network.token
    }
  },

  watch: {
    currency () {
      this.renderChart()
    },

    theme () {
      this.renderChart()
    },

    token () {
      this.renderChart()
    },

    period () {
      this.renderChart()
    },

    isActive (val) {
      if (!val) return // Render the chart when open the component

      this.renderChart()
    }
  },

  mounted () {
    // Avoid creating the gradient when the element is not built
    if (this.isActive) {
      this.renderChart()
    }
  },

  methods: {
    show () {
      this.isReady = true
    },
    async renderChart () {
      // TODO: Add loading
      await this.renderGradient()

      const response = await cryptoCompare.historicByType(this.period, this.token, this.currency)

      // Since BTC price could be very low :(, the linear scale could produce
      // imprecise values, such as converting 0.00001078 to 0, provoking that
      // the chart isn't displayed correctly
      const scaleCorrection = 1000
      const data = response.datasets.map(datum => datum * scaleCorrection)

      const themeGridLines = this.colours[this.theme].lines
      const themeTicks = this.colours[this.theme].ticks

      const fontConfig = {
        fontColor: themeTicks,
        fontSize: 14,
        fontStyle: 600
      }

      this.options = {
        showScale: true,
        responsive: true,
        maintainAspectRatio: false,
        elements: {
          line: {
            cubicInterpolationMode: 'monotone'
            // NOTE: to improve rendering time
            // tension: 0
          }
        },
        legend: {
          display: false
        },
        layout: {
          padding: {
            left: 0,
            right: 0,
            top: 10,
            bottom: 10
          }
        },
        scales: {
          yAxes: [
            {
              gridLines: {
                borderDash: [5, 5],
                color: themeGridLines,
                display: true,
                drawBorder: false
              },
              ticks: {
                padding: 15,
                ...fontConfig,
                callback: (value, index) => {
                  if (index % 2 === 0) return

                  return this.currency_format(value / scaleCorrection, { currency: this.currency })
                }
              }
            }
          ],
          xAxes: [
            {
              gridLines: {
                drawBorder: false,
                display: false
              },
              ticks: {
                padding: 10,
                ...fontConfig,
                callback: (value, index, values) => {
                  if (this.period !== 'day' && index === values.length - 1) {
                    return this.$t('MARKET_CHART.TODAY')
                  } else if (this.period === 'week') {
                    const width = this.$el.clientWidth
                    if (width > 1200) {
                      return this.$t(`MARKET_CHART.WEEK.LONG.${value.toUpperCase()}`)
                    } else {
                      return this.$t(`MARKET_CHART.WEEK.SHORT.${value.toUpperCase()}`)
                    }
                  }

                  return value
                }
              }
            }
          ]
        },
        tooltips: {
          displayColors: false,
          callbacks: {
            label: (item, data) => {
              return this.currency_format(item.yLabel / scaleCorrection, { currency: this.currency })
            },
            title: (items, data) => {
              const { index } = items[0]
              const values = data.datasets[0].data
              let title = items[0].xLabel

              if (this.period === 'day') {
                const midnight = data.labels.indexOf('00:00')
                if (index < midnight) {
                  return this.$t('MARKET_CHART.YESTERDAY_AT', { hour: title })
                }
                return this.$t('MARKET_CHART.TODAY_AT', { hour: title })
              } else if (index === values.length - 1) {
                return this.$t('MARKET_CHART.TODAY')
              } else if (this.period === 'week') {
                return this.$t(`MARKET_CHART.WEEK.LONG.${title.toUpperCase()}`)
              } else {
                const days = values.length
                const today = dayjs()
                today.date(values[days - 1])
                title = today.subtract(days - index - 1, 'day')
                return this.$d(title)
              }
            }
          }
        }
      }

      this.chartData = {
        labels: response.labels,
        datasets: [{
          // Do not show the points, but enable a big target for the tooltip
          pointHitRadius: 12,
          pointRadius: 0,
          borderWidth: 3,
          type: 'line',
          fill: false,
          borderColor: this.gradient || this.colours.gradient[0],
          data
        }]
      }
    },

    async renderGradient () {
      if (this.gradient) return

      await this.$nextTick

      const canvas = this.$refs.chart.getCanvas()
      const width = this.$el.clientWidth

      if (width === 0) return

      this.gradient = canvas.getContext('2d').createLinearGradient(0, 0, width, 0)
      this.gradient.addColorStop(0, this.colours.gradient[1])
      this.gradient.addColorStop(0.5, this.colours.gradient[2])
      this.gradient.addColorStop(1, this.colours.gradient[3])
    },

    changePeriod (period) {
      this.period = period

      this.renderChart()
    },

    getPeriod () {
      return this.period
    }
  }
}
</script>

<style scoped>
.MarketChart {
  min-height: 315px
}
</style>
