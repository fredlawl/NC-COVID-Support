<template>
  <b-container class="bv-example-row px-0" fluid>
    <div class="map">
      <l-map
        ref="covidMap"
        v-if="showMap"
        :zoom="zoom"
        :center="center"
        :options="mapOptions"
        style="height: 100%; width: 100%;"
        @update:center="centerUpdated"
        @update:zoom="zoomUpdated"
        @update:bounds="boundsUpdated"
      >
        <l-control position="topright">
          <div class="mapkey" :class="{ 'show-key': showKey }">
            <div class="title-block">
              <h6 class="title">{{ $t('label.mapkey') }}</h6>
              <i @click="showKey = !showKey" class="fas fa-info-circle" />
            </div>
            <div class="keys" :class="{ 'show-key': showKey }" v-for="item in mapKey" v-bind:key="item.title">
              <icon-list-item :leaflet-icon="item.icon" :title="item.title" link="" />
            </div>
          </div>
        </l-control>
        <l-tile-layer :url="mapUrl" :attribution="attribution" />
        <l-circle
          name="Accuracy"
          :lat-lng="userLocationData"
          v-if="userLocationData"
          :radius="accuracyRadius()"
          :weight="1"
          :class-name="'locAccuracy'"
        ></l-circle>
        <l-circle-marker
          name="Your Location"
          :lat-lng="userLocationData"
          v-if="userLocationData"
          :radius="circleRadius()"
          :weight="1"
          :class-name="'locMarker'"
        ></l-circle-marker>
        <v-marker-cluster ref="marks" :options="clusterOptions">
          <!-- @clusterclick="click()" @ready="ready" -->
          <l-marker
            :lat-lng="latLng(item.marker.gsx$lat.$t, item.marker.gsx$lon.$t)"
            :icon="selectedIcon(location.currentBusiness !== null && item.marker.id.$t === location.currentBusiness.marker.id.$t, item)"
            v-for="(item, index) in filteredMarkers"
            v-bind:key="index"
            @click="$emit('location-selected', { locValue: index, locId: item.marker.id.$t, isSetByMap: true })"
            :options="{ onScreen: true, marker: item }"
          >
          </l-marker>
        </v-marker-cluster>
        <l-control position="bottomright" class="user-location-button">
          <a
            href="#"
            @click="getUserLocation"
            class="user-location-link"
            ref="useLocation"
            :title="$t('label.useyourlocation')"
            :aria-label="$t('label.useyourlocation')"
          >
            <!--<h5>Your location</h5>-->
            <i class="fas fa-location-arrow"></i>
          </a>
        </l-control>
        <b-alert variant="warning" class="location-alert" :show="showError" dismissible @dismissed="resetError" fade>
          {{ errorMessage }}
        </b-alert>
      </l-map>
    </div>
  </b-container>
</template>

<script>
import { BAlert } from 'bootstrap-vue'
import { LMap, LTileLayer, LMarker, LControl, LCircle, LCircleMarker } from 'vue2-leaflet'
import { latLng, Icon, ExtraMarkers, DomUtil } from 'leaflet'
import Vue2LeafletMarkerCluster from 'vue2-leaflet-markercluster'
import IconListItem from './IconListItem.vue'
import { businessIcon } from '../utilities'

const covidLabels = {}

const DIRECTION = {
  NORTH: 0,
  EAST: 1,
  SOUTH: 2,
  WEST: 3
}

const createMarker = (marker, icon) => {
  const scale = DomUtil.getScale(icon).boundingClientRect
  return {
    ...marker,
    icon: icon,
    dimensions: {
      x: scale.width,
      y: scale.height
    },
    position: {
      x: scale.x,
      y: scale.y
    }
  }
}

