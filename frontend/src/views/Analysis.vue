<template>
    <v-container>

        <!-- Date Range Controls & Average Card -->
        <v-row class="mb-2" no-gutters style="align-items: stretch;">

            <!-- Left: Dates + Button -->
            <v-col cols="auto" class="pr-4">
    <div class="d-flex flex-column ga-2">
        <v-text-field
            v-model="startDate"
            label="Start Date"
            type="date"
            variant="outlined"
            density="compact"
            hide-details
            style="width: 170px;"
        />
        <v-text-field
            v-model="endDate"
            label="End Date"
            type="date"
            variant="outlined"
            density="compact"
            hide-details
            style="width: 170px;"
        />
        <div class="d-flex justify-center" style="width: 170px;">
            <v-btn
                color="grey-lighten-2"
                variant="flat"
                :loading="loading"
                size="small"
                style="text-transform: none; font-weight: 400; width: 90px; color: #444;"
                @click="analyzeData"
            >
                Analyze
            </v-btn>
        </div>
    </div>
</v-col>

            <!-- Right: Average Card -->
            <v-col cols="auto">
                <v-sheet
                    elevation="2"
                    class="pa-4 d-flex flex-column align-center justify-center"
                    style="border-radius: 8px; height: 100%; width: 300px;"
                >
                    <div class="text-subtitle-1 font-weight-bold">Average</div>
                    <div class="text-caption text-medium-emphasis mb-1">For the selected period</div>
                    <div class="average-value">
                        <span v-if="hasData">
                            {{ averageReserve.toFixed(1) }}<span class="gal-unit">Gal</span>
                        </span>
                        <span v-else class="text-medium-emphasis">—</span>
                    </div>
                </v-sheet>
            </v-col>

        </v-row>

        <!-- Water Management Analysis (Reserve Line Chart) -->
        <v-row class="mt-3">
            <v-col cols="12">
                <v-sheet elevation="2" class="pa-4">
                    <div id="reserveChart" style="height: 400px;"></div>
                </v-sheet>
            </v-col>
        </v-row>

        <!-- Height and Water Level Correlation (Scatter Chart) -->
        <v-row>
            <v-col cols="12">
                <v-sheet elevation="2" class="pa-4">
                    <div id="scatterChart" style="height: 400px;"></div>
                </v-sheet>
            </v-col>
        </v-row>

    </v-container>
</template>

<script setup>
// IMPORTS
import { ref, watch, onMounted, onBeforeUnmount } from "vue";
import { useRoute, useRouter } from "vue-router";
import { useMqttStore } from "@/store/mqttStore";
import { useAppStore } from "@/store/appStore";
import { storeToRefs } from "pinia";

// Highcharts — load extras and initialise modules
import Highcharts from "highcharts";
import more from "highcharts/highcharts-more";
import Exporting from "highcharts/modules/exporting";

more(Highcharts);
Exporting(Highcharts);

// VARIABLES
const router   = useRouter();
const route    = useRoute();
const Mqtt     = useMqttStore();
const AppStore = useAppStore();
const { payload, payloadTopic } = storeToRefs(Mqtt);

const reserveChart = ref(null);
const scatterChart = ref(null);

const startDate      = ref("");
const endDate        = ref("");
const loading        = ref(false);
const averageReserve = ref(0);
const hasData        = ref(false);

const correlationPoints = ref([]);

// WATCHERS

watch(payload, (data) => {
    if (reserveChart.value) {
        reserveChart.value.series[0].addPoint(
            { x: data.timestamp * 1000, y: parseFloat(data.reserve.toFixed(2)) },
            true,
            reserveChart.value.series[0].data.length >= 600
        );
    }

    if (scatterChart.value) {
        correlationPoints.value.push([
            parseFloat(data.waterheight.toFixed(2)),
            parseFloat(data.radar.toFixed(2))
        ]);
        scatterChart.value.series[0].setData(correlationPoints.value, true);
    }
});

// FUNCTIONS

