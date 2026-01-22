<template>
  <div class="timeline">
    <div class="timeline-item" v-for="(item, index) in steps" :key="index">
      <div class="timeline-dot"></div>
      <div class="timeline-content">
        <div class="timeline-date">{{ item.status }}</div>
        <h4 class="timeline-title">{{ item.title }}</h4>
        <p class="timeline-desc">{{ item.desc }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
const steps = [
  { status: '已完成', title: '摸鱼', desc: '了解上班摸鱼写文档的流程。' },
  { status: '进行中', title: 'Mod 基础教程', desc: '掌握变量、类与对象，这是制作 Mod 的基石。' },
  { status: '计划中', title: 'UI 界面开发', desc: '教你如何在游戏里画出自己的窗口和按钮。' }
]
</script>

<style scoped>
.timeline { margin: 2rem 0; padding-left: 20px; border-left: 2px solid var(--vp-c-divider); }
.timeline-item { position: relative; margin-bottom: 2rem; padding-left: 30px; }
.timeline-dot {
  position: absolute; left: -31px; top: 5px;
  width: 20px; height: 20px;
  background: var(--vp-c-brand);
  border: 4px solid var(--vp-c-bg);
  border-radius: 50%;
}
.timeline-date { font-size: 0.8rem; color: var(--vp-c-brand); font-weight: bold; }
.timeline-title { margin: 5px 0 !important; color: var(--vp-c-text-1); }
.timeline-desc { font-size: 0.9rem; color: var(--vp-c-text-2); margin: 0; }
</style>