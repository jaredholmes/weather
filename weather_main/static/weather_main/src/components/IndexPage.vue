<template lang="html">
  <div v-if="loaded">
    <navbar-item></navbar-item>

    <div
      v-if="!returnVisit"
      id="alert-location"
      class="alert alert-primary bc-light"
      role="alert"
    >
    <span>For a better experience, allow Atmocast to get weather information for your location.</span>
    <span class="alert-buttons-container">
      <button
        @click="$getPosition(true, true)"
        id="btn-location-get"
        class="btn fw-semi"
        type="button"
        name="button"
      >Get my location</button>
      <button
        @click="$focusLocationSearch();
        $hideAlert('alert-location');"
        id="btn-location-search"
        class="btn fw-semi"
      >Search location</button>
      <img
        @click="$hideAlert('alert-location')"
        :src="$store.state.iconLocationPrefix + 'close-light.png'" class="alert-close" alt="Close alert"
      >
    </span>
    </div>

    <div
      id="main-section"
      @click="
        $hideCollapse();
        $hideNavbarSearchButton();
        $hideSearchResults();
      "
    >
      <div id="alert-main" class="alert alert-primary bc-light" role="alert">
        <span v-html="$store.state.alertMessage" class="alert-message"></span>
        <img
          @click="$hideAlert();"
          :src="$store.state.iconLocationPrefix + 'close-light.png'"
          class="alert-close" alt="Close alert"
        >
      </div>

      <div v-if="$store.state.weather.hourly">
        <display-item></display-item>
        <details-pane-item :modeHourly="true"></details-pane-item>
        <details-pane-item :modeHourly="false"></details-pane-item>
      </div>

      <fav-location-list></fav-location-list>

      <footer-item></footer-item>
    </div>
  </div>
</template>

<script>
import localforage from 'localforage';
import NavbarItem from './NavbarItem.vue';
import DisplayItem from './DisplayItem.vue';
import DetailsPaneItem from './DetailsPaneItem.vue';
import FavLocationList from './FavLocationList.vue';
import FooterItem from './FooterItem.vue';

export default {
  name: 'IndexPage',

  components: {
    NavbarItem,
    DisplayItem,
    DetailsPaneItem,
    FavLocationList,
    FooterItem
  },

  data() {
    return {
      loaded: false,
      returnVisit: true,
    }
  },

  computed: {
    metric() {
      return this.$store.state.metric;
    },
    lat() {
      return this.$store.state.coords.lat;
    },
    favLocationExists() {
      return this.$store.state.favLocationExists;
    },
    currentCity() {
      return this.$store.state.currentCity;
    },
    weather() {
      return this.$store.state.weather;
    },
  },
  watch: {
    // Weather data will change and loading spinner can disappear
    weather() {
      this.loaded = true;
    },
    currentCity() {
      document.title = 'Weather in ' + this.currentCity + ' – Atmocast Weather';
    },
    // Convert between metric and imperial
    metric() {
      if (this.metric) {
        this.$convertAll(this.$store.state.weather, this.$fToC, this.$mToKm, this.$inToCm);
      } else {
        // Only executed if data is loaded. Prevents errors
        if (this.loaded) {
          this.$convertAll(this.$store.state.weather, this.$cToF, this.$kmToM, this.$cmToIn);
        }
      }
    },
    // Lat changes when the location is set, which makes it a good time to fetch the main data
    lat() {
      this.$getMainData();
    }
  },
  beforeMount() {
    this.$checkMetric();
    if (this.$store.state.proUser) {
      this.$store.commit({
        type: 'setFavLimit',
        limit: 20,
      });
    }
    this.$checkFavLocation();
  },
  created() {
    // Navabar brand will be hidden if the app is being used
    this.$store.commit({
      type: 'setShowNavBrand',
      bool: !this.$store.state.androidApp,
    });
  },
  beforeUpdate() {
    // Used on the first visit to request the user's location
    if (!this.favLocationExists) {
      localforage.getItem(
        'returnVisit',
        (err, value) => {
          if (!value) {
            this.returnVisit = false;
            localforage.setItem('returnVisit', true);
            this.$showLocationAlert();
          }
        }
      );
    }
  },
  updated() {
    // Open links in browser instead of WebView in app
    // if (this.$store.state.androidApp) {
    //   const links = document.getElementsByTagName('a');
    //
    //   for (var i = 0; i < links.length; i++) {
    //     links[i].addEventListener('click', (event) => {
    //       event.preventDefault();
    //       if (event.target.href) {
    //         window.open(event.target.href, "_system");
    //       }
    //     });
    //   }
    // }
    // Reload data every 60 minutes
    window.setInterval(
      () => {
        if (navigator.onLine) {
          if (this.$store.state.weather) {
            this.$setLocation();
            this.$getMainData();
          }
        }
      },
      45 * 60000
    );
  },
};
</script>

<style scoped lang="sass">
  @import "../stylesheets/styles"

  #alert-main
    position: fixed
    top: 0
    left: 0
    height: auto
    width: 100%
    z-index: 2

  .alert
    margin-bottom: 0
    max-height: 0
    padding: 0
    border: none
    text-align: center
    transition: max-height 100ms ease-in, padding 80ms ease-in
    align-items: center

    .alert-message, .alert-close
      display: none

    .alert-message
      max-width: 90%

    .alert-close
      max-width: 1em
      max-height: 1em
      position: absolute
      right: $s-s-6
      bottom: 50%
      top: 50%
      margin: auto

    .alert-close:hover
      cursor: pointer

  .alert.shown
    max-height: 20em
    height: auto
    padding: $s-s-5 $s-s-4

    .alert-message, .alert-close
      display: inline

  #alert-location

    span
      margin-bottom: $s-s-5

      @include media-large
        margin: 0 $s-s-5

    .btn
      margin: $s-s-5 $s-s-6 4px $s-s-6
      color: $light

      @include media-large
        margin-top: 0

    .btn:hover
      color: $text-primary

    #btn-location-get:hover
      background-color: $cta-red-hover

    #btn-location-get
      background-color: $cta-red

    #btn-location-search:hover
      background-color: $cta-blue-hover

    #btn-location-search
      background-color: $cta-blue

  #alert-location.shown

    .btn
      @include media-tiny
        display: block
        margin-left: auto
        margin-right: auto

</style>
