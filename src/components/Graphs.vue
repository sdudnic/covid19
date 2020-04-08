<template>
  <v-container>
    <v-row>
      <v-col>
        <v-row align="center" justify="center">
          <v-autocomplete
            dense
            :label="strings.chart.country"
            v-model="countryCode"
            :items="countries"
            style="max-width: 250px"
            item-text="name"
            item-value="iso3"
            hide-details
            class
          ></v-autocomplete>
          <v-text-field
            :label="strings.chart.daysBack"
            v-model="daysNumber"
            disabled
            dense
            style="max-width: 45px"
            type="number"
            maxlength="3"
            hide-details
            class="ml-4"
          ></v-text-field>
        </v-row>
        <v-row class="ma-2 pa-0" align="center" justify="center">
          <small>
            {{strings.lastData}}
            <span
              id="lastApiUpdate"
              class="font-weight-bold"
            >{{lastUpdateTime}}</span>
            (vs {{previousUpdateTime}})
          </small>
        </v-row>
        <v-row align="center" justify="center">
          <v-card
            v-for="(color, i) in colors"
            :key="i"
            outlined
            :style="{opacity: (cases.delta[i]===0? .5 : 1)}"
            class="ma-1 pa-2 text-center"
          >
            <div class="overline font-weight-bold" :style="{color:color}">{{strings.chart[i]}}</div>
            <div :style="{borderTop: 'thin solid rgba(0,0,0,.12)', color:color}">
              + {{cases.delta[i]}}
              <small>({{Number(cases.total[i]).toLocaleString()}})</small>
            </div>
          </v-card>
        </v-row>
      </v-col>
    </v-row>

    <v-row justify="center">
      <v-card class="ma-2 pa-2">
        <GChart
          id="deltaDataChart"
          type="LineChart"
          :settings="{ 'packages':['corechart'], language: language}"
          :data="deltaDataTable"
          :options="deltaChartOptions"
          @ready="onChartReady"
        />
      </v-card>
      <v-card class="ma-2 pa-2">
        <GChart
          id="totalDataChart"
          type="LineChart"
          :settings="{ 'packages':['corechart'], language: language}"
          :data="totalDataTable"
          :options="totalChartOptions"
          @ready="onChartReady"
        />
      </v-card>
    </v-row>
  </v-container>
</template>

<script>
import { GChart } from "vue-google-charts";

