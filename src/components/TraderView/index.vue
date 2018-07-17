<template>
  <div id="tv_chart_container">
  </div>
</template>

<script>
import {
  widget as TvWidget // 嫌路径长可以移入src里面 
} from '../../../static/charting_library-master/charting_library/charting_library.min'
import datafeeds from './datafeeds'
import ws from '@/utils/webSocket'

export default {
  props: {
    // webSocket 请求连接
    url: {
      type: String,
      default: 'wss://api.fcoin.com/v2/ws'
    },
    // tv配置信息
    options: {
      type: Object,
      default: () => {
        return {}
      }
    }
  },
  data() {
    return {
      widget: null,
      config: null,
      socket: null,
      cacheData: {},
      currentTicker: null,
      datafeed: new datafeeds(this)
    }
  },
  created() {
    if (!this.socket || !this.$store.state.socketState) {
      ws.init(this.url)
      this.socket = ws.socket
    }

    this.socket.onopen = event => {
      console.log(' >> webSocket 连接成功')
      this.$store.commit('changeSocketState', 1)
    }

    this.socket.onclose = event => {
      console.log(' >> webSocket 连接关闭')
      this.$store.commit('changeSocketState', 0)
    }

    this.socket.onerror = event => {
      console.log(' >> webSocket 连接异常：', event)
    }

    this.socket.onmessage = event => {
      // console.log(' >> webSocket 返回数据：', JSON.parse(event.data))
      this.onUpdateData(JSON.parse(event.data))
    }

  },
  methods: {
    // 初始化
    init() {
      const defaultOptions = {
        symbol: 'BTCUSDT',
        interval: 5,
        container_id: 'tv_chart_container',
        datafeed: this.datafeed,
        library_path: '/static/charting_library-master/charting_library/',
        drawings_access: {
          type: 'black',
          tools: [{ name: 'Regression Trend' }]
        },
        disabled_features: ['header_symbol_search'],
        enabled_features: [],
        numeric_formatting: {
          decimal_sign: '.'
        },
        timezone: 'Asia/Shanghai',
        locale: 'zh',
        debug: false
      }
      this.config = Object.assign(defaultOptions, this.options)
      this.widget = new TvWidget(this.config)
    },
    // 返回数据给图表
    getBars(symbolInfo, resolution, rangeStartDate, rangeEndDate, callback) {
      const currentTicker = symbolInfo.ticker.toUpperCase() + '_' + resolution
      // 取消订阅
      if (currentTicker !== this.currentTicker && this.currentTicker) {
        delete this.cacheData[this.currentTicker]
        this.datafeed.unsubscribeBars(currentTicker)
        const period = this.currentTicker.split('_')
        this.sendData({ cmd: 'unsub', args: [`candle.M${period[1]}.${period[0].toLowerCase()}`] })
      }
      this.currentTicker = currentTicker
      if (this.cacheData[this.currentTicker]) {
        callback(this.cacheData[this.currentTicker])
        // 订阅当前周期K线数据
        const params = {
          cmd: 'sub',
          args: [`candle.M${resolution}.${symbolInfo.ticker.toLowerCase()}`]
        }
        this.sendData(params)
      } else {
        // 获取历史数据
        const params = {
          cmd: 'req',
          args: [`candle.M${resolution}.${symbolInfo.ticker.toLowerCase()}`, 1441, rangeEndDate]
        }
        this.sendData(params, () => {
          this.getBars(symbolInfo, resolution, rangeStartDate, rangeEndDate, callback)
        })
      }
    },
    // 发送数据给服务端
    sendData(params, callback) {
      if (this.$store.state.socketState) {
        this.socket.send(JSON.stringify(params))
      } else {
        this.socket.onopen = event => {
          this.$store.commit('changeSocketState', 1)
          this.socket.send(JSON.stringify(params))
        }
      }
      callback && callback()
    },
    // 更新数据
    onUpdateData(data) {
      if (data.data) {
        data.data.map(val => val.time = val.id * 1000)
        this.cacheData[this.currentTicker] = data.data
      }
      // if (this.cacheData[this.currentTicker]) {
      //   const bars = this.cacheData[this.currentTicker]
      //   // 最后一条线
      //   const lastTime = bars[bars.length - 1].id
      //   if (data.id === (lastTime + this.period * 60)) {
      //     data.time = data.id * 1000
      //     this.cacheData[this.currentTicker].push(data)
      //   }
      // }
    }
  }
}
</script>
