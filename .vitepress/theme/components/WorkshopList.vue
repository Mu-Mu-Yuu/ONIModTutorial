<script setup>
import { ref, onMounted } from 'vue'

const mods = ref([])
const loading = ref(true)

const STATS_CACHE_TTL = 60 * 60 * 1000 // 60 分钟

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms))
}

function getCachedStats(id) {
  const raw = localStorage.getItem('workshop_stats_' + id)
  if (!raw) return null

  const { time, data } = JSON.parse(raw)
  if (Date.now() - time > STATS_CACHE_TTL) return null

  return data
}

function setCachedStats(id, data) {
  localStorage.setItem(
    'workshop_stats_' + id,
    JSON.stringify({
      time: Date.now(),
      data
    })
  )
}

async function fetchWorkshopStats(detailUrl) {
  try {
    const match = detailUrl.match(/id=(\d+)/)
    if (!match) return null

    const cleanUrl = `https://steamcommunity.com/sharedfiles/filedetails/?id=${match[1]}`

    const cached = getCachedStats(match[1])
    if (cached) return cached

    const workerUrl =
      `https://aged-dream-7a55.liukele015.workers.dev/?url=${encodeURIComponent(cleanUrl)}`

    const res = await fetch(workerUrl)
    if (!res.ok) {
      console.warn('Worker 返回失败:', res.status, res.statusText)
      return null
    }

    const data = await res.json()
    if (!data || !data.stats) {
      console.warn('Worker 返回数据不正确:', data)
      return null
    }

    setCachedStats(match[1], data.stats)
    return data.stats

  } catch (err) {
    console.warn('统计获取失败:', detailUrl, err)
    return null
  }
}

async function loadStats(mod) {
  if (!mod.link || mod.link === '#') return
  if (mod.stats || mod.statsLoading) return

  mod.statsLoading = true
  mod.stats = await fetchWorkshopStats(mod.link)
  mod.statsLoading = false
}

async function loadStatsSequentially(list) {
  for (const mod of list) {
    await loadStats(mod)
    await sleep(150)
  }
}

onMounted(async () => {
  try {
    const steamUrl =
      'https://steamcommunity.com/workshop/browse/' +
      '?appid=457140' +
      '&browssemethod=trend' +
      '&section=readytouseitems' +
      '&actualsort=trend' +
      '&p=1' +
      '&numperpage=6'

    // 这里使用 Cloudflare做为代理，避免 CORS 问题
    const proxyUrl =
      `https://aged-dream-7a55.liukele015.workers.dev/?mode=workshop&url=${encodeURIComponent(steamUrl)}`

    const res = await fetch(proxyUrl)
    const html = await res.text()

    const doc = new DOMParser().parseFromString(html, 'text/html')
    const items = doc.querySelectorAll('.workshopItem')

    mods.value = Array.from(items).map(item => {
      const titleEl = item.querySelector('.workshopItemTitle')
      const linkEl = item.querySelector('a')
      const authorEl = item.querySelector('.workshopItemAuthorName a')

      const imgDiv = item.querySelector('.workshopItemImg')
      const bg = imgDiv?.style?.backgroundImage || ''
      const bgUrl = bg.match(/url\(["']?(.*?)["']?\)/)?.[1] || null

      const imgPreview = item.querySelector('img.preview_image')
      const previewUrl = imgPreview?.src || null

      const imgAny = item.querySelector('img')
      const anyUrl = imgAny?.src || null

      let thumbnail =
        bgUrl || previewUrl || anyUrl ||
        'https://via.placeholder.com/200x120?text=No+Image'

      if (thumbnail.startsWith('//')) {
        thumbnail = 'https:' + thumbnail
      }

      thumbnail = `https://images.weserv.nl/?url=${encodeURIComponent(thumbnail)}`

      return {
        title: titleEl?.innerText.trim() ?? '未知模组',
        link: linkEl?.href ?? '#',
        author: authorEl?.innerText.trim() ?? '匿名作者',
        thumbnail,

        stats: null,
        statsLoading: false
      }
    })

    loading.value = false
    loadStatsSequentially(mods.value)

  } catch (err) {
    console.error('抓取 Workshop 失败:', err)
    loading.value = false
  }
})
</script>

<template>
  <div class="workshop-container">
    <div v-if="loading" class="loading-box">
      正在连接 Steam 创意工坊...
    </div>

    <div v-else class="mod-grid">
      <a
        v-for="mod in mods"
        :key="mod.link"
        :href="mod.link"
        target="_blank"
        class="mod-card"
      >
        <div class="mod-img">
          <img :src="mod.thumbnail" loading="lazy" />
        </div>

        <div class="mod-card__body">
          <div class="mod-card__title">{{ mod.title }}</div>
          <div class="mod-card__meta">By {{ mod.author }}</div>

          <div class="mod-card__stats">
            <template v-if="mod.stats">
              <span>访客 {{ mod.stats.visitors }}</span>
              <span>订阅 {{ mod.stats.subscribers }}</span>
              <span>收藏 {{ mod.stats.favorites }}</span>
            </template>

            <template v-else-if="mod.statsLoading">
              <span>统计加载中…</span>
            </template>

            <template v-else>
              <span>—</span>
            </template>
          </div>
        </div>

      </a>
    </div>
  </div>
</template>

<style scoped>
.workshop-container {
  margin: 1.5rem 0;
}

.loading-box {
  text-align: center;
  padding: 2rem;
  color: var(--vp-c-text-2);
  background: var(--vp-c-bg-soft);
  border-radius: 8px;
}

.mod-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}

@media (max-width: 900px) {
  .mod-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 600px) {
  .mod-grid {
    grid-template-columns: 1fr;
  }
}

.mod-card {
  background: var(--vp-c-bg-soft);
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  overflow: hidden;
  text-decoration: none !important;
  transition: all 0.25s ease;
}

.mod-card:hover {
  border-color: var(--vp-c-brand);
  transform: translateY(-4px);
  background: var(--vp-c-bg-mute);
}

.mod-img {
  width: 100%;
  height: 125px;
  background: #000;
}

.mod-img img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.mod-card__body {
  padding: 12px;
}

.mod-card__title {
  color: var(--vp-c-text-1);
  font-weight: 600;
  font-size: 0.95rem;
  line-height: 1.3;
  height: 2.6em;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.mod-card__meta {
  color: var(--vp-c-text-2);
  font-size: 0.8rem;
  margin-top: 8px;
}

.mod-card__stats {
  margin-top: 10px;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  font-size: 0.75rem;
  color: var(--vp-c-text-2);
}

.mod-card__stats span {
  padding: 4px 8px;
  border-radius: 999px;
  border: 1px solid var(--vp-c-divider);
  background: var(--vp-c-bg-soft);
}

</style>
