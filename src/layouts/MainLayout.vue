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
      <canvas class="center q-pt-xl" id="canvas">
        <q-tooltip ref="mapTooltip" v-if="countryName">{{ countryName }} <br>
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
    const SERVICE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InR1ZHdmY21meG9udnVjZnRzdnd6Iiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTY1MjM3NTkzMCwiZXhwIjoxOTY3OTUxOTMwfQ.SpsX5rMIwNio2RFf1PmsgwXW8tAF_CmQ94szG8tVFss';

    const supabaseUrl = 'https://tudwfcmfxonvucftsvwz.supabase.co';
    const supabaseKey = SERVICE_KEY;
    const supabase = createClient(supabaseUrl, supabaseKey);

    var chart = null;
    var options = {};
    var activeCountry = '';
    var countryName = '';
    var activeCases = 0;
    var activeDate = '2020-07-06';

    return {
      supabase,
      full_data: null,
      chart,
      options,
      activeCountry,
      countryName,
      activeCases,
      activeDate,
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
  },
  methods: {
    async getSingleDayData(country, date) {
      try {
        let { data: full_data, error } = await new Promise(resolve => {
          setTimeout(() => {
            resolve(
              this.supabase
                .from('full_data')
                  .select('*')
                    .eq('location', country)
                      .eq('date', date));
          }, 2000);
        });
        //console.log(full_data);
        this.full_data = full_data;
        this.activeCases = full_data[0].num_sequences_total;
        console.log(this.full_data);
        return full_data;
      } catch (err) {
        console.log(err);
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
    getHover(chart, evt) {
      var item = chart.getElementAtEvent(evt);
      console.log("onHover", item);
    },
  },
}
</script>

<style scoped>
  .center {
    width: 100%;
    max-width: 1200px;
    display: block;
    margin: auto;
  }
</style>
