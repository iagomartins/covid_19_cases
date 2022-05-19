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
      <div class="center q-pt-md">
        <div v-if="selectionType == 'Single Day'">
          <q-badge color="secondary">
            Days since pandemy manifestation: {{ numberOfDay * 14 }} <br>
            Selected date: {{ activeDate }}
          </q-badge>
          <q-slider :step="1" snap v-model="numberOfDay" :min="0" :max="13"/>
        </div>
        <div v-else>
          <q-badge color="secondary">
            Number of periods processed (Every two weeks): {{ periodSelected.max - periodSelected.min }} <br>
            First date: {{ activeDateRange.min }} <br>
            Last date: {{ activeDateRange.max }}
          </q-badge>
          <q-range :step="1" snap v-model="periodSelected" :min="0" :max="13"/>
        </div>
        <div class="flex justify-between q-py-sm center">
          <h5 class="date-pointer">Jul, 2020</h5>
          <h5 class="date-pointer">Jan, 2021</h5>
        </div>
      </div>
      <canvas class="map center q-pt-md" id="canvas"></canvas>
      <h3 class="q-py-md text-center text-primary" v-if="countryName">{{ countryName }}</h3>
      <div class="info-box shadow-10" v-if="countryName">
        <span v-for="item in full_data" :key="item.variant">
          Variant: {{ item.variant }}<br>
          Number of variant cases: {{ item.num_sequences }}<br>
        </span>
        <span>Total cases: {{ activeCasesTotal }}</span>
      </div>
      <div class="info-box-loading shadow-10">
            <q-circular-progress
              indeterminate
              size="50px"
              :thickness="0.22"
              color="primary"
              track-color="grey-3"
              class="q-ma-md"
            />
      </div>
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

    const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InR1ZHdmY21meG9udnVjZnRzdnd6Iiwicm9sZSI6ImFub24iLCJpYXQiOjE2NTI3MzU0NzMsImV4cCI6MTk2ODMxMTQ3M30.zoj-O87L0Ox_SXFaw9cAcq2p6Vo6h3aKxQQoMNuQ6nw';

    const supabaseUrl = 'https://tudwfcmfxonvucftsvwz.supabase.co';
    const supabaseKey = SUPABASE_KEY;
    const supabase = createClient(supabaseUrl, supabaseKey);

    const PERIOD_DAYS = 14;

    var chart = null;
    var options = {};
    var activeCountry = '';
    var countryName = '';
    var activeCasesTotal = '';
    var activeDate = '2020-07-06';
    var activeDateRange = {
      min: '2020-07-06',
      max: '2020-08-31'
    };
    var selectedDates = [];
    var periodSelected = {
      min: 0,
      max: 10
    };
    var rangeDataResult = [];
    var tooltipObject = this.$refs.mapTooltip;
    var numberOfDay = 0;
    var selectionType = 'Single Day';

    window.tooltipObject = tooltipObject;

    return {
      PERIOD_DAYS,
      supabase,
      full_data: null,
      chart,
      options,
      activeCountry,
      countryName,
      activeCasesTotal,
      activeDate,
      activeDateRange,
      rangeDataResult,
      rangeDateMin: 0,
      rangeDateMax: 0,
      selectedDates,
      tooltipObject,
      numberOfDay,
      periodSelected,
      selectionType,
      panelX: 0,
      panelY: 0,
    }
  },
  created() {},
  mounted() {
    this.createMundiMap();
  },
  watch: {
    activeCountry(newValue) {
      this.countryName = newValue.raw.feature.properties.name;
      if (this.selectionType == 'Single Day') {        
        this.full_data = this.getSingleDayData(this.countryName, this.activeDate);
      } else {
        this.full_data = this.getDataBetweenTwoDates(this.countryName, this.rangeDataResult);
      }
    },
    numberOfDay(newValue, oldValue) {
      if(newValue > oldValue) {
        this.activeDate = this.getDateBySliderValueGreater(oldValue, this.activeDate, newValue);
      }
      else {
        this.activeDate = this.getDateBySliderValueLess(oldValue, this.activeDate, newValue);
      }
    },
    selectionType(newValue) {
      if(newValue == 'Single Day') {
        this.activeDate = '2020-07-06';
        this.numberOfDay = 0;
      }
      else {
        this.activeDateRange = {
          min: '2020-07-06',
          max: '2020-07-06'
        };
        this.periodSelected = {
          min: 0,
          max: 4
        };
        this.rangeDataResult = this.getPeriodBetweenTwoDates(this.activeDateRange.min, this.activeDateRange.max);
      }
    },
    periodSelected(newValue, oldValue) {
      if(newValue.max != oldValue.max) {
        this.rangeDateMax = newValue.max;
      }
      else if (newValue.min != oldValue.min) {
        this.rangeDateMin = newValue.min;
      }
    },
    rangeDateMin(newValue, oldValue) {
      if(newValue > oldValue) {
        this.activeDateRange.min = this.getDateBySliderValueGreater(oldValue, this.activeDateRange.min, newValue);
      }
      else {
        this.activeDateRange.min = this.getDateBySliderValueLess(oldValue, this.activeDateRange.min, newValue);
      }
      this.rangeDataResult = this.getPeriodBetweenTwoDates(this.activeDateRange.min, this.activeDateRange.max);
      console.log(this.rangeDataResult);
    },
    rangeDateMax(newValue, oldValue) {
      if(newValue > oldValue) {
        this.activeDateRange.max = this.getDateBySliderValueGreater(oldValue, this.activeDateRange.max, newValue);
      }
      else {
        this.activeDateRange.max = this.getDateBySliderValueLess(oldValue, this.activeDateRange.max, newValue);
      }
      this.rangeDataResult = this.getPeriodBetweenTwoDates(this.activeDateRange.min, this.activeDateRange.max);
      console.log(this.rangeDataResult);
    },
  },
  methods: {
    async getSingleDayData(country, date) {
      let panel = document.querySelector('.info-box');
      let loadingBox = document.querySelector('.info-box-loading');
      try {
        this.activeCasesTotal = 'Calculating...';

        // Start loading...
        loadingBox.style.opacity = 1;
        loadingBox.style.top = this.panelY + 'px';
        loadingBox.style.left = this.panelX + 'px';

        panel.style.opacity = 0;
        panel.style.pointerEvents = 'none';
        panel.style.top = this.panelY + 'px';
        panel.style.left = this.panelX + 'px';

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

        this.full_data = full_data;
        console.log(this.full_data);
        this.activeCasesTotal = full_data[0].num_sequences_total;
        loadingBox.style.opacity = 0;
        // Stop loading...

        panel.style.opacity = 1;
        panel.style.pointerEvents = 'all';
        panel.style.top = this.panelY + 'px';
        panel.style.left = this.panelX + 'px';
        return full_data;
      } catch (err) {
        console.log(err);
        this.activeCasesTotal = 'No data available';
        loadingBox.style.opacity = 0;
        panel.style.opacity = 1;
        panel.style.pointerEvents = 'all';
        panel.style.top = this.panelY + 'px';
        panel.style.left = this.panelX + 'px';
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
              vm.panelX = e.x;
              vm.panelY = e.y;
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
      let daysCount = this.PERIOD_DAYS * period;
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
      let daysCount = this.PERIOD_DAYS * period;
      for (let index = oldValue; index > oldValue - daysCount; index--) {
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
    getPeriodBetweenTwoDates(firstDate, lastDate) {
      let ret = [];
      let firstDateElement = firstDate.split('-');
      let lastDateElement = lastDate.split('-');
      let firstYear = parseInt(firstDateElement[0]);
      let firstMonth = parseInt(firstDateElement[1]);
      let firstDay = parseInt(firstDateElement[2]);
      let lastYear = parseInt(lastDateElement[0]);
      let lastMonth = parseInt(lastDateElement[1]);
      let lastDay = parseInt(lastDateElement[2]);
      let year = firstYear;
      let month = firstMonth;
      let day = firstDay;
      let newDate = '';

      ret.push(firstDate);

      while (year < lastYear || month < lastMonth || day < lastDay) {
        for (let index = 0; index < this.PERIOD_DAYS; index++) {
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
        }
        ret.push(newDate);
      }
      return ret;
    },
    async getDataBetweenTwoDates(country, period) {
      let panel = document.querySelector('.info-box');
      let loadingBox = document.querySelector('.info-box-loading');
      let ret = [];
      //console.log(period);

      // Start loading
      loadingBox.style.opacity = 1;
      this.activeCasesTotal = 'Calculating...';
      panel.style.opacity = 0;
      panel.style.pointerEvents = 'none';
      panel.style.top = this.panelY + 'px';
      panel.style.left = this.panelX + 'px';
      for (let index = 0; index < period.length; index++) {
        const element = period[index];
        let response = [];
        try {
          const dayData = await new Promise(resolve => {
            setTimeout(() => {
              resolve(
                this.supabase
                  .from('full_data')
                    .select('*')
                      .eq('location', country)
                        .eq('date', element));
            }, 3000);
          });
          if (dayData.data.length > 0) {
            response = dayData.data;
            if (index === 0) {
              response.forEach(element => {
                element.num_sequences = parseInt(element.num_sequences);
                ret.push(element);
              });
            }
          }
        }
        catch (error) {
          console.log(error);
          response = [];
        }

        if (response.length > 0) {
          if (index > 0) {
            for (let i = 0; i < ret.length; i++) {
              ret[i].num_sequences += parseInt(response[i].num_sequences);
              ret[i].num_sequences_total += parseInt(response[i].num_sequences_total);
            }
          }
        }
      }
      
      loadingBox.style.opacity = 0;
      // Stop loading...

      if (ret.length == 0) {
        this.activeCasesTotal = 'No data available';
      }

      // Total number of cases.

      console.log(ret);
      this.activeCasesTotal = ret[0].num_sequences_total;
      this.full_data = ret;
      panel.style.opacity = 1;
      panel.style.pointerEvents = 'all';
      panel.style.top = this.panelY + 'px';
      panel.style.left = this.panelX + 'px';
      return ret;
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
  .info-box {
    position: fixed;
    z-index: 2001;
    transform-origin: 0  0;
    opacity: 0;
    pointer-events: none;
    display: grid;
    border-radius: 6px;
    grid-template-columns: repeat(3, 1fr);
    background: #fff;
    height: auto;
    transition: all 0.5s cubic-bezier(0.075, 0.82, 0.165, 1);
    overflow: hidden;
    padding: 32px;
    grid-gap: 3px;
  }
  .info-box-loading {
    position: fixed;
    z-index: 2001;
    transform-origin: 0  0;
    opacity: 0;
    pointer-events: none;
    display: block;
    border-radius: 6px;
    padding: 32px;
    background: #fff;
    transition: all 0.5s cubic-bezier(0.075, 0.82, 0.165, 1);
  }
  .info-box span {
    border: 1px solid #cecece;
    color: #121212;
    padding: 5px;
    font-size: 10px;
    max-width: 145px;
  }
</style>
