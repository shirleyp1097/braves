<template>
  <div class="baseball-table">
    <div class="controls">
      <div class="search">
        <input
          v-model="batterQuery"
          type="text"
          placeholder="Filter by Batter"
          @input="handleSearch"
        />
        <input
          v-model="pitcherQuery"
          type="text"
          placeholder="Filter by Pitcher"
          @input="handleSearch"
        />
      </div>
      <div class="filters">
        <select v-model="selectedOutcome" @change="handleFilter">
          <option value="">All Outcomes</option>
          <option v-for="outcome in uniqueOutcomes" :key="outcome" :value="outcome">
            {{ outcome }}
          </option>
        </select>
      </div>
      <div>
        <button class="reset-button" @click="handleReset">Reset Filters</button>
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
    </div>

    <div class="table-container">
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
              <span v-if="sortKey === column.key && column.key !== 'VIDEO_LINK'" class="sort-icon">
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
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import Papa from 'papaparse'

const data = ref([])
const batterQuery = ref('')
const pitcherQuery = ref('')
const sortKey = ref('GAME_DATE')
const sortOrder = ref('desc')
const currentPage = ref(1)
const itemsPerPage = 20
const selectedOutcome = ref('')

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

const uniqueOutcomes = computed(() => {
  const outcomes = new Set(data.value.map((row) => row.PLAY_OUTCOME))
  return Array.from(outcomes).sort()
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
          }))
      },
    })
  } catch (error) {
    console.error('Error loading CSV data:', error)
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
  gap: 20px;
  margin-bottom: 20px;
}

.search {
  display: flex;
  gap: 10px;
}

.search input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 200px;
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
  padding: 8px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

.reset-button:hover {
  background-color: #f8f9fa;
}

.reset-button:active {
  background-color: #e9ecef;
}
</style>
