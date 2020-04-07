<template>
  <v-app>
    <v-app-bar app color="amber">
      <div class="align-center">
        <v-img
          alt="Vuetify Logo"
          class="shrink mr-2 hidden-md-and-up"
          contain
          :src="require('./assets/logo.png')"
          transition="scale-transition"
          height="5em"
        />

        <v-img
          alt="Vuetify Name"
          class="shrink mt-1 hidden-sm-and-down"
          contain
          min-width="100"
          src="./assets/logo.png"
          width="100"
        />
      </div>

      <v-toolbar-title>COVID-19</v-toolbar-title>
      <v-spacer></v-spacer>
      {{countryName}}
    </v-app-bar>

    <v-content>
      <Graphs
        :strings="strings"
        :language="language"
        :colors="colors"
        @countryChanged="countryChanged($event)"
      />
    </v-content>
    <v-footer absolute app>
      <v-row align="center" justify="space-between">
        <v-card color="transparent" outlined class="mx-2 my-0 px-2 py-0">
          <small>
            {{strings.petition.moldovan}} -
            <strong>
              <a :href="strings.petition.link" target="blank">{{strings.petition.title}}</a>
            </strong>
          </small>
        </v-card>
        <v-card color="transparent" outlined class="mx-2 my-0 px-2 py-0">
          <small>
            {{ new Date().getFullYear() }}
            <a
              href="https://github.com/sdudnic/covid-19"
              target="blank"
            >covid-19</a> project by
            <a href="https://github.com/sdudnic" target="blank">sdudnic</a>
            & co
          </small>
        </v-card>
      </v-row>
    </v-footer>
  </v-app>
</template>

<script>
import Graphs from "./components/Graphs";

export default {
  name: "App",
  components: {
    Graphs
  },
  data: () => ({
    strings: {
      chart: {},
      petition: {}
    },
    language: "en",
    colors: {
      confirmed: "orange",
      recovered: "green",
      deaths: "red"
    },
    countryName: ""
  }),
  beforeMount() {
    // get translations
    this.getStrings();
  },
  methods: {
    getStrings() {
      var vm = this;
      var browserLanguage = navigator.language || navigator.userLanguage;
      vm.language = browserLanguage.slice(0, 2);
      return fetch("./data/strings.json")
        .then(s => s.json())
        .then(st => (vm.strings = st[vm.language]));
    },
    changeTitle: function(countryName) {
      const myRegex = /(^.*)(COVID-19 evolution graphs,)/;
      const replacement = countryName + " $2";
      document.title = document.title.replace(myRegex, replacement);
    },
    countryChanged(countryName) {
      this.countryName = countryName;
      this.changeTitle(countryName);
    }
  }
};
</script>
