<template>
  <div class="baseball-table">
    <div v-if="isExplainerOpen" class="explanation-section">
      <button class="explainer-toggle" @click="toggleExplainer" aria-label="Collapse explainer">
        <svg
          width="20"
          height="20"
          viewBox="0 0 20 20"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M6 12L10 8L14 12"
            stroke="#1a237e"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
      </button>
      <h2>Baseball At-Bat Statistics (April 1-14, 2018)</h2>
      <p>
        This table displays detailed statistics from MLB at-bats during the first two weeks of April
        2018. Each row represents a single at-bat, showing the interaction between a batter and
        pitcher. The table also includes video links for select plays, allowing for visual analysis
        of the at-bats.
      </p>
      <h3>Legend</h3>
      <ul class="explainer-list">
        <li><strong>Batter</strong>: The batter's name</li>
        <li><strong>Pitcher</strong>: The pitcher's name</li>
        <li><strong>Game Date</strong>: The date of the game</li>
        <li>
          <strong>Launch Angle</strong>: The vertical angle at which the ball leaves the bat (0 is
          parallel to the ground)
        </li>
        <li><strong>Exit Speed</strong>: The velocity at which the ball leaves the bat</li>
        <li><strong>Hit Distance</strong>: The distance that the ball travels</li>
        <li><strong>Outcome</strong>: The result of the play</li>
        <li><strong>Video</strong>: A url containing video of the play</li>
      </ul>
      <h3>Filtering / Sorting</h3>
      <ul class="explainer-list">
        <li><strong>Filter by Batter</strong>: Filter the data by typing the batter's name</li>
        <li><strong>Filter by Pitcher</strong>: Filter the data by typing the pitcher's name</li>
        <li><strong>Filter by Outcome</strong>: Filter the data by selecting the outcome from the dropdown</li>
        <li><strong>Sort</strong>: Each column can be sorted by clicking on the column header</li>
      </ul>
    </div>
    <div v-else class="explanation-collapsed">
      <span>Show explainer / legend</span>
      <button
        class="explainer-toggle-collapsed"
        @click="toggleExplainer"
        aria-label="Expand explainer"
      >
        <svg
          width="20"
          height="20"
          viewBox="0 0 20 20"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M6 8L10 12L14 8"
            stroke="#1a237e"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
      </button>
    </div>
    <div v-if="isLoading" class="loading-container">
      <div class="loading-spinner"></div>
      <p>Loading baseball data...</p>
    </div>
    <div v-else>
      <div class="controls">
        <div class="search">
          <div class="search-group">
            <label for="batter-search" class="search-label">Filter by Batter</label>
            <div class="search-input-wrapper">
              <img :src="MagnifyingGlassIcon" alt="" class="search-icon" />
              <input
                id="batter-search"
                v-model="batterQuery"
                type="text"
                placeholder="Search batter name"
                @input="handleSearch"
                aria-label="Search by batter name"
              />
            </div>
          </div>
          <div class="search-group">
            <label for="pitcher-search" class="search-label">Filter by Pitcher</label>
            <div class="search-input-wrapper">
              <img :src="MagnifyingGlassIcon" alt="" class="search-icon" />
              <input
                id="pitcher-search"
                v-model="pitcherQuery"
                type="text"
                placeholder="Search pitcher name"
                @input="handleSearch"
                aria-label="Search by pitcher name"
              />
            </div>
          </div>
        </div>
        <div class="filters">
          <label for="outcome-filter" class="filter-label">Filter by Outcome</label>
          <select
            id="outcome-filter"
            v-model="selectedOutcome"
            @change="handleFilter"
            aria-label="Filter by play outcome"
          >
            <option value="">All Outcomes</option>
            <option v-for="outcome in uniqueOutcomes" :key="outcome" :value="outcome">
              {{ outcome }}
            </option>
          </select>
        </div>
        <div>
          <label for="outcome-filter" class="filter-label">&nbsp;</label>
          <button v-if="batterQuery || pitcherQuery || selectedOutcome" class="reset-button" @click="handleReset">
            <img :src="ResetIcon" alt="" class="reset-icon" />
            Reset Filters
          </button>
        </div>
      </div>

      <div class="pagination">
        <button :disabled="currentPage === 1" @click="currentPage--" class="pagination-button">
          Previous
        </button>
        <span class="page-info"> Page {{ currentPage }} of {{ totalPages }} </span>
        <button
          :disabled="currentPage === totalPages"
          @click="currentPage++"
          class="pagination-button"
        >
          Next
        </button>
      </div>

      <div v-if="filteredData.length === 0" class="no-results-container">
        <p>No results found for the current filters</p>
        <button class="reset-button" @click="handleReset">
          <img :src="ResetIcon" alt="" class="reset-icon" />
          Reset Filters
        </button>
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th
                v-for="column in columns"
                :key="column.key"
                @click="column.key !== 'VIDEO_LINK' && sortBy(column.key)"
                :class="{
                  sortable: column.key !== 'VIDEO_LINK',
                  sorted: sortKey === column.key,
                  [`col-${column.key.toLowerCase()}`]: true,
                }"
              >
                {{ column.label }}
                <span
                  v-if="sortKey === column.key && column.key !== 'VIDEO_LINK'"
                  class="sort-icon"
                >
                  {{ sortOrder === 'asc' ? '↑' : '↓' }}
                </span>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="row in paginatedData"
              :key="`${row.BATTER_ID}-${row.PITCHER_ID}-${row.GAME_DATE}-${row.LAUNCH_ANGLE}-${row.EXIT_SPEED}`"
            >
              <td
                v-for="column in columns"
                :key="column.key"
                :class="`col-${column.key.toLowerCase()}`"
              >
                <template v-if="column.key === 'VIDEO_LINK' && row[column.key]">
                  <a :href="row[column.key]" target="_blank" rel="noopener noreferrer">Video</a>
                </template>
                <template v-else>
                  {{ formatValue(row[column.key], column.key) }}
                </template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div v-if="filteredData.length > 0" class="pagination">
        <button :disabled="currentPage === 1" @click="currentPage--" class="pagination-button">
          Previous
        </button>
        <span class="page-info"> Page {{ currentPage }} of {{ totalPages }} </span>
        <button
          :disabled="currentPage === totalPages"
          @click="currentPage++"
          class="pagination-button"
        >
          Next
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import Papa from 'papaparse'
import MagnifyingGlassIcon from './icons/Magnifying_glass_icon.svg'
import ResetIcon from './icons/reset.png'

