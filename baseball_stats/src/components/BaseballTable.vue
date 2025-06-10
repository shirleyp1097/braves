<template>
  <div class="baseball-table">
    <div class="controls">
      <div class="search">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search..."
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
    </div>

    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th
              v-for="column in columns"
              :key="column.key"
              @click="sortBy(column.key)"
              :class="{ sortable: true, sorted: sortKey === column.key }"
            >
              {{ column.label }}
              <span v-if="sortKey === column.key" class="sort-icon">
                {{ sortOrder === 'asc' ? '↑' : '↓' }}
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in paginatedData" :key="row.BATTER_ID + row.GAME_DATE">
            <td v-for="column in columns" :key="column.key">
              {{ formatValue(row[column.key], column.key) }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="pagination">
      <button
        :disabled="currentPage === 1"
        @click="currentPage--"
        class="pagination-button"
      >
        Previous
      </button>
      <span class="page-info">
        Page {{ currentPage }} of {{ totalPages }}
      </span>
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
const searchQuery = ref('')
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
  { key: 'PLAY_OUTCOME', label: 'Outcome' }
]

const uniqueOutcomes = computed(() => {
  const outcomes = new Set(data.value.map(row => row.PLAY_OUTCOME))
  return Array.from(outcomes).sort()
})

const filteredData = computed(() => {
  let result = data.value

  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(row =>
      Object.values(row).some(value =>
        String(value).toLowerCase().includes(query)
      )
    )
  }

  if (selectedOutcome.value) {
    result = result.filter(row => row.PLAY_OUTCOME === selectedOutcome.value)
  }

  return result
})

const sortedData = computed(() => {
  const sorted = [...filteredData.value]
  sorted.sort((a, b) => {
    const aValue = a[sortKey.value]
    const bValue = b[sortKey.value]

    if (sortKey.value === 'GAME_DATE') {
      return sortOrder.value === 'asc'
        ? new Date(aValue) - new Date(bValue)
        : new Date(bValue) - new Date(aValue)
    }

    if (typeof aValue === 'number' && typeof bValue === 'number') {
      return sortOrder.value === 'asc' ? aValue - bValue : bValue - aValue
    }

    return sortOrder.value === 'asc'
      ? String(aValue).localeCompare(String(bValue))
      : String(bValue).localeCompare(String(aValue))
  })

  return sorted
})

const totalPages = computed(() =>
  Math.ceil(sortedData.value.length / itemsPerPage)
)

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

onMounted(async () => {
  try {
    const response = await fetch('/data.csv')
    const csvText = await response.text()
    
    Papa.parse(csvText, {
      header: true,
      complete: (results) => {
        data.value = results.data
          .filter(row => row.PLAY_OUTCOME && row.PLAY_OUTCOME !== 'Error' && row.PLAY_OUTCOME !== 'Undefined')
          .map(row => ({
            ...row,
            LAUNCH_ANGLE: parseFloat(row.LAUNCH_ANGLE),
            EXIT_SPEED: parseFloat(row.EXIT_SPEED),
            HIT_DISTANCE: parseFloat(row.HIT_DISTANCE)
          }))
        console.log('Filtered data loaded:', data.value.length, 'rows')
      }
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
}

.controls {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
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
  border-collapse: collapse;
  background-color: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

th, td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

th {
  background-color: #f8f9fa;
  font-weight: 600;
  cursor: pointer;
}

th.sortable:hover {
  background-color: #e9ecef;
}

th.sorted {
  background-color: #e9ecef;
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
</style> 