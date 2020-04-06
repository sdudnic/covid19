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
            class="par-5"
          ></v-autocomplete>
          <v-text-field
            :label="strings.chart.daysBack"
            :value="daysNumber"
            dense
            style="max-width: 45px"
            type="number"
            hide-details
            class="pal-5"
          ></v-text-field>
        </v-row>
        <v-row class="ma-0 pa-0" align="center" justify="center">
          <small>
            {{strings.lastData}}
            <span
              id="lastApiUpdate"
              class="font-weight-bold"
            >{{history.updateDate | isoDateTime}}</span>
            (+{{strings.today}})
          </small>
        </v-row>
        <v-row align="center" justify="center">
          <v-card
            v-for="(color, i) in colors"
            :key="i"
            outlined
            dark
            :color="color"
            :style="{opacity: (history.new[i]===0? .5 : 1)}"
            class="ma-5 text-center"
          >
            <div class="overline">{{strings.chart[i]}}</div>
            <div class="pa-1">+ {{history.new[i]}} ({{history.total[i]}})</div>
          </v-card>
        </v-row>
      </v-col>
    </v-row>

    <v-row class="text-center">
      <v-col cols="12">
        <GChart
          type="LineChart"
          :settings="{ 'packages':['corechart'], language: language}"
          :data="newData"
          :options="newChartOptions"
        />
      </v-col>
      <v-col cols="12">
        <GChart
          type="LineChart"
          :settings="{ 'packages':['corechart'], language: language}"
          :data="totalData"
          :options="totalChartOptions"
          @ready="onChartReady"
        />
      </v-col>
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
      daysNumber: 5,
      countryCode: "mda",
      history: {
        total: {
          confirmed: 0,
          recovered: 0,
          deaths: 0
        },
        new: {
          confirmed: 0,
          recovered: 0,
          deaths: 0
        },
        updateDate: new Date()
      },
      totalChartOptions: {
        title: "Total cases",
        curveType: "function",
        legend: {
          position: "none"
        },
        // set annotation for -- No Data Copy
        annotations: {
          // remove annotation stem and push to middle of chart
          stem: {
            color: "transparent",
            length: 120
          },
          textStyle: {
            color: "red",
            fontSize: 22
          }
        },
        explorer: {},
        hAxis: {
          title: '',
          viewWindowMode: 'maximized'
          //format: 'short'
        },
        vAxis: {
          viewWindow: { min: 0 }
        },
        colors: Object.values(this.colors),
        pointSize: 5,
        pointShape: "diamond"
      },
      newChartOptions: {},
      totalData: {},
      newData: {},
      countries: [],
      apiUrl: {
        timeline: "https://covidapi.info/api/v1/country/***",
        current:
          "https://services1.arcgis.com/0MSEUqKaxRlEPj5g/arcgis/rest/services/Coronavirus_2019_nCoV_Cases/FeatureServer/2/query?where=UPPER(Country_Region)%20like%20%27%25***%25%27&outFields=Last_Update,Confirmed,Deaths,Recovered,Country_Region&returnGeometry=false&outSR=4326&f=json"
      },
      startDateString: ""
    };
  },
  methods: {
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
    updateFields(current) {
      this.history.updateDate = current.date;
      this.history.total.confirmed = current.confirmed;
      this.history.total.recovered = current.recovered;
      this.history.total.deaths = current.deaths;
    },
    onChartReady(chart, google) {
      var vm = this;
      vm.totalData = new google.visualization.DataTable();
      vm.newData = new google.visualization.DataTable();

      vm.totalData.addColumn("date", "Date");
      vm.totalData.addColumn("number", vm.strings.chart.confirmed);
      vm.totalData.addColumn("number", vm.strings.chart.recovered);
      vm.totalData.addColumn("number", vm.strings.chart.deaths);
      vm.totalData.addColumn({
        role: "annotation",
        type: "string"
      });

      vm.newData.addColumn("date", "Date");
      vm.newData.addColumn("number", vm.strings.chart.confirmed);
      vm.newData.addColumn("number", vm.strings.chart.recovered);
      vm.newData.addColumn("number", vm.strings.chart.deaths);
      vm.newData.addColumn({
        role: "annotation",
        type: "string"
      });

      var formatter = new google.visualization.DateFormat({
        pattern: "dd/MM/yy hh:mm"
      });
      formatter.format(vm.totalData, 0);
      formatter.format(vm.newData, 0);

      vm.newChartOptions = JSON.parse(JSON.stringify(vm.totalChartOptions)); // clone
      vm.loadData(vm.countryCode, vm.countryName);
    },
    cleanTable(dataTable) {
      if (dataTable.getNumberOfRows) {
        var rowsNr = dataTable.getNumberOfRows();
        if (rowsNr) dataTable.removeRows(0, rowsNr);
      }
    },
    addNewDataRow(current, previous, ignored) {
      var vm = this;
      vm.history.updateDate = current.date;

      vm.history.total.confirmed = current.confirmed;
      vm.history.new.confirmed = current.confirmed - previous.confirmed;
      previous.confirmed = current.confirmed;

      vm.history.total.recovered = current.recovered;
      vm.history.new.recovered = current.recovered - previous.recovered;
      previous.recovered = current.recovered;

      vm.history.total.deaths = current.deaths;
      vm.history.new.deaths = current.deaths - previous.deaths;
      previous.deaths = current.deaths;

      if (!ignored) {
        vm.totalData.addRow([
          current.date,
          current.confirmed,
          current.recovered,
          current.deaths,
          null
        ]);
        vm.newData.addRow([
          current.date,
          vm.history.new.confirmed,
          vm.history.new.recovered,
          vm.history.new.deaths,
          null
        ]);
      }
    },
    loadData(countryCode, countryName) {
      var vm = this;
      countryCode = countryCode || vm.countryCode;
      countryName = countryName || vm.countryName;

      var now = new Date();
      var utcNow = new Date(
        Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate())
      );
      var startDate = utcNow.addDays(-vm.daysNumber - 1);
      vm.startDateString = startDate.toISOString().slice(0, 10);

      if (vm.totalData.addRow) {
        vm.totalData.addRow([null, 0, 0, 0, null]);
        vm.newData.addRow([null, 0, 0, 0, null]);

        vm.cleanTable(vm.totalData);
        vm.cleanTable(vm.newData);
      }

      var timelineApi = vm.getApiUrl(countryCode, countryName, true);
      var currentApi = vm.getApiUrl(countryCode, countryName, false);

      var previous = {
        date: {},
        confirmed: 0,
        recovered: 0,
        deaths: 0
      };

      var current = {
        date: {},
        confirmed: 0,
        recovered: 0,
        deaths: 0
      };

      fetch(timelineApi)
        .then(a => a.json())
        .then(apiAnswer => {
          var results = Object.entries(apiAnswer.result).filter(
            vm.filterFromStartDate
          );

          if (results.length === 0) {
            // no data
            vm.totalData.addRow([null, 0, 0, 0, "NO DATA"]);
            vm.newData.addRow([null, 0, 0, 0, "NO DATA"]);
            vm.updateFields();
          } else {
            // we have data
            for (let i = 0; i < results.length; i++) {
              const item = results[i][1];
              var myDate = new Date(results[i][0]);
              current = {
                date: myDate,
                confirmed: item.confirmed,
                recovered: item.recovered,
                deaths: item.deaths
              };
              var ignoreAdding = i == 0;
              vm.addNewDataRow(current, previous, ignoreAdding);
            }
          }
        })
        .then(() => {
          fetch(currentApi)
            .then(d => d.json())
            .then(data => {
              var item = data.features[0].attributes;
              var myDate = new Date(item.Last_Update);
              current = {
                date: myDate,
                confirmed: item.Confirmed,
                recovered: item.Recovered,
                deaths: item.Deaths
              };

              vm.addNewDataRow(current, previous);
              vm.updateFields(current);

              vm.totalChartOptions.title =
                vm.strings.chart.totalTitle + countryName;
              vm.newChartOptions.title =
                vm.strings.chart.newTitle + countryName;
            });
        })
        .catch(() => {
          if (vm.totalData.addRow) {
            vm.totalData.addRow([null, 0, 0, 0, "NO DATA"]);
            vm.newData.addRow([null, 0, 0, 0, "NO DATA"]);
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
    changeTitle: function(countryName) {
      const myRegex = /(^.*)(COVID-19 evolution graphs,)/;
      const replacement = countryName + " $2";
      document.title = document.title.replace(myRegex, replacement);
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
    countryCodeChanged(newCode) {
      var name = this.countryName;
      this.loadData(newCode, name);
      this.changeTitle(name);
    }
  },
  filters: {
    isoDateTime: function(value) {
      var dateText = "-";
      if (value && value instanceof Date) {
        dateText =
          value.toISOString().slice(0, 10) + " " + value.toLocaleTimeString();
      }
      return dateText;
    }
  },
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
    }
  },
  watch: {
    countryCode: "countryCodeChanged",
    daysNumber: "loadData"
  },
  beforeMount() {
    Date.prototype.addDays = function(days) {
      var date = new Date(this.valueOf());
      date.setDate(date.getDate() + days);
      return date;
    };
    this.getCountries();
  },
  mounted: function() {
    var vm = this;

    if (vm.totalData && vm.totalData.setColumnLabel) {
      vm.totalData.setColumnLabel(1, vm.strings.chart.confirmed);
      vm.totalData.setColumnLabel(2, vm.strings.chart.recovered);
      vm.totalData.setColumnLabel(3, vm.strings.chart.deaths);
      vm.newData.setColumnLabel(1, vm.strings.chart.confirmed);
      vm.newData.setColumnLabel(2, vm.strings.chart.recovered);
      vm.newData.setColumnLabel(3, vm.strings.chart.deaths);
    }

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
