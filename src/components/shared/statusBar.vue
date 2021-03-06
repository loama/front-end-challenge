<template>
  <div id="status-bar">
    <v-select v-model="books.initial" :options="books.available" @input="bookChange" :searchable="false"></v-select>

    <div id="ticker">
      <div class="item vol-24-hrs">
        Volumen 24 hrs.
        <span class="value">{{ticker.volume}} {{books.selected.unit}}</span>
      </div>

      <div class="item max">
        Max.
        <span class="value">{{ticker.high}} {{books.selected.comparision}}</span>
      </div>

      <div class="item min">
        Min.
        <span class="value">{{ticker.low}} {{books.selected.comparision}}</span>
      </div>

      <div class="item variation">
        Variación
        <span class="value">{{variation.sign}} {{variation.value}} {{ticker.book.split('_')[1]}} ({{variation.percentage}}%)</span>
      </div>
    </div>
    <div v-on:click="toggleFullscreen()" id="fullscreen-button">
      <span> pantalla completa </span>
      <img src="../../assets/images/fullscreen-yellow.svg" class="yellow">
      <img src="../../assets/images/fullscreen.svg" class="normal">
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import VueResource from 'vue-resource'
import VueHead from 'vue-head'
import vSelect from 'vue-select'

Vue.use(VueResource)
Vue.use(VueHead)

Vue.component('v-select', vSelect)

var websocket = new WebSocket('wss://ws.bitso.com')

export default {
  computed: {
    books: function () {
      return this.$store.state.books
    },
    ticker: function () {
      return this.$store.state.ticker
    }
  },
  data () {
    return {
      variation: {
        percentage: 0,
        sign: '',
        value: 0
      },
      websocket: {
        open: false
      },
      bookLoaded: false
    }
  },
  head: {
    title: function () {
      return {
        inner: this.ticker.last + ' (' + this.books.selected.label.toUpperCase() + ')'
      }
    }
  },
  methods: {
    actualBookChange (val) {
      var self = this
      console.log('0')
      // load data for that book
      console.log('bookChange')
      var data = self.$store.state.books.values[val.url]
      var newBook = {url: val.url, label: val.label, unit: val.unit, comparision: val.comparision, data: data}
      self.$store.dispatch('bookChange', newBook)
      self.variation.value = Math.abs(data[364].value - data[363].value).toFixed(2)
      if ((data[364].value - data[363].value) > 0) {
        self.variation.sign = '+'
      } else {
        self.variation.sign = '-'
      }
      self.variation.percentage = (data[364].value / data[363].value * 100 - 100).toFixed(1)

      // load statusbar
      self.loadTicker()

      // change ws connection
      if (this.websocket.open) {
        websocket.send(JSON.stringify({ action: 'subscribe', book: this.books.selected.url, type: 'trades' }))
        websocket.send(JSON.stringify({ action: 'unsubscribe', book: this.books.last.url, type: 'trades' }))
        websocket.send(JSON.stringify({ action: 'subscribe', book: this.books.selected.url, type: 'diff-orders' }))
        websocket.send(JSON.stringify({ action: 'unsubscribe', book: this.books.last.url, type: 'diff-orders' }))
      }
    },
    bookChange (val) { /* function after changing the books select */
      /* fix unwanted feature that deselected the option if you clicked the same that is currently selected */
      var self = this
      // console.log('bookChange')
      if (val !== null) {
        setTimeout(function () {
          if (self.$store.state.books.values[val.url].length === 0) {
            self.bookChange(val)
          } else {
            self.actualBookChange(val)
          }
        }, 300)
      } else {
        this.books.initial = this.books.selected
      }
      /* end feature */
    },
    toggleFullscreen () {
      this.$fullscreen.toggle(document.getElementById('app'), {
        wrap: false,
        callback: this.fullscreenOn
      })
    },
    fullscreenOn () {
      this.$store.dispatch('fullscreenOn')
    },
    loadTicker (book) {
      var self = this
      Vue.http.get('https://bitso-challenge.firebaseapp.com/ticker?book=' + self.books.selected.url).then(function (data) {
        self.$store.commit('ticker', data.body.payload)
        self.$emit('updateHead')
      }, function (err) {
        console.log(err)
      })
    }
  },
  mounted () {
    var self = this

    this.loadTicker('btc_mxn')

    /* websockets */
    websocket.onopen = function () {
      self.websocket.open = true
      websocket.send(JSON.stringify({ action: 'subscribe', book: 'btc_mxn', type: 'trades' }))
      // websocket.send(JSON.stringify({ action: 'subscribe', book: 'btc_mxn', type: 'diff-orders' }))
      // websocket.send(JSON.stringify({ action: 'subscribe', book: 'btc_mxn', type: 'orders' }))
    }

    websocket.onmessage = function (message) {
      var data = JSON.parse(message.data)
      switch (data.type) {
        case 'trades':
          if (data.book === self.books.selected.url) {
            self.$store.commit('tradesPush', data.payload[0])
            self.$emit('updateHead')
          }
          break
        /* case 'orders':
          self.$store.commit('wsOrders', data)
          break */
        default:
          // console.log(data)
          break
      }
    }
    /* end websockets */
  },
  name: 'statusBar'
}
</script>

