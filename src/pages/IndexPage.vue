<template>
  <div>
    <q-card
      style="width: 600px; background-color: #214d86; color: white"
      v-if="loaded"
    >
      <q-bar>Current Weather - Sourek Trail Akron Ohio</q-bar>
      <q-card-section>
        <div class="row">
          <div class="col-sm-auto">
            <img :src="forecastIcon" />
          </div>
          <div class="col-sm-auto" style="font-size: 26px">
            {{ temperature }}&#176;F
          </div>
          <div
            class="col-sm-3 offset-sm-1"
            style="margin-top: 7px"
            v-if="forecast.length"
          >
            {{ forecastBrief }}
          </div>
        </div>
        <div class="row">
          <div class="col-auto" v-if="forecast.length">
            {{ forecastText }}
          </div>
        </div>

        <div class="row mt-10">
          <div class="col-sm-8">
            <table>
              <tr>
                <td>
                  High: {{ today.hiTemp }}
                  <span style="font-size: 8px">{{ today.hiTempTime }}</span>
                </td>
              </tr>
              <tr>
                <td>
                  Low: {{ today.lowTemp }}
                  <span style="font-size: 8px">{{ today.lowTempTime }}</span>
                </td>
              </tr>
            </table>
          </div>
        </div>

        <div class="row mt-10">
          <div class="col-sm-10">
            <table style="width: 100%; text-align: center">
              <tr>
                <td>Wind</td>
                <td>Humidity</td>
                <td v-if="heatIndex > 60">Heat Index</td>
                <td v-if="windChill < 60">Wind Chill</td>
                <td>Barometer</td>
                <td>Dewpoint</td>
              </tr>
              <tr>
                <td>{{ windSpeed }} {{ windDirection }}</td>
                <td>{{ humidity }}%</td>
                <td v-if="heatIndex > 60">{{ heatIndex }}&#176;</td>
                <td v-if="windChill < 60">{{ windChill }}</td>
                <td>{{ barometer }} {{ barometerTrend }}</td>
                <td>{{ dewpoint }}&#176;</td>
              </tr>
            </table>
          </div>
        </div>

        <div class="row mt-10">
          <div class="col-sm-9">
            <table style="width: 100%; text-align: center">
              <tr>
                <td>Rain Today</td>
                <td>Rain Rate</td>
                <td v-if="stormRain">Storm Rain</td>
                <td>Month Rain</td>
                <td>Year Rain</td>
              </tr>
              <tr>
                <td>{{ dayRain }}</td>
                <td>{{ rainRate }}</td>
                <td v-if="stormRain">
                  {{ stormRain }} since {{ stormStartDt }}
                </td>
                <td>{{ monthRain }}</td>
                <td>{{ yearRain }}</td>
              </tr>
            </table>
          </div>
        </div>
        <article>
          <span style="text-decoration: underline"> Tonight</span>
          <article class="forecast">{{ tonight }}</article>
        </article>
        <article>
          <span style="text-decoration: underline">Tomorrow</span>
          <article class="forecast">{{ tomorrow }}</article>
        </article>
        <article>
          <span style="text-decoration: underline">{{ nextDay.name }} </span>
          <article class="forecast">{{ nextDay.text }}</article>
        </article>

        <div class="row mt-10">
          <div class="col-sm-12">
            Sunrise: {{ sunrise }} &nbsp;&nbsp;Sunset: &nbsp;{{ sunset }}
          </div>
        </div>
        <div class="row mt-10" style="font-size: 10px">
          <div class="col-sm-12">Source: Vantage Vue Weather Station</div>
          <div class="col-sm-12">Forecast by Wunderground</div>
          <div class="col-sm-12">
            <span>Updated {{ lastUpdate }}</span> seconds ago
            <span v-if="updateSwitch">*</span>
          </div>
        </div>
      </q-card-section>
    </q-card>
  </div>
</template>
<script lang="ts">
import { computed, onMounted, reactive, toRefs } from 'vue';
import moment from 'moment';

function getImageUrl(name) {
  return new URL(`../assets/${name}`, import.meta.url).href;
}