const createMarkerLabel = (marker, direction) => {
  if (covidLabels[marker.uuid]) {
    // update this to get new marker positions
    covidLabels[marker.uuid].marker = marker
    return covidLabels[marker.uuid]
  }

  const text = marker.gsx$providername.$t
  const div = document.createElement('div')
  const span = document.createElement('span')
  span.innerText = text
  div.id = `covid-label-${marker.uuid}`
  div.classList.add('covid-marker-label')
  div.append(span)
  document.body.appendChild(div)

  const label = {
    element: div,
    marker: marker,
    hidden: false,
    direction: direction,
    dimensions: {
      x: div.offsetWidth,
      y: div.offsetHeight
    },
    position() {
      return {
        x:
          this.direction === DIRECTION.WEST
            ? this.marker.position.x - this.dimensions.x
            : this.marker.position.x + this.marker.dimensions.x,
        y: this.marker.position.y
      }
    },
    clone() {
      return {
        element: this.element,
        marker: this.marker,
        hidden: this.hidden,
        direction: this.direction,
        dimensions: this.dimensions,
        position: this.position,
        hide: this.hide,
        show: this.show,
        draw: this.draw
      }
    },
    hide() {
      this.hidden = true
      this.element.classList.remove('open')
    },
    show() {
      this.hidden = false
      this.element.classList.add('open')
    },
    draw() {
      if (this.hidden) {
        this.element.classList.remove('open')
        return
      }

      const pos = this.position()
      this.element.style.transform = `translate3d(${pos.x}px, ${pos.y}px, 0px)`

      if (this.direction === DIRECTION.EAST) {
        this.element.style.textAlign = `left`
      } else {
        this.element.style.textAlign = `right`
      }

      this.element.classList.add('open')
    }
  }

  covidLabels[marker.uuid] = label
  return label
}

const createMarkerLabelHitbox = (markerLabel, direction) => {
  const clone = markerLabel.clone()
  clone.direction = direction

  return {
    position: {
      x: clone.direction.x === DIRECTION.WEST && !clone.hidden ? clone.position().x : clone.marker.position.x,
      y: clone.marker.position.y
    },
    dimensions: {
      x: !clone.hidden ? clone.dimensions.x + clone.marker.dimensions.x : clone.marker.dimensions.x,
      y: !clone.hidden ? clone.dimensions.y : clone.marker.dimensions.y
    },
    collidesWith(b) {
      return (
        this.position.x + this.dimensions.x > b.position.x &&
        this.position.y + this.dimensions.y > b.position.y &&
        this.position.x < b.position.x + b.dimensions.x &&
        this.position.y < b.position.y + b.dimensions.y
      )
    },
    equals(b) {
      return (
        this.position.x === b.position.x &&
        this.position.y === b.position.y &&
        this.dimensions.x === b.dimensions.x &&
        this.dimensions.y === b.dimensions.y
      )
    }
  }
}

delete Icon.Default.prototype._getIconUrl
Icon.Default.mergeOptions({
  iconRetinaUrl: require('leaflet/dist/images/marker-icon-2x.png'),
  iconUrl: require('leaflet/dist/images/marker-icon.png'),
  shadowUrl: require('leaflet/dist/images/marker-shadow.png')
})

