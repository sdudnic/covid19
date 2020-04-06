<template>
  <v-app>
    <v-app-bar app color="amber">
      <div class="d-flex align-center">
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

      <!-- <v-btn href="https://github.com/sdudnic/covid19" target="_blank" text>
        <span class="mr-2">Petition</span>
        <v-icon>mdi-open-in-new</v-icon>
      </v-btn>-->
    </v-app-bar>

    <v-content>
      <Graphs :strings="strings" :language="language" :colors="colors" />
    </v-content>
    <v-footer absolute app>
      <v-card-text class="text-right">
        {{ new Date().getFullYear() }}
        <a
          href="https://github.com/sdudnic/covid-19"
          target="blank"
        >covid-19</a> project by
        <a href="https://github.com/sdudnic" target="blank">sdudnic</a> & co
        <v-spacer></v-spacer>
      </v-card-text>
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
    }
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
    }
  }
};
</script>
