<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>

        <q-toolbar-title>
          Covid Daily Cases
        </q-toolbar-title>

        <div>Quasar v{{ $q.version }}</div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <div class="center q-py-md flex justify-end">
        <q-select outlined label="Selection type:" dense v-model="selectionType" :options="['Single Day', 'Period']"></q-select>
      </div>
      <div class="center q-pt-xl">
        <div v-if="selectionType == 'Single Day'">
          <q-badge color="secondary">
            Days since pandemy manifestation: {{ numberOfDay * 14 }} <br>
            Selected date: {{ activeDate }}
          </q-badge>
          <q-slider v-model="numberOfDay" :min="0" :max="50"/>
        </div>
        <div v-else>
          <q-badge color="secondary">
            Days since pandemy manifestation: {{ numberOfDay * 14 }} <br>
            Selected date: {{ activeDate }}
          </q-badge>
          <q-range v-model="periodSelected" :min="0" :max="50"/>
        </div>
        <div class="flex justify-between q-py-lg center">
          <h5 class="date-pointer">Jul, 2020</h5>
          <h5 class="date-pointer">Jan, 2021</h5>
          <h5 class="date-pointer">Jul, 2021</h5>
          <h5 class="date-pointer">Jan, 2022</h5>
        </div>
      </div>
      <canvas class="center q-pt-xl" id="canvas">
        <q-tooltip v-if="countryName">{{ countryName }} <br>
          Cases: {{ activeCases }} <br>
        </q-tooltip>
      </canvas>
    </q-page-container>
  </q-layout>
</template>

<script>
import { createClient } from '@supabase/supabase-js';
import { Chart } from 'chart.js';
import { ChoroplethController, GeoFeature, ColorScale, ProjectionScale } from 'chartjs-chart-geo';
import * as ChartGeo from 'chartjs-chart-geo';