const isLoading = ref(true)
const data = ref([])
const batterQuery = ref('')
const pitcherQuery = ref('')
const sortKey = ref('GAME_DATE')
const sortOrder = ref('asc')
const currentPage = ref(1)
const itemsPerPage = 20
const selectedOutcome = ref('')

// Collapsible explainer state
const isExplainerOpen = ref(true)

const toggleExplainer = () => {
  isExplainerOpen.value = !isExplainerOpen.value
}

const columns = [
  { key: 'BATTER', label: 'Batter' },
  { key: 'PITCHER', label: 'Pitcher' },
  { key: 'GAME_DATE', label: 'Game Date' },
  { key: 'LAUNCH_ANGLE', label: 'Launch Angle' },
  { key: 'EXIT_SPEED', label: 'Exit Speed' },
  { key: 'HIT_DISTANCE', label: 'Hit Distance' },
  { key: 'PLAY_OUTCOME', label: 'Outcome' },
  { key: 'VIDEO_LINK', label: 'Video' },
]

const outcomeOrder = ['Out', "Fielder's Choice", 'Sacrifice', 'Single', 'Double', 'Triple', 'Home Run']

const sortByOutcome = (a, b, ascending = true) => {
  const indexA = outcomeOrder.indexOf(a)
  const indexB = outcomeOrder.indexOf(b)
  
  if (indexA !== -1 && indexB !== -1) {
    return ascending ? indexA - indexB : indexB - indexA
  }
  if (indexA !== -1) return ascending ? -1 : 1
  if (indexB !== -1) return ascending ? 1 : -1
  return ascending ? String(a).localeCompare(String(b)) : String(b).localeCompare(String(a))
}

const uniqueOutcomes = computed(() => {
  const outcomes = new Set(data.value.map((row) => row.PLAY_OUTCOME))
  return Array.from(outcomes).sort((a, b) => sortByOutcome(a, b))
})

const filteredData = computed(() => {
  let result = data.value

  if (batterQuery.value) {
    const query = batterQuery.value.toLowerCase()
    result = result.filter((row) => String(row.BATTER).toLowerCase().includes(query))
  }

  if (pitcherQuery.value) {
    const query = pitcherQuery.value.toLowerCase()
    result = result.filter((row) => String(row.PITCHER).toLowerCase().includes(query))
  }

  if (selectedOutcome.value) {
    result = result.filter((row) => row.PLAY_OUTCOME === selectedOutcome.value)
  }

  return result
})

