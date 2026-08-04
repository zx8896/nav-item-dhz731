<template>
  <div class="wc-bar">
    <!-- 左下角：IP 归属地 + 天气 -->
    <div class="wc-weather" @click="loadWeather" title="点击刷新天气">
      <span class="wc-icon">{{ wIcon }}</span>
      <div class="wc-info">
        <div class="wc-loc">{{ wLoc }}</div>
        <div class="wc-temp">{{ wTemp }}°C <span class="wc-desc">{{ wDesc }}</span></div>
      </div>
    </div>
    <!-- 右下角：日期时间 + 春节倒计时 -->
    <div class="wc-clock">
      <div class="wc-date">{{ dateStr }}</div>
      <div class="wc-time">{{ timeStr }}</div>
      <div class="wc-cny">{{ cnyStr }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const wIcon = ref('🌤️');
const wLoc = ref('正在定位…');
const wTemp = ref('--');
const wDesc = ref('');
const dateStr = ref('');
const timeStr = ref('');
const cnyStr = ref('');

// 已收录 2026~2035 年春节（正月初一）日期
const CNY_DATES = [
  '2026-02-17','2027-02-06','2028-01-26','2029-02-13','2030-02-03',
  '2031-01-23','2032-02-11','2033-01-31','2034-02-19','2035-02-08'
];

const WEATHER_MAP = {
  0:['☀️','晴'],1:['🌤️','晴间多云'],2:['⛅','多云'],3:['☁️','阴'],
  45:['🌫️','雾'],48:['🌫️','雾凇'],
  51:['🌦️','毛毛雨'],53:['🌦️','毛毛雨'],55:['🌧️','毛毛雨'],56:['🌧️','冻毛毛雨'],57:['🌧️','冻毛毛雨'],
  61:['🌧️','小雨'],63:['🌧️','中雨'],65:['🌧️','大雨'],66:['🌧️','冻雨'],67:['🌧️','冻雨'],
  71:['🌨️','小雪'],73:['🌨️','中雪'],75:['❄️','大雪'],77:['❄️','雪粒'],
  80:['🌦️','阵雨'],81:['🌧️','强阵雨'],82:['⛈️','暴雨'],
  85:['🌨️','阵雪'],86:['🌨️','强阵雪'],95:['⛈️','雷阵雨'],96:['⛈️','雷阵雨伴冰雹'],99:['⛈️','强雷暴']
};

async function getLocation() {
  const apis = [
    async () => { const r = await fetch('https://ipapi.co/json/'); const d = await r.json();
      return { city: d.city, region: d.region, lat: d.latitude, lon: d.longitude }; },
    async () => { const r = await fetch('https://api.ip.sb/geoip'); const d = await r.json();
      return { city: d.city, region: d.region, lat: d.latitude, lon: d.longitude }; },
    async () => { const r = await fetch('https://ipinfo.io/json'); const d = await r.json();
      const [lat, lon] = (d.loc || '39.9,116.4').split(',').map(Number);
      return { city: d.city, region: d.region, lat, lon }; }
  ];
  for (const api of apis) {
    try { const loc = await api(); if (loc && loc.lat && loc.lon) return loc; } catch (e) {}
  }
  return { city: '北京', region: '北京', lat: 39.9042, lon: 116.4074 };
}

async function loadWeather() {
  try {
    const loc = await getLocation();
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${loc.lat}&longitude=${loc.lon}` +
      `&current=temperature_2m,relative_humidity_2m,weather_code&daily=temperature_2m_max,temperature_2m_min` +
      `&timezone=auto&forecast_days=1`;
    const res = await fetch(url);
    if (!res.ok) throw new Error();
    const d = await res.json();
    const cur = d.current;
    const [icon, desc] = WEATHER_MAP[cur.weather_code] || ['🌡️', '未知'];
    const max = d.daily.temperature_2m_max[0], min = d.daily.temperature_2m_min[0];
    wIcon.value = icon;
    wLoc.value = (loc.region && loc.region !== loc.city) ? `${loc.city} · ${loc.region}` : (loc.city || '未知');
    wTemp.value = Math.round(cur.temperature_2m);
    wDesc.value = `${desc} ${Math.round(min)}~${Math.round(max)}° 湿度${cur.relative_humidity_2m}%`;
  } catch (e) {
    wLoc.value = '天气获取失败，点击重试';
    wTemp.value = '--';
    wDesc.value = '';
  }
}

function tick() {
  const now = new Date();
  const weeks = ['日','一','二','三','四','五','六'];
  dateStr.value = `${now.getFullYear()}年${now.getMonth()+1}月${now.getDate()}日 星期${weeks[now.getDay()]}`;
  const p = n => String(n).padStart(2, '0');
  timeStr.value = `${p(now.getHours())}:${p(now.getMinutes())}:${p(now.getSeconds())}`;
  // 春节倒计时
  let next = null;
  for (const d of CNY_DATES) {
    const t = new Date(d + 'T00:00:00').getTime();
    if (t > now.getTime()) { next = { year: d.slice(0, 4), time: t }; break; }
  }
  if (!next) { cnyStr.value = '🧧 春节日期未收录'; return; }
  const diff = next.time - now.getTime();
  if (diff <= 0) { cnyStr.value = '🧧 今天就是春节！恭喜发财 🎉'; return; }
  const days = Math.floor(diff / 86400000);
  const hours = Math.floor(diff % 86400000 / 3600000);
  const mins = Math.floor(diff % 3600000 / 60000);
  const secs = Math.floor(diff % 60000 / 1000);
  cnyStr.value = `🧧 距${next.year}年春节 ${days}天 ${hours}时 ${mins}分 ${secs}秒`;
}

let timer = null;
onMounted(() => {
  tick();
  timer = setInterval(tick, 1000);
  loadWeather();
});
onUnmounted(() => { if (timer) clearInterval(timer); });
</script>

<style scoped>
.wc-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 9999;
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  padding: 0 14px calc(10px + env(safe-area-inset-bottom));
  pointer-events: none;   /* 容器不挡点击，只有两个小卡片可点 */
  font-family: -apple-system, "Segoe UI", "Microsoft YaHei", sans-serif;
}
.wc-weather {
  pointer-events: auto;
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(0, 0, 0, .38);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,.14);
  border-radius: 12px;
  padding: 6px 11px;
  color: #fff;
  cursor: pointer;
  user-select: none;
  transition: .2s;
  max-width: 46vw;
}
.wc-weather:hover { background: rgba(0,0,0,.55); }
.wc-icon { font-size: 24px; line-height: 1; flex-shrink: 0; }
.wc-loc { font-size: 12px; color: rgba(255,255,255,.78); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.wc-temp { font-size: 15px; font-weight: 700; line-height: 1.2; white-space: nowrap; }
.wc-desc { font-size: 11px; font-weight: 400; color: rgba(255,255,255,.7); white-space: nowrap; }
.wc-clock {
  pointer-events: auto;
  text-align: right;
  background: rgba(0, 0, 0, .38);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,.14);
  border-radius: 12px;
  padding: 6px 11px;
  color: #fff;
  user-select: none;
  max-width: 52vw;
}
.wc-date { font-size: 12px; color: rgba(255,255,255,.78); white-space: nowrap; }
.wc-time { font-size: 19px; font-weight: 800; letter-spacing: 1px; font-variant-numeric: tabular-nums; line-height: 1.2; white-space: nowrap; }
.wc-cny { font-size: 11px; color: #ffd54d; font-weight: 600; margin-top: 1px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

/* 手机适配：整体缩小，贴住屏幕底部边缘 */
@media (max-width: 768px) {
  .wc-bar { padding: 0 6px calc(6px + env(safe-area-inset-bottom)); }
  .wc-weather { gap: 6px; padding: 5px 8px; border-radius: 10px; }
  .wc-icon { font-size: 18px; }
  .wc-loc { font-size: 10px; }
  .wc-temp { font-size: 13px; }
  .wc-desc { font-size: 10px; }
  .wc-clock { padding: 5px 8px; border-radius: 10px; }
  .wc-date { font-size: 10px; }
  .wc-time { font-size: 14px; }
  .wc-cny { font-size: 10px; }
}
</style>
