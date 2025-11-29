<template>
  <main>
    <div class="button-bar">
      <button
        @click="pickScheme"
        :disabled="!schemes"
      >
        Pick
      </button>
      <button
        @click="resetIndex"
        :disabled="!schemes"
      >
        Reset
      </button>
      <button
        @click="fetchScheme"
      >
        Fetch
      </button>
      <div class="checkbox-group">
        <label>
          <input
            type="checkbox"
            v-model="queryOptions.japaneseOnly"
          />
          Japanese Only
        </label>
      </div>
    </div>
    <div class="container">
      <div class="left-panel">
        <div v-if="fetchedData && schemes && index >= 0">
          <img
            :src="schemes[index]?.image_uris.normal"
            alt="Scheme Card"
          />
        </div>
        <div v-else-if="fetchedData">
          <p>Total Cards: {{ fetchedData.total_cards }}</p>
        </div>
        <div v-else>
          <p>Loading...</p>
        </div>
      </div>
      <div class="right-panel">
        <p>Ongoing Schemes:</p>
        <div class="scrollable-list">
          <ul>
            <li
              v-for="(s, i) in ongoingSchemes"
              :key="i"
            >
              <img
                :src="s.image_uris.small"
                alt="Ongoing Scheme Card"
                @click="activateOngoingScheme(s)"
              />
            </li>
          </ul>
        </div>
      </div>
    </div>
  </main>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'

const API_URL = 'https://api.scryfall.com'
const BASE_QUERY = 'type:scheme'

type TypeLine = '計略' | '持続計略' | 'Scheme' | 'Ongoing Scheme'

interface Scheme {
  image_uris: {
    small: string
    normal: string
  },
  type_line: TypeLine
}

interface ScryfallListResponse {
  total_cards: number
  has_more: boolean
  next_page?: string
  data: Scheme[]
}

const fetchedData = ref<ScryfallListResponse | null>(null)
const schemes = ref<Scheme[]>([])
const index = ref<number>(-1)
const ongoingSchemes = ref<Scheme[]>([])

interface QueryOptions {
  japaneseOnly: boolean
}

const queryOptions = ref<QueryOptions>({
  japaneseOnly: true,
})

const fetchScheme = async () => {
  try {
    let query = BASE_QUERY
    if (queryOptions.value.japaneseOnly) {
      query += ' lang:ja'
    }

    const res = await fetch(`${API_URL}/cards/search?q=${encodeURI(query)}`)
    fetchedData.value = await res.json() as ScryfallListResponse
    
    schemes.value = randomizeScheme(fetchedData.value?.data || [])
    index.value = -1
    ongoingSchemes.value = []
  } catch (error) {
    console.error('Error fetching data:', error)
    return
  }
}

const randomizeScheme = (data: Scheme[]): Scheme[] => {
  return data
    .map(value => ({ value, sort: Math.random() }))
    .sort((a, b) => a.sort - b.sort)
    .map(({ value }) => value)
}

onMounted(async () => {
  await fetchScheme()
})

const pickScheme = () => {
  if (schemes.value && index.value !== null) {
    if (schemes.value[index.value]?.type_line === 'Ongoing Scheme' || schemes.value[index.value]?.type_line === '持続計略') {
      schemes.value.splice(index.value, 1)
      index.value = index.value >= schemes.value.length ? -1 : index.value - 1
    }

    index.value = (index.value + 1) % schemes.value.length

    if (schemes.value[index.value]?.type_line === 'Ongoing Scheme' || schemes.value[index.value]?.type_line === '持続計略') {
      ongoingSchemes.value.push(schemes.value[index.value] as Scheme)
    }
  }
}

const resetIndex = () => {
  if (fetchedData.value) {
    randomizeScheme(fetchedData?.value.data)
  }
  index.value = -1
  ongoingSchemes.value = []
}

const activateOngoingScheme = (scheme: Scheme) => {
  if (schemes.value.includes(scheme)) {
    return
  }

  schemes.value.splice(index.value, 0, scheme)
  index.value = (index.value + 1) % schemes.value.length
  const schemeIndex = ongoingSchemes.value.indexOf(scheme)
  if (schemeIndex > -1) {
    ongoingSchemes.value.splice(schemeIndex, 1)
  }
}
</script>
