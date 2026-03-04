<template>
  <div id="app">
    <header>
      <h1>LCSH Subject Heading Browser</h1>
      <div class="search-bar">
        <input
          v-model="searchQuery"
          @keyup.enter="doKeywordSearch()"
          placeholder="Search subject headings (e.g. Censorship--Japan)"
        />
        <button @click="doKeywordSearch()" :disabled="loading">Search</button>
        <button @click="clearAll" class="clear-btn">Clear</button>
      </div>
      <div class="starters">
        <button @click="doKeywordSearch('Vocational education')" class="starter-btn">Vocational education</button>
      </div>
    </header>

    <div class="blocks">
      <div v-for="(block, idx) in blocks" :key="idx" :ref="el => setBlockRef(idx, el)" class="block">

        <!-- KEYWORD SEARCH RESULTS -->
        <template v-if="block.type === 'search-results'">
          <h2 class="block-title">Search: "{{ block.query }}"</h2>
          <p v-if="block.loading" class="loading">Loading...</p>
          <p v-else-if="block.error" class="error">{{ block.error }}</p>
          <ul v-else class="hit-list">
            <li v-for="hit in block.hits" :key="hit.uri" @click="selectHit(hit, block.query)">
              <span v-html="highlightMatch(hit.suggestLabel, block.query)"></span>
            </li>
          </ul>
          <p v-if="!block.loading && !block.error && block.hits.length === 0">No results found.</p>
        </template>

        <!-- SELECTED TERM - PICK CUTOFF -->
        <template v-if="block.type === 'pick-cutoff'">
          <h2 class="block-title">Pick a cutoff point:</h2>
          <div class="cutoff-display">
            <template v-for="(part, pIdx) in block.parts" :key="pIdx">
              <span
                class="cutoff-part"
                :class="{ clickable: pIdx < block.parts.length - 1 }"
                @click="pIdx < block.parts.length - 1 ? pickCutoff(block, pIdx) : null"
              >{{ part }}</span>
              <span v-if="pIdx < block.parts.length - 1" class="separator">--</span>
            </template>
          </div>
          <p class="hint">Click a segment to see everything filed under that prefix.</p>
        </template>

        <!-- HIERARCHY RESULTS (left-anchored) -->
        <template v-if="block.type === 'hierarchy'">
          <h2 class="block-title">{{ block.prefix }}*</h2>
          <p v-if="block.loading" class="loading">Loading...</p>
          <p v-else-if="block.error" class="error">{{ block.error }}</p>
          <ul v-else class="hit-list">
            <li v-for="label in block.hits" :key="label" @click="clickHierarchyTerm(label)">
              {{ label }}
            </li>
          </ul>
          <p v-if="!block.loading && !block.error && block.hits.length === 0">No results found.</p>
          <p v-if="block.truncated" class="truncated">1,000 result limit reached. Narrow your search for more specific results.</p>
        </template>

      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      searchQuery: '',
      blocks: [],
      blockRefMap: {},
      loading: false,
    }
  },
  methods: {
    setBlockRef(idx, el) {
      this.blockRefMap[idx] = el
    },

    async doKeywordSearch(queryOverride) {
      const q = (queryOverride || this.searchQuery || '').trim()
      if (!q) return

      this.loading = true
      this.blocks.push({
        type: 'search-results',
        query: q,
        hits: [],
        loading: true,
        error: null,
      })
      const blockIndex = this.blocks.length - 1
      this.$nextTick(() => this.scrollToBlock(blockIndex))

      try {
        const url = `https://preprod-8080.id.loc.gov/entities/subjects/suggest2/?q=${encodeURIComponent(q)}&searchtype=keyword&count=250`
        const res = await fetch(url)
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        const data = await res.json()
        const hits = data.hits || []
        hits.sort((a, b) => a.suggestLabel.localeCompare(b.suggestLabel))
        this.blocks[blockIndex].hits = hits
      } catch (e) {
        this.blocks[blockIndex].error = 'Failed to fetch results: ' + e.message
      } finally {
        this.blocks[blockIndex].loading = false
        this.loading = false
      }
    },

    selectHit(hit, query) {
      const block = {
        type: 'pick-cutoff',
        fullLabel: hit.suggestLabel,
        query: query,
        parts: hit.suggestLabel.split('--'),
      }
      this.blocks.push(block)
      this.$nextTick(() => this.scrollToBlock(this.blocks.length - 1))
    },

    async pickCutoff(block, partIndex) {
      const prefix = block.parts.slice(0, partIndex + 1).join('--') + '--'

      this.blocks.push({
        type: 'hierarchy',
        prefix: prefix,
        hits: [],
        loading: true,
        error: null,
        truncated: false,
      })
      const blockIndex = this.blocks.length - 1
      this.$nextTick(() => this.scrollToBlock(blockIndex))

      try {
        let allHits = []
        let offset = 1
        let total = Infinity
        const MAX_HITS = 1000

        while (offset <= total) {
          const url = `https://preprod-8080.id.loc.gov/entities/subjects/suggest2/?q=${encodeURIComponent(prefix)}&searchtype=left&rawlist=true&count=250&offset=${offset}`
          const res = await fetch(url)
          if (!res.ok) throw new Error(`HTTP ${res.status}`)
          const data = await res.json()
          total = data.count || 0
          const hits = data.hits || []
          allHits = allHits.concat(hits)
          if (allHits.length >= MAX_HITS) {
            allHits = allHits.slice(0, MAX_HITS)
            this.blocks[blockIndex].truncated = true
            break
          }
          if (hits.length < 250) break
          offset += 250
        }

        this.blocks[blockIndex].hits = allHits
      } catch (e) {
        this.blocks[blockIndex].error = 'Failed to fetch hierarchy: ' + e.message
      } finally {
        this.blocks[blockIndex].loading = false
      }
    },

    clickHierarchyTerm(label) {
      this.searchQuery = label
      this.doKeywordSearch(label)
    },

    highlightMatch(text, query) {
      const escaped = query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
      try {
        const regex = new RegExp(`(${escaped})`, 'gi')
        return text.replace(regex, '<mark>$1</mark>')
      } catch {
        return text
      }
    },

    scrollToBlock(idx) {
      this.$nextTick(() => {
        const el = this.blockRefMap[idx]
        if (el) {
          el.scrollIntoView({ behavior: 'smooth', block: 'start' })
        }
      })
    },

    clearAll() {
      this.blocks = []
      this.blockRefMap = {}
      this.searchQuery = ''
    },
  },
}
</script>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #f5f5f5;
  color: #222;
}