const CreateCharts = () => {
    reserveChart.value = Highcharts.chart("reserveChart", {
        chart:   { zoomType: "x" },
        title:   { text: "Water Management Analysis", align: "left" },
        tooltip: { shared: true },
        xAxis: {
            type:  "datetime",
            title: { text: "Time" },
            tickLength: 6,
            min: Date.UTC(2024, 0, 1, 14, 40),
            max: Date.UTC(2024, 0, 1, 16,),
            tickInterval: 10 * 60 * 1000,
            labels: { format: "{value:%H:%M}"}
},
        yAxis: {
            title:  { text: "Water Reserve" },
            labels: { format: "{value} Gal" },
            min: 0,
            max: 1250,
            tickInterval: 250,
            gridLineWidth: 1,
            gridLineColor: "#e0e0e0",
            showFirstLabel: true,
        },
        legend: { enabled: true, align: "center" },
        series: [{
            name:  "Reserve",
            type:  "line",
            data:  [],
            color: "#00FFFF"
        }]
    });

    scatterChart.value = Highcharts.chart("scatterChart", {
        chart:   { type: "scatter", zoomType: "xy" },
        title:   { text: "Height and Water Level Correlation Analysis", align: "left" },
        tooltip: {
            headerFormat: "",
            pointFormat:  "Water Height: <b>{point.x} in</b><br/>Height: <b>{point.y} in</b>"
        },
        xAxis: {
            title:  { text: "Water Height" },
            labels: { format: "{value} in" },
            min: 0,
            max: 90,
            tickInterval: 10,
        },
        yAxis: {
            title:  { text: "Height" },
            labels: { format: "{value} in" },
            min: 0,
            max: 100,
            tickInterval: 25,
            gridLineWidth: 1,
            gridLineColor: "#e0e0e0",
        },
        legend: { enabled: true, align: "center" },
        series: [{
            name:   "Analysis",
            data:   [],
            color:  "#00FFFF",
            marker: { radius: 4 }
        }]
    });
};

const analyzeData = async () => {
    if (!startDate.value || !endDate.value) return;

    loading.value = true;
    try {
        // Convert date strings to Unix timestamps (seconds)
        const startTs = Math.floor(new Date(startDate.value).getTime() / 1000);
        const endTs   = Math.floor(new Date(endDate.value + "T23:59:59").getTime() / 1000);

        // Match the actual backend route: /api/reserve/<start>/<end>
        const res = await fetch(`/api/reserve/${startTs}/${endTs}`);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);

        const json = await res.json();

        // Backend returns { status, data: [...] }
        const records = json.data;

        if (Array.isArray(records) && records.length) {
            const sum = records.reduce((acc, r) => acc + r.reserve, 0);
            averageReserve.value = sum / records.length;
            hasData.value = true;

            const reservePoints = records.map(r => ({
                x: r.timestamp * 1000,   // MongoDB stores seconds → ms for Highcharts
                y: parseFloat(r.reserve.toFixed(2))
            }));
            reserveChart.value.series[0].setData(reservePoints, true);

            const scatterPoints = records.map(r => [
                parseFloat(r.waterheight.toFixed(2)),
                parseFloat(r.radar.toFixed(2))
            ]);
            correlationPoints.value = scatterPoints;
            scatterChart.value.series[0].setData(scatterPoints, true);

        } else {
            averageReserve.value = 0;
            hasData.value = false;
            reserveChart.value.series[0].setData([], true);
            scatterChart.value.series[0].setData([], true);
        }
    } catch (err) {
        console.error("Failed to fetch sensor data:", err);
    } finally {
        loading.value = false;
    }
};
// LIFECYCLE

onMounted(() => {
    CreateCharts();
    Mqtt.connect();
    setTimeout(() => {
        Mqtt.subscribe("620169689");
    }, 3000);
});

onBeforeUnmount(() => {
    Mqtt.unsubscribeAll();
});
</script>

<style scoped>
.average-value {
    font-size: 3.5rem;
    font-weight: 300;
    line-height: 1;
    color: #1a1a1a;
    display: flex;
    align-items: flex-end;
    gap: 4px;
}

.gal-unit {
    font-size: 1.2rem;
    font-weight: 400;
    color: #888;
    margin-bottom: 6px;
}
</style>