export default {
  name: 'ResourceMap',
  components: {
    BAlert,
    LMap,
    LTileLayer,
    LMarker,
    LControl,
    LCircle,
    LCircleMarker,
    'v-marker-cluster': Vue2LeafletMarkerCluster,
    IconListItem
  },
  props: {
    filteredMarkers: Array,
    location: { locValue: Number, currentBusiness: Object, isSetByMap: Boolean },
    mapUrl: String,
    attribution: String,
    centroid: { lat: Number, lng: Number }
  },
  data() {
    return {
      center: latLng(this.centroid.lat, this.centroid.lng),
      zoom: this.centroid.zoom,
      showParagraph: true,
      showError: false,
      errorMessage: '',
      userLocationData: false,
      mapOptions: {
        zoomSnap: 0.5,
        setView: true,
        zoomControl: true
      },
      showMap: true,
      locationData: location,
      accuracy: 0,
      circle: {
        border: 'white',
        fill: '#f00'
      },
      clusterOptions: { spiderfyOnMaxZoom: true, maxClusterRadius: 40, disableClusteringAtZoom: 16 },
      showKey: false,
      bounds: null,
      covidLabels: {}
    }
  },
  mounted() {
    this.editZoomControl()
    this.$nextTick(() => {
      this.bounds = this.$refs.covidMap.mapObject.getBounds()

      this.$refs.covidMap.mapObject.on('movestart', () => {
        this.drawLabels()
      })

      this.$refs.covidMap.mapObject.on('move', () => {
        this.drawLabels()
      })

      this.$refs.covidMap.mapObject.on('zoom', () => {
        this.drawLabels()
      })

      this.$refs.covidMap.mapObject.on('zoomend', () => {
        this.drawLabels()
      })
    })
  },
  computed: {
    mapKey() {
      return [
        {
          title: this.$t('label.open'),
          icon: ExtraMarkers.icon({
            className: 'markeropen',
            icon: 'na',
            prefix: 'fa',
            svg: true
          })
        },
        {
          title: this.$t('label.closedonday'),
          icon: ExtraMarkers.icon({
            className: 'markerclosed',
            icon: 'na',
            prefix: 'fa',
            svg: true
          })
        },
        {
          title: this.$t('label.selected'),
          icon: ExtraMarkers.icon({
            className: 'markerselected',
            icon: 'na',
            prefix: 'fa',
            svg: true
          })
        }
      ]
    }
  },
  methods: {
    centerUpdated(center) {
      this.center = center
      this.$emit('center', center)
    },
    boundsUpdated(bounds) {
      this.bounds = bounds
      this.$emit('bounds', bounds)
    },
    resetError() {
      this.showError = false
      this.errorMessage = ''
    },
    userIcon() {
      const icon = ExtraMarkers.icon({
        markerColor: 'usermarker',
        icon: 'fas fa-home',
        prefix: 'fa',
        svg: true
      })
      return icon
    },
    getUserLocation() {
      var map = this.$refs.covidMap.mapObject
      map.locate({ setView: true, enableHighAccuracy: true, watch: true, maximumAge: 60000 })
      map.on('locationfound', (locationEvent) => {
        if (locationEvent.latitude && locationEvent.longitude) {
          this.userLocationData = latLng(locationEvent.latitude, locationEvent.longitude)
          // this.centerUpdated(this.userLocationData)
          this.accuracy = locationEvent.accuracy
          this.$refs.useLocation.classList.add('active')
        }
      })
      map.on('locationerror', (err) => {
        if (err.message && err.code != err.TIMEOUT) {
          this.showError = true
          this.errorMessage = err.message
          this.errorMessage += ' Please check your browser settings and ensure you have given our site permission to view your location.'
          this.$refs.useLocation.classList.add('disabled')
        }
      })
    },
    editZoomControl() {
      const zoomControl = this.$el.querySelector('.leaflet-top.leaflet-left')
      zoomControl.className = 'leaflet-bottom leaflet-right'
    },
    circleRadius() {
      var radius = 8
      if (radius <= 5) {
        radius = 5
      }
      return radius
    },
    accuracyRadius() {
      var radius = this.accuracy
      return radius
    },
    latLng,
    selectedIcon(selected, item) {
      const isOpen = item.oc
      let markerColor = isOpen ? 'markeropen' : 'markerclosed'
      const iconClasses = businessIcon(item.marker)
      if (selected) {
        markerColor = 'markerselected'
      }
      var markerIcon = ExtraMarkers.icon({
        className: markerColor,
        icon: iconClasses,
        prefix: 'fa',
        svg: true
        // ,
        // name: item.marker.gsx$providername.$t,
        // nameClasses: 'markerName'
      })

      return markerIcon
    },
    zoomUpdated(val) {
      const markers = this.markersInView()
      this.zoom = val

      this.$emit('markersInView', markers)
    },
    markersInView() {
      return this.filteredMarkers.filter((m) => {
        return this.bounds.contains(latLng(m.marker.gsx$lat.$t, m.marker.gsx$lon.$t))
      })
    },
    drawLabels() {
      const availableMarkerLabels = []
      for (const layer of Object.values(this.$refs.covidMap.mapObject._layers)) {
        if (!layer?.options?.onScreen) {
          continue
        }

        const marker = createMarker(layer.options.marker.marker, layer._icon)
        const label = createMarkerLabel(marker, DIRECTION.WEST)
        label.hidden = false
        availableMarkerLabels.push(label)
      }

      for (const a of availableMarkerLabels) {
        a.direction = DIRECTION.WEST

        for (const b of availableMarkerLabels) {
          const aEast = createMarkerLabelHitbox(a, DIRECTION.EAST)
          const bEast = createMarkerLabelHitbox(b, DIRECTION.EAST)
          const aWest = createMarkerLabelHitbox(a, DIRECTION.WEST)
          const bWest = createMarkerLabelHitbox(b, DIRECTION.WEST)

          if (aWest.equals(bWest) || aEast.equals(bEast)) {
            continue
          }

          if (aWest.collidesWith(bWest)) {
            a.direction = DIRECTION.EAST
          }

          if (a.direction === DIRECTION.EAST && aEast.collidesWith(bEast)) {
            a.hidden = true
          }
        }

        a.draw()
      }
    }
    // eslint-disable-next-line no-console
    // click: (e) => console.log('clusterclick', e),
    // eslint-disable-next-line no-console
    // ready: (e) => console.log('ready', e)
  },
  watch: {
    // https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png
    location(locationVal) {
      if (locationVal.isSetByMap) {
        return
      }
      // var item = this.filteredMarkers[locationVal.locValue]
      if (locationVal.currentBusiness !== null && this.$refs.covidMap.mapObject.getZoom() < 17) {
        this.$refs.covidMap.mapObject.setView(
          latLng(locationVal.currentBusiness.marker.gsx$lat.$t, locationVal.currentBusiness.marker.gsx$lon.$t),
          17,
          { duration: 1 }
        )
      }
    }
  }
}
</script>