#app {
  max-width: 960px;
  margin: 0 auto;
  padding: 20px;
}

header {
  position: sticky;
  top: 0;
  background: #f5f5f5;
  padding: 16px 0;
  z-index: 10;
  border-bottom: 2px solid #ccc;
  margin-bottom: 20px;
}

h1 {
  font-size: 1.4rem;
  margin-bottom: 12px;
}

.search-bar {
  display: flex;
  gap: 8px;
}

.starters {
  margin-top: 8px;
}

.starter-btn {
  padding: 4px 12px;
  font-size: 0.85rem;
  cursor: pointer;
  border: 1px solid #aaa;
  border-radius: 12px;
  background: #fff;
  color: #333;
}

.starter-btn:hover {
  background: #e8f0fe;
}

.search-bar input {
  flex: 1;
  padding: 10px 14px;
  font-size: 1rem;
  border: 1px solid #aaa;
  border-radius: 4px;
}

.search-bar button {
  padding: 10px 20px;
  font-size: 1rem;
  cursor: pointer;
  border: none;
  border-radius: 4px;
  background: #1a56db;
  color: #fff;
}

.search-bar button:hover {
  background: #1242b0;
}

.search-bar button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.clear-btn {
  background: #666 !important;
}

.clear-btn:hover {
  background: #444 !important;
}

.block {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 20px;
  margin-bottom: 16px;
}

.block-title {
  font-size: 1.1rem;
  margin-bottom: 12px;
  color: #555;
}

.hit-list {
  list-style: none;
  padding: 0;
}

.hit-list li {
  padding: 8px 12px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
  font-size: 1rem;
}

.hit-list li:hover {
  background: #e8f0fe;
}

.hit-list li:last-child {
  border-bottom: none;
}

mark {
  background: #fef08a;
  padding: 0 2px;
  border-radius: 2px;
}

.cutoff-display {
  font-size: 1.6rem;
  font-weight: 600;
  line-height: 2;
  margin: 12px 0;
}

.cutoff-part {
  padding: 4px 8px;
  border-radius: 4px;
}

.cutoff-part.clickable {
  cursor: pointer;
  background: #e8f0fe;
  border: 1px solid #bcd;
}

.cutoff-part.clickable:hover {
  background: #c5dafc;
}

.separator {
  color: #999;
  margin: 0 2px;
}

.hint {
  color: #888;
  font-size: 0.85rem;
  margin-top: 8px;
}

.loading {
  color: #666;
  font-style: italic;
}

.error {
  color: #c00;
}

.truncated {
  margin-top: 12px;
  padding: 8px 12px;
  background: #fff3cd;
  border: 1px solid #ffc107;
  border-radius: 4px;
  color: #856404;
  font-size: 0.9rem;
}
</style>
