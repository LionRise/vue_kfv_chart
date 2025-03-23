<template>
  <div class="p-6 bg-white rounded-xl shadow-md">
    <div v-if="isLoading" class="text-center text-lg font-semibold text-gray-500">
      Daten werden geladen...
    </div>

    <div v-else>
      <h1 class="text-2xl font-bold text-gray-800 mb-1">
        Getötete im Straßenverkehr in Österreich
      </h1>
      <h2 class="text-lg text-gray-600 mb-6">
        nach Bundesland und Berichtsjahr
      </h2>

      <!-- Filter Section -->
      <div class="flex flex-wrap gap-4 items-end mb-6">
        <div class="flex flex-wrap gap-4 flex-grow">
          <Filter v-for="filter in filters" :key="filter.id" :id="filter.id" :label="filter.label"
            :options="filter.options" :displayMap="filter.displayMap" :modelValue="filter.model.value"
            @update:modelValue="val => filter.model.value = val" @change="updateChart" class="min-w-[180px] flex-1" />
        </div>
        <div>
          <button @click="resetFilters"
            class="h-[42px] px-4 py-2 bg-red-500 text-white rounded-md shadow-sm hover:bg-red-600 transition text-sm">
            Filter zurücksetzen
          </button>
        </div>
      </div>
    </div>

    <!-- Chart Section -->
    <div class="rounded-lg bg-gray-50 p-4 shadow-inner">
      <VueApexCharts ref="chartRef" type="area" :series="areaChartSeries" :options="areaChartOptions" height="450" />
    </div>

    <!-- Source -->
    <p class="mt-4 text-xs text-gray-500 text-right">
      Quelle:
      <a href="https://www.kfv.at/" target="_blank" rel="noopener noreferrer"
        class="underline hover:text-gray-700 transition">
        KFV - Kuratorium für Verkehrssicherheit
      </a>
    </p>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from "vue";
import VueApexCharts from "vue3-apexcharts";
import Filter from "./components/FilterComponent.vue";

const apiUrl = 'https://dashboards.kfv.at/api/udm_verkehrstote/json';

// ID-to-label mappings
const stateMap: Record<string, string> = {
  "1": "Burgenland",
  "2": "Kärnten",
  "3": "Niederösterreich",
  "4": "Oberösterreich",
  "5": "Salzburg",
  "6": "Steiermark",
  "7": "Tirol",
  "8": "Vorarlberg",
  "9": "Wien"
};

const monthMap: Record<string, string> = {
  "1": "Januar",
  "2": "Februar",
  "3": "März",
  "4": "April",
  "5": "Mai",
  "6": "Juni",
  "7": "Juli",
  "8": "August",
  "9": "September",
  "10": "Oktober",
  "11": "November",
  "12": "Dezember"
};

const weekdayMap: Record<string, string> = {
  "1": "Montag",
  "2": "Dienstag",
  "3": "Mittwoch",
  "4": "Donnerstag",
  "5": "Freitag",
  "6": "Samstag",
  "7": "Sonntag"
};

const sexMap: Record<string, string> = {
  "1": "Männlich",
  "2": "Weiblich"
};

// Reference to the chart instance
const chartRef = ref<ApexCharts | null>(null);

// Type for each dataset record
type TrafficDeathRecord = {
  Berichtsjahr: string;
  Geschlecht_ID: string;
  Monat_ID: string;
  Wochentag_ID: string;
  Bundesland_ID: string;
  Getötete: string;
};

// Raw data from the api
const rawData = ref<TrafficDeathRecord[]>([]);

// Filters (v-model)
const selectedYear = ref<string>('');
const selectedSex = ref<string>('');
const selectedMonth = ref<string>('');
const selectedWeekday = ref<string>('');

const availableYears = ref<string[]>([]);
const availableSexes = ref<string[]>([]);
const availableMonths = ref<string[]>([]);
const availableWeekdays = ref<string[]>([]);

// Chart series (data) reference
const areaChartSeries = ref<ApexAxisChartSeries>([]);

// Chart config
const areaChartOptions = ref({
  chart: {
    type: 'area',
    stacked: true,
    toolbar: {
      show: true
    }
  },
  colors: [
    "#2b3990", "#c51d7f", "#c0c2c4", "#00a99d",
    "#008fd5", "#f6eb16", "#f9b233", "#ed1c24", "#00aeef"
  ],
  xaxis: {
    type: 'category',
    title: { text: "Berichtsjahr" },
    categories: [] as string[]
  },
  yaxis: {
    title: { text: "Getötete" }
  },
  legend: {
    position: 'top',
    horizontalAlign: 'left',
  },
  stroke: {
    curve: 'smooth',
    width: 2
  },
  dataLabels: {
    enabled: true,
  },
  tooltip: {
    shared: true,
    intersect: false,
  },
  grid: {
    strokeDashArray: 3
  }
});