export default {
  name: 'MainLayout',
  components: {
    //EssentialLink
  },
  data () {
    window.page = this;

    const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InR1ZHdmY21meG9udnVjZnRzdnd6Iiwicm9sZSI6ImFub24iLCJpYXQiOjE2NTIzNzU5MzAsImV4cCI6MTk2Nzk1MTkzMH0.Wy9QiOnilCZXuvHzMePxaIAOMgEF0dNfyNMT-mmVfwA';
    const SERVICE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InR1ZHdmY21meG9udnVjZnRzdnd6Iiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTY1MjQ3OTkxOSwiZXhwIjoxOTY4MDU1OTE5fQ.q1lOOVzi5PyhOzq_ealudq6DIg4Pk0VFpyvevJTQb9g';

    const supabaseUrl = 'https://tudwfcmfxonvucftsvwz.supabase.co';
    const supabaseKey = SERVICE_KEY;
    const supabase = createClient(supabaseUrl, supabaseKey);

    var chart = null;
    var options = {};
    var activeCountry = '';
    var countryName = '';
    var activeCases = 0;
    var activeDate = '2020-07-06';
    var periodSelected = {
      min: 0,
      max: 10
    };
    var tooltipObject = this.$refs.mapTooltip;
    var numberOfDay = 0;
    var selectionType = 'Single Day';

    window.tooltipObject = tooltipObject;

    return {
      supabase,
      full_data: null,
      chart,
      options,
      activeCountry,
      countryName,
      activeCases,
      activeDate,
      tooltipObject,
      numberOfDay,
      periodSelected,
      selectionType,
    }
  },
  created() {
    //this.full_data = this.getSingleDayData();
  },
  mounted() {
    this.createMundiMap();
  },
  watch: {
    activeCountry(newValue) {
      //console.log(newValue);
      this.countryName = newValue.raw.feature.properties.name;
      this.full_data = this.getSingleDayData(this.countryName, this.activeDate);
    },
    numberOfDay(newValue, oldValue) {
      if(newValue > oldValue) {
        this.activeDate = this.getDateBySliderValueGreater(oldValue, this.activeDate, newValue);
      }
      else {
        this.activeDate = this.getDateBySliderValueLess(oldValue, this.activeDate, newValue);
      }
    },
  },
  methods: {
    async getSingleDayData(country, date) {
      try {
        this.activeCases = 'Calculating...';
        let { data: full_data, error } = await new Promise(resolve => {
          setTimeout(() => {
            resolve(
              this.supabase
                .from('full_data')
                  .select('*')
                    .eq('location', country)
                      .eq('date', date));
          }, 3000);
        });
        //console.log(full_data);
        this.full_data = full_data;
        console.log(this.full_data);
        this.activeCases = full_data[0].num_sequences_total;
        return full_data;
      } catch (err) {
        console.log(err);
        this.activeCases = 'Unregistered data.';
      }
    },
    createMundiMap() {
      let vm = this;
      fetch('https://unpkg.com/world-atlas/countries-50m.json').then((r) => r.json()).then((data) => {
            const countries = ChartGeo.topojson.feature(data, data.objects.countries).features;
            const myData = {
                labels: countries.map((d) => d.properties.name),
                datasets: [{
                  label: 'Countries',
                  data: countries.map((d) => ({feature: d, value: Math.random()})),
                }],
            };

            const chart = new Chart(document.getElementById("canvas").getContext("2d"), {
              type: 'choropleth',
              data: myData,
              options: {
                showOutline: true,
                showGraticule: true,
                interaction: {
                  intersect: false,
                  mode: 'nearest',
                  axis: 'xy',
                },
                plugins: {
                  title: {
                    display: true,
                  },
                  tooltip: {
                    enabled: true,
                  }
                },
                scales: {
                  xy: {
                    projection: 'equalEarth'
                  }
                },
              },
            });

            chart._options.onHover = function(e) {
              vm.activeCountry = chart.getElementsAtEventForMode(e, 'nearest', { intersect: false }, true)[0].element.$context;
            };

            window.chart = chart;
            vm.chart = chart;
      });
    },
    getDateBySliderValueGreater(oldValue, date, newValue) {
      const element = date.split('-');
      let year = parseInt(element[0]);
      let month = parseInt(element[1]);
      let day = parseInt(element[2]);
      let newDate = '';
      let period = newValue - oldValue;
      let daysCount = 14 * period;
      for (let index = oldValue; index < oldValue + daysCount; index++) {
        if (month == 2) {
          if (day == 28) {
            day = 1;
            month = 3;
          }
          else {
            day++;
          }
        }
        else if (month == 4 || month == 6 || month == 9 || month == 11) {
          if(day == 30) {
            day = 1;
            month++;
          }
          else {
            day++;
          }
        }
        else if (month === 12) {
          if (day == 31) {
            month = 1;
            year++;
            day = 1;
          }
          else {
            day++;
          }
        }
        else {
          if (day == 31) {
            day = 1;
            month++;
          }
          else {
            day++;
          }
        }
      }
      if (day < 10) {
        if (month < 10) {
          newDate = year + '-0' + month + '-0' + day;
        }
        else {
          newDate = year + '-' + month + '-0' + day;
        }
      }
      else {
        if (month < 10) {
          newDate = year + '-0' + month + '-' + day;
        }
        else {
          newDate = year + '-' + month + '-' + day;
        }
      }
      return newDate;
    },
    getDateBySliderValueLess(oldValue, date, newValue) {
      const element = date.split('-');
      let year = parseInt(element[0]);
      let month = parseInt(element[1]);
      let day = parseInt(element[2]);
      let newDate = '';
      let period = oldValue - newValue;
      let daysCount = 14 * period;
      for (let index = oldValue; index > newValue - daysCount; index--) {
        if (month == 3) {
          if (day == 1) {
            day = 28;
            month = 2;
          }
          else {
            day--;
          }
        }
        else if (month == 2 || month == 4 || month == 6 || month == 8 || month == 9 || month == 11) {
          if (day == 1) {
            day = 31;
            month--;
          }
          else {
            day--;
          }
        }
        else if (month === 1) {
          if (day == 1) {
            month = 12;
            year--;
            day = 31;
          }
          else {
            day--;
          }
        }
        else {
          if (day == 1) {
            day = 30;
            month--;
          }
          else {
            day--;
          }
        }
      }
      if (day < 10) {
        if (month < 10) {
          newDate = year + '-0' + month + '-0' + day;
        }
        else {
          newDate = year + '-' + month + '-0' + day;
        }
      }
      else {
        if (month < 10) {
          newDate = year + '-0' + month + '-' + day;
        }
        else {
          newDate = year + '-' + month + '-' + day;
        }
      }
      return newDate;
    },      
  },
}
</script>

<style scoped>
  .center {
    width: 100%;
    max-width: 1200px;
    margin: auto;
  }
  .date-pointer {
    width: fit-content;
    width: -moz-fit-content;
  }
  .info-tooltip {
    position: fixed;
    z-index: 9999;
    box-shadow: 0px 0px 5px #000000;
  }
</style>