<style lang="scss">
.covid-marker-label {
  border: 1px solid #ff0000;
  max-width: 132px;
  font-size: 12px;
  text-shadow: 1px 1px 1px #000000;
  color: #ffffff;
  z-index: 999999;
  visibility: hidden;
  position: absolute;
  backface-visibility: hidden;
  will-change: transform;
  box-sizing: border-box;
  min-height: 44px;
  display: flex;
  align-items: center;

  &.open {
    visibility: visible;
  }
}

.map {
  width: auto;
  height: 100%;
  top: 0;
  padding: 0;
  /* margin-left: 8px;
    margin-right: 8px; */
}

.leaflet-tooltip {
  visibility: hidden;

  &.open {
    visibility: visible;
  }
}

.locAccuracy {
  color: $map-accuracy-outline;
  fill: $map-accuracy;
  fill-opacity: 0.15;
  cursor: grab !important;
}

.locMarker {
  color: $map-location-outline;
  fill: $map-location;
  fill-opacity: 1;
  cursor: grab !important;
}

.alert-warning {
  @media (prefers-color-scheme: dark) {
    background-color: theme-color-level('warning', 2) !important;
    color: theme-color-level('warning', 8) !important;
    border-color: theme-color-level('warning', 2) !important;
  }
}

.bv-example-row {
  height: calc(100% - 124px);
}

@include media-breakpoint-up(sm) {
  .bv-example-row {
    height: calc(100% - 116px);
  }
}

.markerselected svg path {
  fill: $marker-selected;
}

div.markeropen svg path {
  fill: $marker-open;
}

.markerclosed svg path {
  fill: $marker-closed;
}

.usermarker {
  background-color: $user-marker;
}

.noselection.bv-example-row {
  height: 100%;
}

.mapkey {
  padding: 16px;

  &.show-key {
    background-color: $map-key-bg !important;
    color: $map-key;
    box-shadow: 0 0 20px rgba(0, 0, 0, 0.3);

    @media (prefers-color-scheme: dark) {
      background-color: $map-key-bg-dark !important;
      color: $map-key-dark;
    }
  }

  i {
    font-size: 2rem;
    opacity: 0.4;
    color: #000;
    cursor: pointer;
    vertical-align: middle;

    @media (prefers-color-scheme: dark) {
      color: #fff;
    }
  }

  &.show-key i {
    opacity: 1;
  }
}