const isLoading = ref(true);

// Filter definitions
const filters = computed(() => [
  {
    id: "yearSelect",
    label: "Berichtsjahr",
    options: availableYears.value,
    model: selectedYear,
  },
  {
    id: "sexSelect",
    label: "Geschlecht",
    options: availableSexes.value,
    model: selectedSex,
    displayMap: sexMap,
  },
  {
    id: "monthSelect",
    label: "Monat",
    options: availableMonths.value,
    model: selectedMonth,
    displayMap: monthMap,
  },
  {
    id: "weekdaySelect",
    label: "Wochentag",
    options: availableWeekdays.value,
    model: selectedWeekday,
    displayMap: weekdayMap,
  },
]);

onMounted(() => {
  fetchData();
});

// Retrieve data, build filters, and update chart
async function fetchData() {
  isLoading.value = true;

  try {
    const response = await fetch(apiUrl);
    const jsonData = await response.json();
    rawData.value = jsonData.verkehrstote;

    const sets = {
      years: new Set<string>(),
      sexes: new Set<string>(),
      months: new Set<string>(),
      weekdays: new Set<string>()
    };

    rawData.value.forEach((item) => {
      sets.years.add(item.Berichtsjahr);
      sets.sexes.add(item.Geschlecht_ID);
      sets.months.add(item.Monat_ID);
      sets.weekdays.add(item.Wochentag_ID);
    });

    availableYears.value = Array.from(sets.years).sort((a, b) => parseInt(a) - parseInt(b));
    availableSexes.value = Array.from(sets.sexes).sort((a, b) => parseInt(a) - parseInt(b));
    availableMonths.value = Array.from(sets.months).sort((a, b) => parseInt(a) - parseInt(b));
    availableWeekdays.value = Array.from(sets.weekdays).sort((a, b) => parseInt(a) - parseInt(b));

    // Build the initial chart (unfiltered)
    updateChart();
  } catch (error) {
    console.error("Error fetching data:", error);
  } finally {
    isLoading.value = false;
  }
}

function getFilteredData(): TrafficDeathRecord[] {
  return rawData.value.filter(item => {
    if (selectedYear.value && item.Berichtsjahr !== selectedYear.value) return false;
    if (selectedSex.value && item.Geschlecht_ID !== selectedSex.value) return false;
    if (selectedMonth.value && item.Monat_ID !== selectedMonth.value) return false;
    if (selectedWeekday.value && item.Wochentag_ID !== selectedWeekday.value) return false;
    return true;
  });
}


// Updates the chart based on current filter selection
function updateChart() {
  const filteredData = getFilteredData();

  const aggregatedData: Record<string, Record<string, number>> = {};
  const yearSet = new Set<string>();
  const stateSet = new Set<string>();

  for (const item of filteredData) {
    const year = item.Berichtsjahr;
    const stateId = item.Bundesland_ID;
    const deaths = parseInt(item.Getötete) || 0;
    const state = stateMap[stateId] || `Bundesland-${stateId}`;

    yearSet.add(year);
    stateSet.add(state);

    if (!aggregatedData[year]) aggregatedData[year] = {};
    if (!aggregatedData[year][state]) aggregatedData[year][state] = 0;
    aggregatedData[year][state] += deaths;
  }

  const sortedYears = Array.from(yearSet).sort((a, b) => parseInt(a) - parseInt(b));
  const sortedStates = Array.from(stateSet).sort();

  const newSeries = sortedStates.map(state => ({
    name: state,
    data: sortedYears.map(year => aggregatedData[year]?.[state] || 0)
  }));

  if (chartRef.value) {
    chartRef.value.updateSeries(newSeries, true);
    chartRef.value.updateOptions({ xaxis: { categories: sortedYears } });
  }

  areaChartSeries.value = newSeries;
  areaChartOptions.value.xaxis.categories = sortedYears;
}

// Reset all filters and rebuild the chart
function resetFilters() {
  selectedYear.value = '';
  selectedSex.value = '';
  selectedMonth.value = '';
  selectedWeekday.value = '';
  updateChart();
}
</script>
