<template>
  <div>
    <h1>Weather Forecast</h1>
    <div class="search-container" @click="closeDropdown">
      <input
        v-model="query"
        placeholder="Search for a city..."
        @keyup.enter="searchLocation"
        @focus="showDropdown = true"
        @input="searchLocation"
      />
      <div class="dropdown" v-if="showDropdown && locations.length">
        <ul>
          <li v-for="location in locations" :key="location.id" @click="selectLocation(location)">
            {{ location.name }} ({{ location.country }})
          </li>
        </ul>
      </div>
      <div v-if="loading" class="spinner">Loading...</div> <!-- Loading spinner -->
      <div v-if="errorMessage" class="error">{{ errorMessage }}</div> <!-- Error message -->
    </div>
    <div v-if="weather">
      <h2>Current Weather in {{ selectedLocation.name }}</h2>
      <p>Temperature: {{ weather.current_weather.temperature }}°C</p>
      <p>Wind Speed: {{ weather.current_weather.windspeed }} km/h</p>

      <h3>Hourly Forecast</h3>
      <canvas id="weatherChart" width="400" height="200"></canvas>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import { Chart as ChartJS, Title, Tooltip, Legend, LineElement, PointElement, LinearScale, CategoryScale, LineController } from 'chart.js';
import { nextTick } from 'vue';

// Register necessary Chart.js components
ChartJS.register(Title, Tooltip, Legend, LineElement, PointElement, LinearScale, CategoryScale, LineController);

export default {
  data() {
    return {
      query: '',
      locations: [],
      weather: null,
      selectedLocation: null,
      chartData: {
        labels: [],
        datasets: [{
          label: 'Temperature (°C)',
          backgroundColor: 'rgba(75, 192, 192, 0.2)',
          borderColor: 'rgba(75, 192, 192, 1)',
          borderWidth: 1,
          data: [],
        }],
      },
      weatherChart: null,
      showDropdown: false,
      loading: false, // For loading spinner
      errorMessage: null, // For error messages
    };
  },
  methods: {
    async searchLocation() {
      this.errorMessage = null; // Reset error message
      this.loading = true; // Start loading
      try {
        const response = await axios.get(`https://geocoding-api.open-meteo.com/v1/search?name=${this.query}&count=10&language=en&format=json`);
        this.locations = response.data.results;
        this.showDropdown = this.locations.length > 0; // Show dropdown only if locations are found
      } catch (error) {
        console.error("Error fetching location data:", error);
        this.errorMessage = "Failed to fetch location data. Please try again."; // Set error message
      } finally {
        this.loading = false; // Stop loading
      }
    },
    selectLocation(location) {
      this.selectedLocation = location;
      this.showDropdown = false; 
      this.getWeather(location);
    },
    async getWeather(location) {
      const { latitude, longitude } = location;
      this.loading = true; // Start loading
      this.errorMessage = null; // Reset error message
      try {
        const response = await axios.get(`https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&hourly=temperature_2m&current_weather=true`);
        this.weather = response.data;

        // Prepare chart data
        this.chartData.labels = this.weather.hourly.time.map(time => this.formatTime(time));
        this.chartData.datasets[0].data = this.weather.hourly.temperature_2m;

        // Render chart after the DOM has been updated
        await nextTick();
        this.renderChart();
      } catch (error) {
        console.error("Error fetching weather data:", error);
        this.errorMessage = "Failed to fetch weather data. Please try again."; // Set error message
      } finally {
        this.loading = false; // Stop loading
      }
    },
    formatTime(timestamp) {
      const date = new Date(timestamp);
      return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    },
    renderChart() {
      const ctx = document.getElementById('weatherChart').getContext('2d');

      // Destroy any existing chart before creating a new one
      if (this.weatherChart) {
        this.weatherChart.destroy();
      }

      this.weatherChart = new ChartJS(ctx, {
        type: 'line',
        data: this.chartData,
        options: {
          responsive: true,
          plugins: {
            legend: {
              display: true,
            },
          },
          scales: {
            x: {
              title: {
                display: true,
                text: 'Time',
              },
            },
            y: {
              title: {
                display: true,
                text: 'Temperature (°C)',
              },
              beginAtZero: true,
            },
          },
        },
      });
    },
    closeDropdown() {
      this.showDropdown = false;
    },
  },
};
</script>

<style scoped>
.search-container {
  position: relative;
}

input {
  padding: 8px;
  margin-bottom: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 300px;
  font-size: 16px;
}

.dropdown {
  position: absolute;
  z-index: 1;
  background-color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  width: 300px;
  max-height: 200px;
  overflow-y: auto;
}

ul {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

li {
  cursor: pointer;
  padding: 10px;
  transition: background-color 0.2s;
}

li:hover {
  background-color: #f0f0f0;
}

canvas {
  border: 1px solid black; /* Temporary for visibility */
  margin-top: 20px; /* Add some space above the chart */
}

p {
  margin-left: 20px;
}

.spinner {
  color: #007BFF; /* Loading spinner color */
  margin-top: 10px;
}

.error {
  color: red; /* Error message color */
  margin-top: 10px;
}
</style>
