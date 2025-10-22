<template>
  <div class="p-4">
    <h2 class="mb-2 font-bold text-xl">Finland Traffic Map (2014–2021)</h2>

    <div class="controls mb-4">
      <div class="control">
        <label>Years</label>
        <select v-model="selectedYears" multiple>
          <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
        </select>
      </div>

      <div class="control">
        <label>Region</label>
        <select v-model="selectedRegion">
          <option value="">(All)</option>
          <option v-for="r in regionList" :key="r.id" :value="r.id">{{ r.name }}</option>
        </select>
      </div>

      <div class="control">
        <label>Region Borders</label>
        <button @click="toggleOutlines">
          {{ showOutlines ? 'Hide Borders' : 'Show Borders' }}
        </button>
      </div>

      <button @click="resetFilters">Reset</button>
    </div>

    <div class="map-container">
      <div id="map"></div>

      <!-- 🎨 Year Color Legend -->
      <div class="legend">
        <h3>Year Colors</h3>
        <ul>
          <li v-for="(color, year) in yearColors" :key="year">
            <span class="legend-color" :style="{ backgroundColor: color }"></span>
            {{ year }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref, watch } from "vue";
import * as Plotly from "plotly.js-dist-min";
import * as d3 from "d3";
import * as turf from "@turf/turf";

const CSV_URL =
  "https://raw.githubusercontent.com/AnUnkownUser/Intro-to-data-science-Project/refs/heads/main/FinalData.csv";
const GEOJSON_URL =
  "https://raw.githubusercontent.com/AnUnkownUser/Intro-to-data-science-Project/refs/heads/main/fi.json";

// --- Reactive state ---
const years = ref([]);
const regionList = ref([]);
const selectedYears = ref([]);
const selectedRegion = ref("");
const showOutlines = ref(false);

let allRows = [];
let regions = {};
let regionTraceIndices = []; // store indices for toggling region outlines

const yearColors = {
  2014: "#1f77b4",
  2015: "#ff7f0e",
  2016: "#2ca02c",
  2017: "#d62728",
  2018: "#9467bd",
  2019: "#8c564b",
  2020: "#17becf",
  2021: "#7f7f7f",
};

// --- Load data + initialize map ---
onMounted(async () => {
  const [csv, geo] = await Promise.all([
    d3.csv(CSV_URL),
    fetch(GEOJSON_URL).then((r) => r.json()),
  ]);

  console.log("✅ CSV loaded:", csv.slice(0, 5));
  console.log("✅ GeoJSON loaded:", geo.features.map((f) => f.properties.name));

  // Parse CSV
  allRows = csv
    .map((d) => ({
      id: d.id,
      year: +d.year,
      lon: +d.lon,
      lat: +d.lat,
      seriousness: d.Seriousness,
      vehicle_mass: d.vehicle_mass,
    }))
    .filter((d) => !isNaN(d.lon) && !isNaN(d.lat));

  years.value = [...new Set(allRows.map((d) => d.year))].sort();
  selectedYears.value = [...years.value];

  // Parse region GeoJSON
  geo.features.forEach((f) => {
    const id = f.properties.name;
    const name = f.properties.name;
    if (!f.geometry) return;
    try {
      const bbox = turf.bbox(f);
      regions[id] = { id, name, bbox, feature: f };
    } catch (err) {
      console.warn("⚠️ Skipped region with invalid geometry:", name, err);
    }
  });

  regionList.value = Object.values(regions).sort((a, b) =>
    a.name.localeCompare(b.name)
  );

  drawMap(allRows, geo);
});