const sortedData = computed(() => {
  const sorted = [...filteredData.value]

  sorted.sort((a, b) => {
    const aValue = a[sortKey.value]
    const bValue = b[sortKey.value]

    // Handle null/undefined values
    if (aValue === null || aValue === undefined) return 1
    if (bValue === null || bValue === undefined) return -1

    let comparison = 0

    // Handle dates
    if (sortKey.value === 'GAME_DATE') {
      const dateA = new Date(aValue)
      const dateB = new Date(bValue)
      comparison = sortOrder.value === 'asc' ? dateA - dateB : dateB - dateA
    }
    // Handle PLAY_OUTCOME with custom ordering
    else if (sortKey.value === 'PLAY_OUTCOME') {
      comparison = sortByOutcome(aValue, bValue, sortOrder.value === 'asc')
    }
    // Handle numbers
    else if (typeof aValue === 'number' && typeof bValue === 'number') {
      comparison = sortOrder.value === 'asc' ? aValue - bValue : bValue - aValue
    }
    // Handle strings with case-insensitive comparison
    else {
      const strA = String(aValue).toLowerCase()
      const strB = String(bValue).toLowerCase()
      comparison = sortOrder.value === 'asc' ? strA.localeCompare(strB) : strB.localeCompare(strA)
    }

    // If primary sort values are equal, apply secondary sort
    if (comparison === 0) {
      if (sortKey.value === 'BATTER') {
        // When sorting by batter, secondary sort by pitcher
        const pitcherA = String(a.PITCHER).toLowerCase()
        const pitcherB = String(b.PITCHER).toLowerCase()
        return pitcherA.localeCompare(pitcherB)
      } else {
        // For all other columns, secondary sort by batter
        const batterA = String(a.BATTER).toLowerCase()
        const batterB = String(b.BATTER).toLowerCase()
        return batterA.localeCompare(batterB)
      }
    }

    return comparison
  })

  return sorted
})

const totalPages = computed(() => Math.ceil(sortedData.value.length / itemsPerPage))

const paginatedData = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage
  const end = start + itemsPerPage
  return sortedData.value.slice(start, end)
})

const formatValue = (value, key) => {
  if (key === 'GAME_DATE') {
    return new Date(value).toLocaleDateString()
  }
  if (['LAUNCH_ANGLE', 'EXIT_SPEED', 'HIT_DISTANCE'].includes(key)) {
    return Number(value).toFixed(2)
  }
  if (key === 'VIDEO_LINK') {
    return value ? 'Video' : ''
  }
  return value
}

const sortBy = (key) => {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortOrder.value = 'asc'
  }
}

const handleSearch = () => {
  currentPage.value = 1
}

const handleFilter = () => {
  currentPage.value = 1
}

const handleReset = () => {
  batterQuery.value = ''
  pitcherQuery.value = ''
  selectedOutcome.value = ''
  currentPage.value = 1
}

onMounted(async () => {
  try {
    isLoading.value = true
    const response = await fetch('/data.csv')
    const csvText = await response.text()

    Papa.parse(csvText, {
      header: true,
      complete: (results) => {
        data.value = results.data
          .filter(
            (row) =>
              row.PLAY_OUTCOME && row.PLAY_OUTCOME !== 'Error' && row.PLAY_OUTCOME !== 'Undefined',
          )
          .map((row) => ({
            ...row,
            LAUNCH_ANGLE: parseFloat(row.LAUNCH_ANGLE),
            EXIT_SPEED: parseFloat(row.EXIT_SPEED),
            HIT_DISTANCE: parseFloat(row.HIT_DISTANCE),
            PLAY_OUTCOME: row.PLAY_OUTCOME === 'FieldersChoice' 
              ? "Fielder's Choice" 
              : row.PLAY_OUTCOME === 'HomeRun'
                ? 'Home Run'
                : row.PLAY_OUTCOME
          }))
        isLoading.value = false
      },
      error: (error) => {
        console.error('Error parsing CSV data:', error)
        isLoading.value = false
      },
    })
  } catch (error) {
    console.error('Error loading CSV data:', error)
    isLoading.value = false
  }
})
</script>

<style scoped>
.baseball-table {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
  font-family: 'Roboto', sans-serif;
}

.controls {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  padding: 20px;
  border: 1px solid #ddd;
}

.search {
  display: flex;
  gap: 10px;
}

.search-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.search-label,
.filter-label {
  font-size: 12px;
  font-weight: 800;
  color: #333;
  margin-bottom: 2px;
}

