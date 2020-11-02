<template>
<div id="control-panel">
  <h2>YouBike Map</h2>
  <h3>區域顯示</h3>
  <div class="filter-district">
    <span>
      <input type="checkbox" v-model="allDistrict" id="district-all" />
      <label @click.prevent="selectAllDistrict" for="district-all">全部</label>
    </span>
    <span v-for="key in keysOfDistricts" :key="key">
      <b-button v-b-tooltip.hover :title="key">
        <input type="checkbox" v-model="selectedDistrict" :value="key" :id="key | districtId" />
        <label :id="key" :for="key | districtId" @click="setCenter">{{
          districts[key].name
        }}</label>
      </b-button>
    </span>
  </div>
</div>
</template>

<script>
import {
  reactive,
  computed,
  onMounted,
  onBeforeMount,
} from "@vue/composition-api";
import {
  keys,
  includes
} from "lodash";
import axios from "axios";
import GoogleMapsApiLoader from 'google-maps-api-loader'

const ubikeURL =
  "https://tcgbusfs.blob.core.windows.net/blobyoubike/YouBikeTP.json";

export default {
  setup() {
    const state = reactive({
      ubikeStops: [],
      districts: {
        100: {
          name: "中正區",
          lat: 25.0361013,
          lng: 121.5169923,
        },
        103: {
          name: "大同區",
          lat: 25.0630815,
          lng: 121.5142474,
        },
        104: {
          name: "中山區",
          lat: 25.0685018,
          lng: 121.5280918,
        },
        105: {
          name: "松山區",
          lat: 25.056242,
          lng: 121.5573418,
        },
        106: {
          name: "大安區",
          lat: 25.0274139,
          lng: 121.5372087,
        },
        108: {
          name: "萬華區",
          lat: 25.0275628,
          lng: 121.497341,
        },
        110: {
          name: "信義區",
          lat: 25.0327753,
          lng: 121.5715193,
        },
        111: {
          name: "士林區",
          lat: 25.0866494,
          lng: 121.5198019,
        },
        112: {
          name: "北投區",
          lat: 25.1211862,
          lng: 121.5075251,
        },
        114: {
          name: "內湖區",
          lat: 25.0713177,
          lng: 121.5866103,
        },
        115: {
          name: "南港區",
          lat: 25.0557703,
          lng: 121.6072372,
        },
        116: {
          name: "文山區",
          lat: 24.9982971,
          lng: 121.5544842,
        },
      },
      districtCount: 12,
      allDistrict: true,
      selectedDistrict: [],
      googleMap: {
        map: null,
        center: {
          lat: 25.03737,
          lng: 121.56355,
        },
        zoom: 16,
        markers: [],
        style: [{
            "featureType": "all",
            "elementType": "geometry",
            "stylers": [{
              "color": "#202c3e"
            }]
          },
          {
            "featureType": "all",
            "elementType": "labels.text.fill",
            "stylers": [{
                "gamma": 0.01
              },
              {
                "lightness": 20
              },
              {
                "weight": "1.39"
              },
              {
                "color": "#ffffff"
              }
            ]
          },
          {
            "featureType": "all",
            "elementType": "labels.text.stroke",
            "stylers": [{
                "weight": "0.96"
              },
              {
                "saturation": "9"
              },
              {
                "visibility": "on"
              },
              {
                "color": "#000000"
              }
            ]
          },
          {
            "featureType": "all",
            "elementType": "labels.icon",
            "stylers": [{
              "visibility": "off"
            }]
          },
          {
            "featureType": "landscape",
            "elementType": "geometry",
            "stylers": [{
                "lightness": 30
              },
              {
                "saturation": "9"
              },
              {
                "color": "#29446b"
              }
            ]
          },
          {
            "featureType": "poi",
            "elementType": "geometry",
            "stylers": [{
              "saturation": 20
            }]
          },
          {
            "featureType": "poi.park",
            "elementType": "geometry",
            "stylers": [{
                "lightness": 20
              },
              {
                "saturation": -20
              }
            ]
          },
          {
            "featureType": "road",
            "elementType": "geometry",
            "stylers": [{
                "lightness": 10
              },
              {
                "saturation": -30
              }
            ]
          },
          {
            "featureType": "road",
            "elementType": "geometry.fill",
            "stylers": [{
              "color": "#193a55"
            }]
          },
          {
            "featureType": "road",
            "elementType": "geometry.stroke",
            "stylers": [{
                "saturation": 25
              },
              {
                "lightness": 25
              },
              {
                "weight": "0.01"
              }
            ]
          },
          {
            "featureType": "water",
            "elementType": "all",
            "stylers": [{
              "lightness": -20
            }]
          }
        ]
      },
      google: null,
      map: null,

      keysOfDistricts: computed(() => keys(state.districts)),
      selectedDistrictValues: computed(() =>
        state.selectedDistrict.map((key) => state.districts[key].name)
      ),
      filteredUbikeStops: computed(() => {
        var selectedDistrictValues = state.selectedDistrictValues;
        var filtered = state.ubikeStops.filter((data) =>
          includes(selectedDistrictValues, data.sarea)
        );
        return filtered;
      }),
    });

    const startRetrieveUbikeData = () => {
      sendUbikeDataRequest();
      setInterval(() => sendUbikeDataRequest(), 1000 * 60);
    };
    const sendUbikeDataRequest = () => {
      axios.get(ubikeURL).then((response) => {
        state.ubikeStops = keys(response.data.retVal).map(
          (key) => response.data.retVal[key]
        );

        establishMarkers();
      });
    };

    const initGoogleMap = () => {
      state.googleMap.map = new state.google.maps.Map(
        document.getElementById("map"), {
          zoom: state.googleMap.zoom,
          center: state.googleMap.center,
          styles: state.googleMap.style,
          disableDefaultUI: true,
        }
      );
    };

    const establishMarker = (data) => {
      let marker = new state.google.maps.Marker({
        position: {
          lat: Number(data.lat),
          lng: Number(data.lng),
        },
        map: state.googleMap.map,
        title: data.sna,
      });

      (function (data, marker) {
        var infowindow = new state.google.maps.InfoWindow({
          content: `
            站場名稱：${data.sna}<br/>
            區域地址：${data.sarea} - ${data.ar}<br/>
            總數量：${data.tot}<br/>
    <strong>目前車輛數量：${data.sbi}</strong><br/>
            更新時間：${timeFormat(data.mday)}`,
        });

        state.google.maps.event.addListener(marker, "mouseover", () =>
          infowindow.open(state.googleMap.map, marker)
        );
        state.google.maps.event.addListener(marker, "mouseout", () =>
          infowindow.close()
        );
      }.bind(this)(data, marker));

      return marker;
    };

    const establishMarkers = () => {
      if (state.googleMap.markers.length !== 0) {
        for (let marker of state.googleMap.markers) {
          marker.setMap(null);
        }
        state.googleMap.markers = [];
      }
      loopThroughUbikeStops((data) =>
        state.googleMap.markers.push(establishMarker(data))
      );
    };

    const selectAllDistrict = () => {
      state.selectedDistrict = [];
      if (state.allDistrict) {
        state.allDistrict = false;
      } else {
        state.allDistrict = true;
        state.selectedDistrict = state.keysOfDistricts;
      }
    };

    const setCenter = (event) => {
      let key = event.target.id;
      if (includes(state.selectedDistrict, key)) {
        let data = state.districts[key];
        state.googleMap.map.setCenter(
          new state.google.maps.LatLng(data.lat, data.lng)
        );
        state.googleMap.map.setZoom(15);
      }
    };

    const loopThroughUbikeStops = (callback) => {
      for (let data of state.filteredUbikeStops) {
        callback(data);
      }
    };

    const timeFormat = (t) => {
      var date = [],
        time = [];
      date.push(t.substr(0, 4));
      date.push(t.substr(4, 2));
      date.push(t.substr(6, 2));
      time.push(t.substr(8, 2));
      time.push(t.substr(10, 2));
      time.push(t.substr(12, 2));
      return date.join("/") + " " + time.join(":");
    };

    onBeforeMount(() => {
      startRetrieveUbikeData();
      state.selectedDistrict = state.keysOfDistricts;
    });

    onMounted(async () => {
      const googleMapApi = await GoogleMapsApiLoader({
        apiKey: 'AIzaSyDaBCjx6xCo59SI6O55rQ4e6q2FG3l9PD4'
      })
      state.google = googleMapApi
      initGoogleMap();
    });

    return {
      state,
      districts: state.districts,
      allDistrict: state.allDistrict,
      selectAllDistrict,
      selectedDistrict: state.selectedDistrict,
      keysOfDistricts: state.keysOfDistricts,
      setCenter,
    };
  },
  filters: {
    districtId: function (id) {
      return `district-${id}`;
    },
  },
};
</script>