export default {
  name: "Graphs",
  components: {
    GChart
  },
  props: {
    strings: { type: Object, required: true },
    language: { type: String, required: true },
    colors: { type: Object, required: true }
  },
  data() {
    return {
      ready: { totalChart: false, deltaChart: false },
      daysNumber: 5,
      countryCode: "",
      cases: {
        total: {
          confirmed: 0,
          recovered: 0,
          deaths: 0
        },
        delta: {
          confirmed: 0,
          recovered: 0,
          deaths: 0
        },
        updateDate: new Date()
      },
      totalChartOptions: {
        title: "Total cases",
        titleTextStyle: {
          color: "black",
          fontName: "Roboto",
          fontSize: 14,
          bold: false,
          italic: true,
          opacity: 0.5
        },
        //titlePosition: "out",
        //axisTitlesPosition: "in",
        fontName: "Roboto",
        curveType: "function",
        legend: { position: "none" },
        //chartArea: { left: 0, top: 0, width: "100%+40px", height: "calc(100%-5)" },
        height: 500,
        hAxis: {
          title: "",
          viewWindowMode: "maximized",
          format: "dd/MM"
          //viewWindow: { max: "2020-04-09" }
          //maxValue: new Date(2020, 4, 9)
        },
        vAxis: { viewWindow: { min: 0 }, format: "short", titlePosition: "in" },
        colors: Object.values(this.colors),
        pointSize: 10,
        pointShape: "diamond",
        theme: "maximized"
      },
      deltaChartOptions: {},
      current: {},
      previous: {},
      totalDataTable: {},
      deltaDataTable: {},
      countries: [],
      apiUrl: {
        timeline: "https://covidapi.info/api/v1/country/***",
        current:
          "https://services1.arcgis.com/0MSEUqKaxRlEPj5g/arcgis/rest/services/Coronavirus_2019_nCoV_Cases/FeatureServer/2/query?where=UPPER(Country_Region)%20like%20%27%25***%25%27&outFields=Last_Update,Confirmed,Deaths,Recovered,Country_Region&returnGeometry=false&outSR=4326&f=json"
      }
    };
  },
  methods: {
    chartsReady() {
      return this.ready.totalChart && this.ready.deltaChart;
    },

    getCountries() {
      var vm = this;
      return fetch("./data/countries.json")
        .then(s => s.json())
        .then(c => (vm.countries = c));
    },

    getApiUrl(countryCode, countryName, isTimeline) {
      if (!countryCode) {
        return null;
      }
      const placeholder = "***";
      var url = "";

      if (isTimeline) {
        url = this.apiUrl.timeline.replace(placeholder, countryCode);
      } else {
        // current value, no timeline
        if (countryName) {
          url = this.apiUrl.current.replace(placeholder, countryName);
        } else {
          return null;
        }
      }
      return url;
    },

    filterFromStartDate(e) {
      return e[0] >= this.startDateString;
    },
    updateFields(total, delta) {
      if (!total && !delta) {
        this.cases.updateDate = new Date(2000, 1, 1);

        this.cases.total.confirmed = 0;
        this.cases.total.recovered = 0;
        this.cases.total.deaths = 0;

        this.cases.delta.confirmed = 0;
        this.cases.delta.recovered = 0;
        this.cases.delta.deaths = 0;
      } else {
        this.cases.updateDate = (total || delta).date;
        if (total) {
          this.cases.total.confirmed = total.confirmed;
          this.cases.total.recovered = total.recovered;
          this.cases.total.deaths = total.deaths;
        }
        if (delta) {
          this.cases.delta.confirmed = delta.confirmed;
          this.cases.delta.recovered = delta.recovered;
          this.cases.delta.deaths = delta.deaths;
        }
      }
    },

    addColumns(dt) {
      var vm = this;
      dt.addColumn("date", "Date");
      dt.addColumn("number", vm.strings.chart.confirmed);
      dt.addColumn("number", vm.strings.chart.recovered);
      dt.addColumn("number", vm.strings.chart.deaths);
      dt.addColumn({
        role: "annotation",
        type: "string"
      });
    },

    onChartReady(chart, google) {
      var vm = this;
      var id = chart.container.id;
      var formatter = new google.visualization.DateFormat({
        pattern: "dd/MM/yy hh:mm"
      });

      if (id == "deltaDataChart") {
        vm.deltaDataTable = new google.visualization.DataTable();
        vm.addColumns(vm.deltaDataTable);
        formatter.format(vm.deltaDataTable, 0);
        vm.ready.deltaChart = true;
      } else if (id == "totalDataChart") {
        vm.totalDataTable = new google.visualization.DataTable();
        vm.addColumns(vm.totalDataTable);
        formatter.format(vm.totalDataTable, 0);
        vm.ready.totalChart = true;
      }
    },

    tryReloadData() {
      var vm = this;
      if (vm.readyToLoadData) {
        vm.loadData();
        return true;
      } else {
        return false;
      }
    },

    cleanTable(dataTable) {
      if (dataTable.getNumberOfRows) {
        var rowsNr = dataTable.getNumberOfRows();
        if (rowsNr) dataTable.removeRows(0, rowsNr);
      }
    },

    tableAddRow(table, row) {
      if (!row) {
        row = { date: null, confirmed: 0, recovered: 0, deaths: 0 };
      }
      table.addRow([row.date, row.confirmed, row.recovered, row.deaths, null]);
    },

    addDataEntry(current, previous, ignoreChartPoint) {
      var vm = this;
      var totalData = current;
      var deltaData = {
        date: totalData.date,
        confirmed: current.confirmed - previous.confirmed,
        recovered: current.recovered - previous.recovered,
        deaths: current.deaths - previous.deaths
      };

      if (!ignoreChartPoint) {
        vm.tableAddRow(vm.totalDataTable, totalData);
        vm.tableAddRow(vm.deltaDataTable, deltaData);
      }

      return deltaData;
    },

    loadData() {
      var vm = this;

      if (vm.totalDataTable.addRow) {
        vm.tableAddRow(vm.totalDataTable, null);
        vm.tableAddRow(vm.deltaDataTable, null);

        vm.cleanTable(vm.totalDataTable);
        vm.cleanTable(vm.deltaDataTable);
      }

      vm.previous = {
        date: {},
        confirmed: 0,
        recovered: 0,
        deaths: 0
      };

      vm.current = {
        date: {},
        confirmed: 0,
        recovered: 0,
        deaths: 0
      };

      fetch(vm.timelineApi)
        .then(a => a.json())
        .then(apiAnswer => {
          var results = Object.entries(apiAnswer.result).filter(
            vm.filterFromStartDate
          );

          if (results.length === 0) {
            // no data
            vm.tableAddRow(vm.totalDataTable, null);
            vm.tableAddRow(vm.deltaDataTable, null);
            vm.updateFields();
          } else {
            // we have data
            for (let i = 0; i < results.length; i++) {
              // take first one day before, for 'new' count
              var ignoreChartPoint = i == 0;
              const item = results[i][1];
              var myDate = new Date(results[i][0]);

              vm.current = {
                date: myDate,
                confirmed: item.confirmed,
                recovered: item.recovered,
                deaths: item.deaths
              };

              vm.addDataEntry(vm.current, vm.previous, ignoreChartPoint);

              vm.previous = vm.current;
            }
          }
        })
        .then(() => {
          fetch(vm.currentApi)
            .then(d => d.json())
            .then(data => {
              var item = data.features[0].attributes;
              var myDate = new Date(item.Last_Update);
              vm.current = {
                date: myDate,
                confirmed: item.Confirmed,
                recovered: item.Recovered,
                deaths: item.Deaths
              };

              var delta = vm.addDataEntry(vm.current, vm.previous);
              vm.updateFields(vm.current, delta);

              vm.totalChartOptions.title =
                vm.strings.chart.totalTitle + vm.countryName;
              vm.deltaChartOptions.title =
                vm.strings.chart.newTitle + vm.countryName;
            });
        })
        .catch(() => {
          if (vm.totalDataTable.addRow) {
            vm.tableAddRow(vm.totalDataTable, null);
            vm.tableAddRow(vm.deltaDataTable, null);
            vm.updateFields();
          }
        });
    },

    getIsoCodeFromUrl() {
      var vm = this;
      var stringUrl = window.location.href;
      var url = new URL(stringUrl);
      var code = url.searchParams.get("c") || url.searchParams.get("country");

      if (!code) {
        return null;
      }

      code = code.toLowerCase();

      var countryFound = vm.countries.find(c => c.iso3 === code);
      if (countryFound) {
        return countryFound.iso3;
      }

      countryFound = vm.countries.find(c => c.name.toLowerCase() === code);
      if (countryFound) {
        return countryFound.iso3;
      } else {
        return null;
      }
    },

    getIso3CodeFromIp: async function() {
      try {
        var jsonUrl = {
          ip: "https://ip.nf/me.json",
          iso3:
            "https://raw.githubusercontent.com/sdudnic/covid-19/master/countries/iso3.json"
        };
        var init = {
          headers: {
            Origin: "*"
          }
        };
        var response = await fetch(jsonUrl.ip, init)
          .then(j2 => j2.json())
          .then(iso2 =>
            fetch(jsonUrl.iso3)
              .then(j3 => j3.json())
              .then(iso3 => {
                return iso3[iso2.ip.country_code];
              })
          );
        return response;
      } catch (error) {
        return null;
      }
    },

    countryCodeChanged() {
      var vm = this;
      vm.$emit("countryChanged", vm.countryName);
    }
  },
  filters: {},
  computed: {
    countryName() {
      var iso3 = this.countryCode;
      var countries = this.countries;

      if (!iso3 || !countries) {
        return "";
      }

      var country = countries.find(c => c.iso3 === iso3.toLowerCase());
      if (country && country.name) {
        return country.name;
      } else {
        return "";
      }
    },
    timelineApi() {
      return this.getApiUrl(this.countryCode, this.countryName, true);
    },
    currentApi() {
      return this.getApiUrl(this.countryCode, this.countryName, false);
    },
    readyToLoadData() {
      return (
        this.ready.deltaChart &&
        this.ready.totalChart &&
        this.countryCode &&
        this.daysNumber
      );
    },
    startDateString() {
      var now = new Date();
      var utcNow = new Date(
        Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate())
      );
      var startDate = utcNow.addDays(-this.daysNumber - 1);
      return startDate.toISOString().slice(0, 10);
    },
    lastUpdateTime() {
      if (!this.cases.updateDate) {
        return "";
      }
      return this.cases.updateDate.toLocaleString();
    },
    previousUpdateTime() {
      if (!this.previous.date) {
        return "";
      }
      return this.previous.date.toLocaleString();
    }
  },
  watch: {
    countryCode: ["countryCodeChanged"],
    //daysNumber: "loadData"
    readyToLoadData: { handler: "tryReloadData", deep: true }
  },
  beforeMount() {
    var vm = this;
    Date.prototype.addDays = function(days) {
      var date = new Date(this.valueOf());
      date.setDate(date.getDate() + days);
      return date;
    };
    vm.getCountries();
    vm.deltaChartOptions = JSON.parse(JSON.stringify(vm.totalChartOptions)); // clone
  },
  mounted: function() {
    var vm = this;

    var urlCode = vm.getIsoCodeFromUrl();
    if (urlCode) {
      vm.countryCode = urlCode;
    } else {
      vm.getIso3CodeFromIp().then(iso3 => {
        if (iso3) {
          iso3 = iso3.toLowerCase();
          vm.countryCode = iso3;
        } else {
          // try to throw a countrycode change
          //vm.countryCode = vm.countryCode;
        }
      });
    }
  }
};
</script>
