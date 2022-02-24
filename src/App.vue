<template>
  <div class="flex flex-col w-screen h-screen text-[#061944] font-medium text-xl">
    <!-- Table heading -->
    <div class="relative grid grid-cols-10 w-full bg-[#d8e0ed] h-min px-8 py-1">
      <span class="col-span-1">Vertrek</span>
      <span class="col-span-6">Naar / Opmerkingen</span>
      <span class="col-span-1">Spoor</span>
      <span class="col-span-2">Trein</span>

      <!-- Selected station -->
      <span class="absolute left-1/3 bg-[#061944] text-white px-8 py-1">{{ selectedStation?.namen?.lang }}</span>

      <!-- Current time -->
      <span class="absolute right-0 bg-[#061944] text-white px-8 py-1">{{ currentTime }}</span>
    </div>

    <!-- List of times -->
    <div class="grid grid-rows-7 bg-[#d8e0ed] h-full">
      <div v-for="item in departures" :key="item" class="grid grid-cols-10 w-full px-8 odd:bg-white" :class="item.cancelled ? 'text-gray-400' : ''">
        <div class="col-span-1 text-6xl flex flex-col w-min pt-3">
          <span>{{ item.departure }}</span>
          <span v-if="item.delay !== 0" class="text-red-600 self-end">+{{ item.delay }}</span>
        </div>
        <div class="col-span-6 flex flex-col justify-between pb-2 overflow-hidden pt-3">
          <span class="text-6xl">{{ item.destination }}</span>
          <div v-if="((item.stops && item.remarks.length === 0) || !showRemarks) && !item.cancelled">
            <span v-for="(station, index) in item.stops" :key="station">
              <span class="text-4xl pb-4">{{ station }}{{ index < item.stops.length - 1 ? ', ' : '' }}</span>
            </span>
          </div>
          <div v-else>
            <span :class="['text-4xl py-2', item.remarks[0].style === 'WARNING' ? 'text-red-600' : 'bg-[#061944] text-white px-4']">{{ item.remarks[0].message }}</span>
          </div>
        </div>
        <div class="relative col-span-1 pt-2">
          <div class="relative border-[3px] border-[#061944] my-2 aspect-[0.9/1] w-24 flex items-center justify-center bg-white" :class="item.cancelled ? 'border-gray-400' : ''">
            <div class="absolute top-0 left-0 w-4 h-4 bg-[#061944]" :class="item.cancelled ? 'bg-gray-400' : ''">&nbsp;</div>
            <span v-if="item.actualTrack" class="text-5xl leading-none text-red-600">{{ item.actualTrack }}</span>
            <span v-else class="text-5xl leading-none">{{ item.plannedTrack }}</span>
          </div>
        </div>
        <span class="col-span-2 text-5xl pt-3">{{ item.train }}</span>
      </div>
    </div>

    <!-- Search dialog -->
    <Dialog :open="isOpen" @close="setIsOpen" as="div" class="fixed inset-0 overflow-y-auto p-4 pt-[25vh]">
      <DialogOverlay class="fixed inset-0 bg-gray-500/75" />

      <Combobox v-model="selectedStation" as="div" class="relative mx-auto max-w-xl divide-y divide-gray-100 overflow-hidden rounded-xl bg-white shadow-2xl ring-1 ring-black/5">
        <div class="flex items-center px-4 space-x-2">
          <SearchIcon class="h-6 w-6 text-gray-500" />
          <ComboboxInput @change="query = $event.target.value" class="h-12 w-full border-0 bg-transparent text-base text-gray-800 placeholder-gray-400 focus:ring-0 outline-none" placeholder="Zoek station" aria-autocomplete="off" />
        </div>

        <ComboboxOptions static class="max-h-96 overflow-y-auto py-4 text-sm">
          <ComboboxOption v-for="item in filteredStations" :key="item" :value="item" v-slot="{ active }">
            <div :class="['px-4 py-2 cursor-pointer', active ? 'bg-[#061944]' : 'bg-white']">
              <span :class="['font-medium text-lg', active ? 'text-white' : 'text-gray-900']">{{ item?.namen?.lang }}</span>
            </div>
          </ComboboxOption>
          <ComboboxOption v-if="filteredStations.length === 0">
            <div class="px-4 py-2 bg-white">
              <span class="font-medium text-lg italic text-gray-600">Geen stations gevonden</span>
            </div>
          </ComboboxOption>
        </ComboboxOptions>
      </Combobox>
    </Dialog>
  </div>
</template>

<script>
import moment from 'moment'
import axios from 'axios'
import { ref } from 'vue'
import {
  Combobox,
  ComboboxInput,
  ComboboxOptions,
  ComboboxOption,
  Dialog,
  DialogOverlay,
  DialogTitle,
  DialogDescription,
} from '@headlessui/vue'

import { SearchIcon } from '@heroicons/vue/outline'

export default {
  name: 'App',
  components: {
    Combobox,
    ComboboxInput,
    ComboboxOptions,
    ComboboxOption,
    Dialog,
    DialogOverlay,
    DialogTitle,
    DialogDescription,
    SearchIcon
  },
  data: () => ({
    currentTime: '',
    stations: [],
    selectedStation: {},
    query: '',
    showRemarks: false,
    departures: []
  }),
  setup() {
    let isOpen = ref(true)

    return {
      isOpen,
      setIsOpen(value) {
        isOpen.value = value
      }
    }
  },
  mounted () {
    axios.get('https://gateway.apiportal.ns.nl/reisinformatie-api/api/v2/stations', {
      headers: {
        'Ocp-Apim-Subscription-Key': '429ce2d802e34d9c92a0b36f7294e1c9'
      }
    }).then((response) => {
      this.stations = response.data.payload
    })

    window.addEventListener('keydown', (event) => {
      if (event.key === 'k' && (event.metaKey || event.ctrlKey) && !this.isOpen) {
        event.preventDefault()
        this.setIsOpen(true) 
      }
    })

    setInterval(() => {
      this.currentTime = moment().format('HH:mm')
    }, 1000)

    setInterval(() => {
      this.showRemarks = !this.showRemarks
    }, 3500)

    setInterval(() => {
      this.getStationData(this.selectedStation)
    }, 10000)
  },
  computed: {
    filteredStations () {
      if (this.query) return this.stations.filter((station) => {
        return station.namen.lang.toLowerCase().includes(this.query.toLowerCase())
      }).splice(0, 20)

      return this.stations.splice(0, 20)
    }
  },
  methods: {
    getStationData (station) {
      if (station?.code) {
        const url = `https://gateway.apiportal.ns.nl/reisinformatie-api/api/v2/departures?station=${station.code}`

        axios.get(url, {
          headers: {
            'Ocp-Apim-Subscription-Key': '429ce2d802e34d9c92a0b36f7294e1c9'
          }
        }).then((response) => {
          this.departures = response.data.payload.departures.splice(0, 7).map((departure) => {
            return {
              departure: moment(departure.plannedDateTime).format('HH:mm'),
              delay: moment(departure.actualDateTime).diff(moment(departure.plannedDateTime), 'minutes'),
              destination: departure.direction,
              stops: departure.routeStations.map((station) => station.mediumName),
              remarks: departure.messages,
              plannedTrack: departure.plannedTrack,
              actualTrack: departure.actualTrack,
              train: departure.product.shortCategoryName,
              cancelled: departure.cancelled
            }
          })
        })
      }
    }
  },
  watch: {
    selectedStation: {
      handler (station) {
        this.isOpen = false
        this.getStationData(station)
      }
    }
  }
}
</script>