export default {
  name: 'WS Weather',

  setup() {
    //setup is for using compostion api rather than options api
    let tmr;
    let wsocket: WebSocket;
    //vm is the object used for data binding in the view
    const vm = reactive({
      forecast: [{ text: '', icon: '', short: '', name: '' }],
      today: {
        hiTemp: null,
        lowTemp: null,
        hiTempTime: '',
        lowTempTime: '',
        hiWind: null,
        hiWindTime: null,
        hourRain: 0,
      },
      alerts: '',
      barometer: null,
      barometerTrend: null,
      dayRain: null,
      dewpoint: 0,
      humidity: 0,
      lastUpdated: new Date(),
      loaded: false,
      monthRain: null,
      rainRate: null,
      stormDate: '',
      stormRain: null,
      sunrise: null,
      sunset: null,
      temperature: 0,
      updated: new Date(),
      updateSwitch: false,
      windDirCSS: null,
      windAvg: null,
      windChill: 0,
      windDirection: null,
      windSpeed: null,
      yearRain: null,
    });

    onMounted(() => {
      connect();
    });
    //internal methods...
    const connect = () => {
      const protocol = window.location.protocol == 'https:' ? 'wss' : 'ws';
      const hostname = protocol + '://smart-app.live/vpws/vantage/clientws';
      wsocket = new WebSocket(hostname);

      try {
        wsocket.onopen = () => {
          if (tmr) {
            clearInterval(tmr);
            tmr = null;
            console.log('reconnected');
          } else {
            console.log('ws connected');
          }
        };

        wsocket.onmessage = (evnt) => {
          const data = evnt.data;
          if (data.startsWith('{')) {
            const ws = JSON.parse(data);
            if (ws.current) {
              Object.assign(vm, ws.current);
              vm.lastUpdated = vm.updated;
              vm.updated = new Date();
              vm.updateSwitch = !vm.updateSwitch;
              vm.temperature = round(vm.temperature);
              vm.dewpoint = round(vm.dewpoint);
              vm.windChill = round(vm.windChill);
              vm.loaded = true;
            }
            if (ws.hiLows) {
              updateDaily(ws.hiLows);
            } else {
            }
            if (ws.alerts) {
              vm.alerts =
                ws.alerts && ws.alerts.length ? ws.alerts[0].message : '';
            }
          }
        };
        wsocket.onclose = () => {
          reconnect();
        };
        wsocket.onerror = () => {
          reconnect();
        };
      } catch (e) {
        console.log('socket error:' + e);
        reconnect();
      }
    };

    const reconnect = () => {
      tmr = setInterval(() => {
        connect();
      }, 10000);
    };

    const updateDaily = (hilows) => {
      vm.forecast = [];

      if (hilows.forecast) {
        for (const period in hilows.forecast) {
          let fcast = hilows.forecast[period];
          vm.forecast.push({
            text: fcast.longText,
            short: fcast.shortText,
            icon: fcast.icon,
            name: period
              ? period.substr(0, 1).toUpperCase() + period.substr(1)
              : '',
          });
        }
      }

      vm.today = {
        hiTemp: hilows.temperature.dailyHi,
        lowTemp: hilows.temperature.dailyLow,
        hiTempTime: formatTime(hilows.temperature.dailyHighTime),
        lowTempTime: formatTime(hilows.temperature.dailyLowTime),
        hiWind: hilows.windSpeed.dailyHi,
        hiWindTime: hilows.windSpeed.dailyHighTime,
        hourRain: round(hilows.rain1hour / 100, 2),
      };

      if (isNaN(vm.today.hourRain)) {
        vm.today.hourRain = 0;
      }
    };
    const round = (nbr, precision?) => {
      if (typeof nbr === 'string') {
        nbr = parseFloat(nbr);
      }
      if (!precision) {
        precision = 0;
      }
      return parseFloat(nbr.toFixed(precision));
    };

    const formatTime = (time): string => {
      return moment(time, 'HH:mm:ss').format('hh:mm A');
    };

    //computed...
    const tonight = computed(() => {
      return vm.forecast.length > 1 ? vm.forecast[1].text : '';
    });
    const tomorrow = computed(() => {
      return vm.forecast.length > 2 ? vm.forecast[2].text : '';
    });
    const nextDay = computed(() => {
      const nday = { name: '', text: '' };
      if (vm.forecast.length >= 5) {
        nday.name = vm.forecast[4].name;
        nday.text = vm.forecast[4].text;
      }
      return nday;
    });
    const forecastBrief = computed(() => {
      return vm.forecast.length ? vm.forecast[0].short : '';
    });
    const forecastText = computed(() => {
      return vm.forecast.length ? vm.forecast[0].text : '';
    });

    const forecastIcon = computed(() => {
      console.log(vm.forecast);
      if (vm.forecast.length == 0 || !vm.forecast[0].icon) return '';
      try {
        const icon = getImageUrl(vm.forecast[0].icon + '.svg');
        return icon;
      } catch (e) {
        console.log(e);
        return '';
      }
    });
    const stormStartDt = computed(() => {
      return vm.stormDate
        ? vm.stormDate.substr(5, 5) + '-' + vm.stormDate.substr(0, 4)
        : '';
    });
    const lastUpdate = computed(() => {
      if (vm.updated == undefined) {
        return '';
      }
      const diff = Math.abs(vm.updated - vm.lastUpdated);
      const seconds = Math.floor(diff / 1000);
      //let minutes = Math.floor(seconds / 60);
      //const hours = Math.floor(minutes / 60);
      //minutes = minutes % 60;

      return seconds;
    });
    const heatIndex = computed(() => {
      if (!vm.temperature || !vm.humidity) {
        return '';
      }
      const temp = vm.temperature;
      const humidity = vm.humidity;
      let hi = 0.5 * (temp + 61 + (temp - 68) * 1.2 + humidity * 0.094);
      if (temp >= 80) {
        const heatIndexBase =
          -42.379 +
          2.04901523 * temp +
          10.14333127 * humidity +
          -0.22475541 * temp * humidity +
          -0.00683783 * temp * temp +
          -0.05481717 * humidity * humidity +
          0.00122874 * temp * temp * humidity +
          0.00085282 * temp * humidity * humidity +
          -0.00000199 * temp * temp * humidity * humidity;
        // adjustment
        if (humidity < 13 && temp <= 112) {
          hi =
            heatIndexBase -
            ((13 - humidity) / 4) * Math.sqrt((17 - Math.abs(temp - 95)) / 17);
        } else if (humidity > 85 && temp <= 87) {
          hi = heatIndexBase + ((humidity - 85) / 10) * ((87 - temp) / 5);
        } else {
          hi = heatIndexBase;
        }
      }

      return round(hi);
    });

    //returns vm and computed methods
    return {
      ...toRefs(vm),
      heatIndex,
      lastUpdate,
      forecastBrief,
      forecastIcon,
      forecastText,
      stormStartDt,
      tonight,
      tomorrow,
      nextDay,
    };
  },
};
</script>
<style>
.mt-10 {
  margin-top: 10px;
}
.forecast {
  margin-left: 5px;
  font-size: 12px;
}
</style>
