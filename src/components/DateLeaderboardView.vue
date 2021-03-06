<template>
  <div class="leaderboard">
    <h1 class="title has-text-centered">{{ title }}<img src="/calendar-icon.svg" alt="Select a date" v-el:calendar-icon></h1>
    <pulse-loader :class="'has-text-centered'" :loading="loading" :color="'#1fc8db'"></pulse-loader>
    <div class="columns is-desktop">
      <div class="column is-half is-offset-one-quarter">
        <div class="notification is-warning" v-show="noData">No data yet</div>
      </div>
    </div>
    <div class="columns is-desktop">
      <div class="column">
        <chart :type="'horizontalBar'" :data="barChartData" :options="barChartOptions" :height="barChartHeight" v-show="dataReady"></chart>
      </div>
      <div class="column">
        <h4 class="title has-text-centered is-4" v-show="dataReady">Stats</h4>
        <chart :type="'doughnut'" :data="doughnutChartData" :options="doughnutChartOptions" v-show="dataReady"></chart>
      </div>
    </div>
  </div>
</template>

<script>
import _ from 'lodash';
import moment from 'moment';
import Pikaday from 'pikaday';

import Chart from './Chart.vue';
import PulseLoader from 'vue-spinner/src/PulseLoader.vue';
import store from '../store';

export default {
  name: 'DateLeaderboardView',
  components: {
    Chart,
    PulseLoader
  },
  data() {
    return {
      title: 'Leaderboard',
      loading: true,
      updating: false,
      interval: [store.today],
      datePicker: null,
      barChartData: {
        labels: [],
        datasets: []
      },
      barChartOptions: {
        scales: {
          xAxes: [{ stacked: true }],
          yAxes: [{ stacked: true }]
        },
        legend: { display: false }
      },
      barChartHeight: 400,
      doughnutChartData: {
        labels: [],
        datasets: []
      },
      doughnutChartOptions: {}
    };
  },
  computed: {
    dataReady() {
      return !this.loading && this.barChartData.labels.length;
    },
    noData() {
      return !this.loading && !this.barChartData.labels.length && this.interval;
    }
  },
  route: {
    data({ to: { path, params: { year, month, day }}}) {
      const today = new Date();
      switch (path) {
        case '/today':
          this.interval = [
            store.today.unix(),
            store.today.unix()
          ];
          this.title = 'Leaderboard for today';
          break;
        case '/yesterday':
          this.interval = [
            store.yesterday.unix(),
            store.yesterday.unix()
          ];
          this.title = 'Leaderboard for yesterday';
          break;
        case '/alltime':
          this.interval = [
            moment('2016-06-01').unix(),
            store.today.unix()
          ];
          this.title = 'All-time leaderboard';
          break;
        case '/week':
          this.interval = [
            store.today.clone().subtract(1, 'w').unix(),
            store.today.unix()
          ];
          this.title = 'Leaderboard for the past week';
          break;
        default:
          const date = moment({ year, month: month - 1, day });
          this.interval = [date.unix(), date.unix()];
          this.title = `Leaderboard for ${date.format('dddd, MMMM D')}`;
          break;
      }

      this.reload();
    }
  },
  ready() {
    this.datePicker = new Pikaday({
      field: this.$els.calendarIcon,
      firstDay: 1,
      onSelect: date => {
        const momentDate = moment(date);
        this.$route.router.go(`/${moment(date).format('YYYY/MM/DD')}`);
      }
    });
  },
  created() {
    store.on('leaderboard-updated', this.update);
  },
  destroyed() {
    store.removeListener('leaderboard-updated', this.update);
  },
  methods: {
    reload() {
      this.barChartData = {
        labels: [],
        datasets: []
      };
      this.doughnutChartData = {
        labels: [],
        datasets: []
      };
      this.loading = true;

      this.update();
    },
    update() {
      if (this.updating || !this.interval) return;

      this.updating = true;

      store
        .fetchDateLeaderboard(this.interval)
        .then(leaderboard => {
          this.loading = false;
          this.barChartData = {
            labels: _.map(
              leaderboard,
              stats => `${stats.user.first_name } ${ stats.user.last_name }`
            ),
            datasets: _.map(_.keys(store.colors), color => ({
              label: color,
              backgroundColor: store.colors[color],
              data: _.map(leaderboard, `points.${color}`)
            }))
          };
          this.barChartHeight = Math.max(leaderboard.length * 60, 400);
          this.doughnutChartData = {
            labels: ['Bronze', 'Silver', 'Gold'],
            datasets: [{
              data: _.reduce(leaderboard, (data, stats) => [
                data[0] + stats.bronze,
                data[1] + stats.silver,
                data[2] + stats.gold
              ], [0, 0, 0]),
              backgroundColor: [
                store.colors.bronze,
                store.colors.silver,
                store.colors.gold
              ],
              label: 'Leaderboard'
            }]
          };
          this.updating = false;
        });
    }
  },
  watch: {
    title(title) {
      document.title = title;
    }
  }
}
</script>

<style lang="stylus" scoped>
.title img
  margin-left: 10px
</style>