// --- Draw base map ---
function drawMap(filtered, geo) {
  const trace = {
    type: "scattermapbox",
    mode: "markers",
    lat: filtered.map((d) => d.lat),
    lon: filtered.map((d) => d.lon),
    marker: {
      size: 3,
      opacity: 0.5,
      color: filtered.map((d) => yearColors[d.year] || "#999"),
    },
  };

  const layout = {
    mapbox: {
      style: "open-street-map",
      center: { lon: 25, lat: 64 },
      zoom: 4.3,
    },
    margin: { t: 0, b: 0, l: 0, r: 0 },
    showlegend: false,
  };

  Plotly.newPlot("map", [trace], layout, { scrollZoom: true });

  // --- Add region outlines (always black) ---
  const regionOutlines = geo.features
    .map((f) => {
      const coords = f.geometry.coordinates;
      const name = f.properties.name;
      const polygons = f.geometry.type === "Polygon" ? [coords] : coords;

      return polygons.map((poly) => ({
        type: "scattermapbox",
        mode: "lines",
        lon: poly[0].map((c) => c[0]),
        lat: poly[0].map((c) => c[1]),
        line: { width: 2, color: "#000" },
        hoverinfo: "text",
        text: name,
        visible: false, // hidden initially
      }));
    })
    .flat();

  Plotly.addTraces("map", regionOutlines);
  regionTraceIndices = regionOutlines.map((_, i) => i + 1);
}

// --- Update map when filters change ---
function updateMap() {
  let filtered = allRows.filter((d) => selectedYears.value.includes(d.year));

  if (selectedRegion.value && regions[selectedRegion.value]) {
    const region = regions[selectedRegion.value];
    filtered = filtered.filter((d) =>
      turf.booleanPointInPolygon([d.lon, d.lat], region.feature)
    );
    const [minX, minY, maxX, maxY] = region.bbox;
    const center = { lon: (minX + maxX) / 2, lat: (minY + maxY) / 2 };

    setTimeout(() => {
      Plotly.relayout("map", {
        "mapbox.center": center,
        "mapbox.zoom": 6,
      });
    }, 100);
  } else {
    setTimeout(() => {
      Plotly.relayout("map", {
        "mapbox.center": { lon: 25, lat: 64 },
        "mapbox.zoom": 4.3,
      });
    }, 100);
  }

  // ✨ Always color dots by their year
  const markerColors = filtered.map((d) => yearColors[d.year] || "#999");

  setTimeout(() => {
    Plotly.restyle(
      "map",
      {
        lon: [filtered.map((d) => d.lon)],
        lat: [filtered.map((d) => d.lat)],
        "marker.color": [markerColors],
      },
      [0]
    );
  }, 150);
}

// --- Toggle region outlines ---
function toggleOutlines() {
  showOutlines.value = !showOutlines.value;
  const visibility = showOutlines.value ? true : "legendonly";

  // Always black outlines
  Plotly.restyle(
    "map",
    { visible: visibility, "line.color": "#000" },
    regionTraceIndices
  );
}

// --- Reset filters ---
function resetFilters() {
  selectedYears.value = [...years.value];
  selectedRegion.value = "";
  updateMap();
}

watch([selectedYears, selectedRegion], updateMap);
</script>

<style scoped>
body {
  background: #f7f8fa;
  font-family: "Inter", "Segoe UI", sans-serif;
  color: #222;
}

h2 {
  text-align: center;
  margin-bottom: 0.75rem;
}

.controls {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 1rem;
}

.control {
  display: flex;
  flex-direction: column;
  font-size: 0.9rem;
}

select {
  padding: 4px;
  border-radius: 6px;
  border: 1px solid #ccc;
  min-width: 160px;
  height: 100px;
  background: white;
}

button {
  align-self: center;
  padding: 8px 14px;
  background: #2b74ff;
  border: none;
  color: white;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
}

button:hover {
  background: #1858db;
}

/* === Layout and legend === */
.map-container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  gap: 1.5rem;
  margin-top: 1rem;
}

#map {
  flex: 1;
  height: 80vh;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

.legend {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 10px 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  font-size: 0.9rem;
  line-height: 1.4;
  max-height: 80vh;
  overflow-y: auto;
  color: #111;
}

.legend h3 {
  margin: 0 0 8px;
  font-size: 1rem;
  font-weight: 600;
  text-align: center;
}

.legend ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.legend li {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 4px 0;
}

.legend-color {
  display: inline-block;
  width: 16px;
  height: 16px;
  border-radius: 4px;
  border: 1px solid #000;
}
</style>