<style scoped>
  #status-bar {
    position: fixed;
    top: 66px;
    left: 0;
    width: 100vw;
    height: 44px;
    background: #151A1E;
    z-index: 20;
  }

  #status-bar #ticker {
    position: absolute;
    top: 0;
    left: 160px;
    color: #767D83;
    font-size: 14px;
    height: 44px;
    line-height: 40px;
    min-width: 830px;
  }

  #status-bar #ticker .item {
    display: inline-block;
    margin-right: 32px;
  }

  #status-bar #ticker .item .value {
    color: #9D9FA0;
    text-transform: uppercase;
  }

  #fullscreen-button {
    position: absolute;
    top: 0;
    right: 0;
    height: 44px;
    text-align: right;
    padding-right: 44px;
    cursor: pointer;
    color: #EBC256;
  }

  #fullscreen-button span {
    opacity: 0;
    white-space: nowrap;
    line-height: 40px;
    transition: all 0s;
  }

  #fullscreen-button img {
    position: absolute;
    top: 10px;
    right: 10px;
  }

  #fullscreen-button:hover img.normal {
    opacity: 0;
    transition: all 0s;
  }

  #fullscreen-button:hover span {
    opacity: 1;
  }

  @media screen and (max-width: 992px) {
    #fullscreen-button {
      display: none;
    }

    #status-bar #ticker {
      -webkit-animation: ticker 60s linear infinite; /* Safari 4+ */
      -moz-animation:    ticker 60s linear infinite; /* Fx 5+ */
      -o-animation:      ticker 60s linear infinite; /* Opera 12+ */
      animation:         ticker 60s linear infinite; /* IE 10+, Fx 29+ */
    }

    -webkit-keyframes ticker {
      0%   { transform: translate3d(0, 0, 0); }
      50% { transform: translate3d(calc(100vw - 160px - 816px), 0, 0); }
      100%   { transform: translate3d(0, 0, 0); }
    }
    @-moz-keyframes ticker {
      0%   { transform: translate3d(0, 0, 0); }
      50% { transform: translate3d(calc(100vw - 160px - 816px), 0, 0); }
      100%   { transform: translate3d(0, 0, 0); }
    }
    @-o-keyframes ticker {
      0%   { transform: translate3d(0, 0, 0); }
      50% { transform: translate3d(calc(100vw - 160px - 816px), 0, 0); }
      100%   { transform: translate3d(0, 0, 0); }
    }
    @keyframes ticker {
      0%   { transform: translate3d(0, 0, 0); }
      50% { transform: translate3d(calc(100vw - 160px - 816px), 0, 0); }
      100%   { transform: translate3d(0, 0, 0); }
    }
  }
</style>