.title-block {
  width: 100%;
  text-align: right;
}

.mapkey .title {
  vertical-align: middle;
  margin: 0 8px;
  display: none;
}

.keys {
  display: none;
}

.show-key {
  display: block;
}

.mapkey.show-key .title {
  display: inline;
}

.location-alert {
  position: absolute;
  bottom: 0px;
  left: calc(50% - 175px);
  width: 350px;
  z-index: 1000;
}

.leaflet-bottom .leaflet-control-zoom {
  margin-bottom: 26px !important;
  @media (pointer: coarse) {
    visibility: hidden;
  }
  @media (pointer: none) {
    visibility: hidden;
  }
}

.leaflet-control-zoom a:hover {
  background-color: #f4f4f4 !important;

  @media (prefers-color-scheme: dark) {
    background-color: $gray-300 !important;
  }
}

.leaflet-control-zoom a.leaflet-disabled {
  background-color: #f4f4f4 !important;

  @media (prefers-color-scheme: dark) {
    background-color: $gray-300 !important;
  }
}

.user-location-button {
  @media (pointer: coarse) {
    bottom: 0px !important;
  }
  @media (pointer: none) {
    bottom: 0px !important;
  }
  @media (pointer: fine) {
    bottom: 68px !important;
  }
  right: 0px !important;
}

.user-location-link {
  border-radius: 2.5px;
  background-position: 50% 50%;
  background-repeat: no-repeat;
  background-color: $white;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.65);
  width: 26px;
  height: 26px;
  display: block;
  line-height: 26px;
  text-align: center;
  color: #000 !important;

  @media (pointer: none) {
    width: 50px;
    height: 50px;
    line-height: 50px;
    border-radius: 25px;
    font-size: 18px;
  }
  @media (pointer: coarse) {
    width: 50px;
    height: 50px;
    line-height: 50px;
    border-radius: 25px;
    font-size: 18px;
  }

  &:hover {
    background-color: #f4f4f4;

    @media (prefers-color-scheme: dark) {
      background-color: $gray-300 !important;
    }
  }

  &.active {
    color: theme-color('primary') !important;
  }

  &.disabled {
    color: theme-color('#bbb') !important;
  }
}

.user-location-link1 {
  border-radius: 20px;
  background-position: 50% 50%;
  background-repeat: no-repeat;
  background-color: $white;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.65);
  width: 30px;
  height: 30px;
  display: block;
  line-height: 30px;
  text-align: center;
  color: rgba(0, 0, 0, 0.7) !important;

  @media (pointer: none) {
    width: 50px;
    height: 50px;
    line-height: 50px;
    border-radius: 25px;
    font-size: 18px;
  }

  @media (pointer: coarse) {
    width: 50px;
    height: 50px;
    line-height: 50px;
    border-radius: 25px;
    font-size: 18px;
  }

  &:hover {
    background-color: #f4f4f4;

    @media (prefers-color-scheme: dark) {
      background-color: $gray-300 !important;
    }
  }

  &.active {
    color: theme-color('primary') !important;
  }

  &.disabled {
    color: theme-color('#bbb') !important;
  }
}

.user-location-link2 {
  border-radius: 15px;
  background-color: $white;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.65);
  width: 110px;
  height: 25px;
  display: flex;
  line-height: 15px;
  text-align: center;
  padding-top: 7px;
  color: rgba(0, 0, 0, 0.3) !important;

  &:hover {
    background-color: #f4f4f4;

    @media (prefers-color-scheme: dark) {
      background-color: $gray-300 !important;
    }
  }

  &.active {
    color: theme-color('primary') !important;
  }

  &.disabled {
    color: theme-color('#bbb') !important;
  }

  h5 {
    color: rgba(0, 0, 0, 0.3);
    font-size: 11px;
    font-family: Geneva, sans-serif;
    padding-right: 10px;
    margin-left: 8px;
  }
}
</style>