.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 8px;
  width: 16px;
  height: 16px;
  opacity: 0.5;
}

.search input {
  padding: 8px 8px 8px 32px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 200px;
}

.filters {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.filters select {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  min-width: 150px;
}

.table-container {
  overflow-x: auto;
  margin-bottom: 20px;
}

table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  background-color: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

th,
td {
  padding: 12px;
  text-align: left;
  border: 1px solid #ddd;
}

/* Column widths */
.col-batter,
.col-pitcher {
  width: 180px;
}

.col-game_date {
  width: 120px;
}

.col-launch_angle,
.col-exit_speed,
.col-hit_distance {
  width: 120px;
}

.col-play_outcome {
  width: 150px;
}

.col-video_link {
  width: 100px;
  text-align: center;
}

.col-video_link a {
  color: #1a237e;
  text-decoration: none;
  padding: 4px 8px;
  border-radius: 4px;
  background-color: #f8f9fa;
  transition: background-color 0.2s;
}

.col-video_link a:hover {
  background-color: #e9ecef;
  text-decoration: underline;
}

th {
  background-color: #f8f9fa;
  font-weight: 600;
  cursor: pointer;
  border-bottom: 2px solid #ddd;
  white-space: nowrap;
}

td {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

th:first-child {
  border-top-left-radius: 6.5px;
}

th:last-child {
  border-top-right-radius: 6.5px;
}

tr:last-child td:first-child {
  border-bottom-left-radius: 6.5px;
}

tr:last-child td:last-child {
  border-bottom-right-radius: 6.5px;
}

th.sortable:hover {
  background-color: #e9ecef;
}

th.sorted {
  background-color: #e9ecef;
  border: 2px solid #1a237e;
}

.sort-icon {
  margin-left: 5px;
}

tr:hover {
  background-color: #f8f9fa;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}

.pagination-button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: white;
  cursor: pointer;
}

.pagination-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pagination-button:hover:not(:disabled) {
  background-color: #f8f9fa;
}

.page-info {
  font-size: 14px;
  color: #666;
}

.reset-button {
  padding: 9px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #b7b7b7;
  cursor: pointer;
  transition: background-color 0.2s;
  display: flex;
  align-items: center;
  gap: 8px;
}

.reset-icon {
  width: 16px;
  height: 16px;
  opacity: 0.7;
}

.reset-button:hover {
  background-color: #f29b9b;
}

.reset-button:active {
  background-color: #f38787;
}

.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  gap: 16px;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #1a237e;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.loading-container p {
  color: #666;
  font-size: 16px;
  margin: 0;
}

.no-results-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  gap: 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
  padding: 20px;
  margin: 20px 0;
}

.no-results-container p {
  color: #666;
  font-size: 16px;
  margin: 0;
}

.explanation-section {
  background-color: #f8f9fa;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  border: 1px solid #ddd;
  position: relative;
}

.explainer-toggle {
  position: absolute;
  top: 12px;
  right: 12px;
  background: white;
  border: 1px solid #ddd;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
  z-index: 2;
}

.explainer-toggle:hover {
  background-color: #e0e0e0;
}

.explainer-toggle:active {
  background-color: #f29b9b;
}

.explainer-toggle-collapsed {
  background: white;
  border: 1px solid #ddd;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-left: 12px;
}

.explainer-toggle-collapsed:hover {
  background-color: #e0e0e0;
}

.explainer-toggle-collapsed:active {
  background-color: #f29b9b;
}

.explanation-section h2 {
  color: #1a237e;
  font-size: 1.5rem;
  margin: 0 0 12px 0;
  font-weight: 600;
}

.explanation-section p {
  color: #333;
  font-size: 1rem;
  line-height: 1.5;
  margin: 0;
}

.explanation-collapsed {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 20px 12px 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  margin-bottom: 20px;
  border: 1px solid #ddd;
  min-height: 48px;
}

.explanation-collapsed span {
  font-size: 15px;
  font-weight: 600;
  color: #333;
  flex: 1;
  text-align: left;
}

.explanation-collapsed button {
  background: white;
  border: 1px solid #ddd;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-left: 12px;
}

.explanation-collapsed button:hover {
  background-color: #e0e0e0;
}

.explanation-collapsed button:active {
  background-color: #f29b9b;
}

.explainer-list {
  margin: 0 0 0 10px;
  padding: 0 0 0 18px;
  font-size: 1rem;
}

.explainer-list li {
  margin-bottom: 6px;
  line-height: 1.5;
}
</style>
