<template>
  <div class="container card-grid" :class="[animationClass, { 'admin-drag': isAdmin }]"
       @dragover="onGridDragOver" @drop="onGridDrop">
    <div v-for="(card, index) in localCards" :key="card.id"
         class="link-item"
         :class="{ dragging: dragId === card.id }"
         :data-id="String(card.id)"
         :style="getCardStyle(index)"
         :draggable="isAdmin"
         @dragstart="onDragStart($event, card)"
         @dragend="onDragEnd">
      <a :href="card.url" target="_blank" :title="getTooltip(card)">
        <img class="link-icon" :src="getLogo(card)" alt="" @error="onImgError($event, card)" loading="lazy">
        <span class="link-text">{{ truncate(card.title) }}</span>
      </a>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, nextTick, onMounted, onUnmounted } from 'vue';

const props = defineProps({
  cards: { type: Array, default: () => [] },
  // ===== 新增：标识所属菜单，用于排序持久化 =====
  menuKey: { type: String, default: '' }
});
const emit = defineEmits(['reorder']);

/* ================= 新增：管理模式 + 拖动排序 ================= */
const isAdmin = ref(!!localStorage.getItem('token'));
const dragId = ref(null);
const localCards = ref([]);

function checkAdmin() { isAdmin.value = !!localStorage.getItem('token'); }
function orderKey() { return 'nav_order_' + (props.menuKey || 'default'); }
function getSavedOrder() {
  try { return JSON.parse(localStorage.getItem(orderKey()) || 'null'); } catch (e) { return null; }
}
function persistOrder() {
  localStorage.setItem(orderKey(), JSON.stringify(localCards.value.map(c => String(c.id))));
  emit('reorder', localCards.value.map((c, i) => ({ id: c.id, sort: i })));
}

watch(() => props.cards, (val) => {
  localCards.value = val ? [...val] : [];
  const saved = getSavedOrder();
  if (saved && Array.isArray(saved) && saved.length === localCards.value.length) {
    const map = new Map(localCards.value.map(c => [String(c.id), c]));
    const ordered = saved.map(id => map.get(String(id))).filter(Boolean);
    if (ordered.length === localCards.value.length) localCards.value = ordered;
  }
}, { deep: true, immediate: true });

function onDragStart(e, card) {
  if (!isAdmin.value) return;
  dragId.value = card.id;
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', String(card.id));
  window.__navCard = { card, menuKey: props.menuKey }; // 供 MenuBar 跨标题移动读取
  requestAnimationFrame(() => e.target.classList.add('dragging'));
}
function onDragEnd() {
  dragId.value = null;
  window.__navCard = null;
}
function onGridDragOver(e) {
  if (!isAdmin.value) return;
  e.preventDefault();
  e.dataTransfer.dropEffect = 'move';
}
function onGridDrop(e) {
  if (!isAdmin.value) return;
  e.preventDefault();
  const cardId = dragId.value || e.dataTransfer.getData('text/plain');
  dragId.value = null;
  if (!cardId || !localCards.value.some(c => String(c.id) === String(cardId))) return;
  const target = e.target.closest('.link-item');
  const list = [...e.currentTarget.querySelectorAll('.link-item')];
  const from = localCards.value.findIndex(c => String(c.id) === String(cardId));
  const to = target ? list.indexOf(target) : localCards.value.length - 1;
  if (from === -1 || to === -1 || from === to) return;
  const arr = [...localCards.value];
  const [moved] = arr.splice(from, 1);
  arr.splice(to, 0, moved);
  localCards.value = arr;
  persistOrder();
}

// 卡片被移动到其它标题后，本网格立即移除它
function onCardMoved(e) {
  const cardId = e.detail && e.detail.cardId;
  if (!cardId) return;
  if (localCards.value.some(c => String(c.id) === String(cardId))) {
    localCards.value = localCards.value.filter(c => String(c.id) !== String(cardId));
    persistOrder();
  }
}
onMounted(() => {
  checkAdmin();
  window.addEventListener('nav:card-moved', onCardMoved);
  window.addEventListener('auth:unauthorized', checkAdmin);
  window.addEventListener('storage', checkAdmin);
});
onUnmounted(() => {
  window.removeEventListener('nav:card-moved', onCardMoved);
  window.removeEventListener('auth:unauthorized', checkAdmin);
  window.removeEventListener('storage', checkAdmin);
});
/* ================= 新增结束 ================= */

/* ================= 原代码：动画 ================= */
const animationClass = ref('');
const animationType = ref('slideUp');

watch(() => props.cards, (newCards, oldCards) => {
  if (newCards && newCards.length > 0) {
    const isDataChanged = !oldCards || oldCards.length === 0 || JSON.stringify(newCards) !== JSON.stringify(oldCards);
    if (isDataChanged) {
      nextTick(() => { triggerAnimation(); });
    }
  }
}, { deep: true, immediate: false });

function triggerAnimation() {
  const animations = ['slideUp', 'radial', 'fadeIn', 'slideLeft', 'slideRight', 'convergeIn', 'flipIn'];
  const randomIndex = Math.floor(Math.random() * animations.length);
  animationType.value = animations[randomIndex];
  animationClass.value = `animate-${animationType.value}`;
  setTimeout(() => { animationClass.value = ''; }, 1200);
}

function getCardStyle(index) {
  if (!animationClass.value) return {};
  const isMobile = window.innerWidth <= 480;
  if (isMobile) return { animationDelay: '0s' };
  if (animationType.value === 'slideUp') {
    return { animationDelay: `${index * 0.05}s` };
  } else if (animationType.value === 'radial') {
    const cols = window.innerWidth <= 768 ? 3 : (window.innerWidth <= 1200 ? 4 : 6);
    const row = Math.floor(index / cols);
    const col = index % cols;
    const centerCol = Math.floor(cols / 2);
    const distance = Math.abs(col - centerCol) + row;
    return { animationDelay: `${distance * 0.08}s` };
  } else if (animationType.value === 'fadeIn') {
    return { animationDelay: `${Math.random() * 0.5}s` };
  } else if (animationType.value === 'slideLeft') {
    const cols = window.innerWidth <= 768 ? 3 : (window.innerWidth <= 1200 ? 4 : 6);
    const row = Math.floor(index / cols);
    return { animationDelay: `${row * 0.1}s` };
  } else if (animationType.value === 'slideRight') {
    const cols = window.innerWidth <= 768 ? 3 : (window.innerWidth <= 1200 ? 4 : 6);
    const row = Math.floor(index / cols);
    const col = index % cols;
    return { animationDelay: `${(row + (cols - col - 1) * 0.02) * 0.08}s` };
  } else if (animationType.value === 'convergeIn') {
    const cols = window.innerWidth <= 768 ? 3 : (window.innerWidth <= 1200 ? 4 : 6);
    const col = index % cols;
    const centerCol = Math.floor(cols / 2);
    const distanceFromCenter = Math.abs(col - centerCol);
    return { animationDelay: `${(cols - distanceFromCenter - 1) * 0.08}s` };
  } else if (animationType.value === 'flipIn') {
    const cols = window.innerWidth <= 768 ? 3 : (window.innerWidth <= 1200 ? 4 : 6);
    const row = Math.floor(index / cols);
    const col = index % cols;
    return { animationDelay: `${(row + col) * 0.06}s` };
  }
  return {};
}

function getLogo(card) {
  if (card.custom_logo_path) return 'http://localhost:3000/uploads/' + card.custom_logo_path;
  if (card.logo_url) return card.logo_url;
  try {
    const url = new URL(card.url);
    return url.origin + '/favicon.ico';
  } catch {
    return '/default-favicon.png';
  }
}
function onImgError(e, card) { e.target.src = '/default-favicon.png'; }
function getTooltip(card) {
  let tip = '';
  if (card.desc) tip += card.desc + '\n';
  tip += card.url;
  return tip;
}
function truncate(str) {
  if (!str) return '';
  return str.length > 20 ? str.slice(0, 20) + '...' : str;
}
</script>

<style scoped>
.container {
  max-width: 55rem;
  margin: 0 auto;
  width: 100%;
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 15px;
  opacity: 1;
  transition: opacity 0.2s ease;
}
@media (max-width: 1200px) { .container { grid-template-columns: repeat(4, 1fr); } }
@media (max-width: 768px) { .container { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 480px) { .container { grid-template-columns: repeat(3, 1fr); } }
.link-item {
  background-color: rgba(255, 255, 255, 0.15);
  border-radius: 15px;
  padding: 0;
  transition: all 0.2s;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  text-align: center;
  min-height: 85px;
  height: 85px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.link-item:hover {
  background-color: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.15);
}
.link-item a {
  text-decoration: none;
  color: #ffffff;
  font-weight: 500;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 0;
  box-sizing: border-box;
}
.link-icon {
  width: 25px;
  height: 25px;
  margin: 4px auto;
  object-fit: contain;
}
.link-text {
  padding-right: 4px;
  padding-left: 4px;
  font-size: 14px;
  text-align: center;
  word-break: break-all;
  max-width: 100%;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: normal;
  line-height: 1;
  min-height: 1.5em;
}

/* ===== 新增：管理拖动样式 ===== */
.admin-drag .link-item { cursor: grab; }
.admin-drag .link-item:active { cursor: grabbing; }
.link-item.dragging { opacity: .35; border: 2px dashed #399dff; }

/* ===== 以下为原有动画样式（原样保留） ===== */
.animate-slideUp .link-item { animation: slideUpIn 0.6s ease-out forwards; opacity: 0; transform: translateY(30px); }
@keyframes slideUpIn { 0% { opacity: 0; transform: translateY(30px); } 100% { opacity: 1; transform: translateY(0); } }
.animate-radial .link-item { animation: radialIn 0.5s ease-out forwards; opacity: 0; transform: scale(0.3); }
@keyframes radialIn { 0% { opacity: 0; transform: scale(0.3); } 50% { opacity: 1; transform: scale(1.1); } 100% { opacity: 1; transform: scale(1); } }
.animate-fadeIn .link-item { animation: fadeIn 0.6s ease-out forwards; opacity: 0; }
@keyframes fadeIn { 0% { opacity: 0; transform: translateY(10px); } 100% { opacity: 1; transform: translateY(0); } }
.animate-slideLeft .link-item { animation: slideLeftIn 0.6s ease-out forwards; opacity: 0; transform: translateX(-50px); }
@keyframes slideLeftIn { 0% { opacity: 0; transform: translateX(-50px); } 100% { opacity: 1; transform: translateX(0); } }
.animate-slideRight .link-item { animation: slideRightIn 0.6s ease-out forwards; opacity: 0; transform: translateX(50px); }
@keyframes slideRightIn { 0% { opacity: 0; transform: translateX(50px); } 100% { opacity: 1; transform: translateX(0); } }
.animate-convergeIn .link-item { animation: convergeIn 0.7s ease-out forwards; opacity: 0; }
.animate-convergeIn .link-item:nth-child(6n+1),
.animate-convergeIn .link-item:nth-child(6n+6) { transform: translateX(-80px); }
.animate-convergeIn .link-item:nth-child(6n+2),
.animate-convergeIn .link-item:nth-child(6n+5) { transform: translateX(-40px); }
.animate-convergeIn .link-item:nth-child(6n+3),
.animate-convergeIn .link-item:nth-child(6n+4) { transform: translateY(-30px); }
@media (max-width: 1200px) and (min-width: 769px) {
  .animate-convergeIn .link-item:nth-child(4n+1),
  .animate-convergeIn .link-item:nth-child(4n+4) { transform: translateX(-60px); }
  .animate-convergeIn .link-item:nth-child(4n+2),
  .animate-convergeIn .link-item:nth-child(4n+3) { transform: translateY(-30px); }
}
@media (max-width: 768px) {
  .animate-convergeIn .link-item:nth-child(3n+1),
  .animate-convergeIn .link-item:nth-child(3n+3) { transform: translateX(-50px); }
  .animate-convergeIn .link-item:nth-child(3n+2) { transform: translateY(-30px); }
}
@keyframes convergeIn { 0% { opacity: 0; } 100% { opacity: 1; transform: translate(0, 0); } }
.animate-flipIn .link-item { animation: flipIn 0.7s ease-out forwards; opacity: 0; transform: rotateY(-90deg); }
@keyframes flipIn {
  0% { opacity: 0; transform: rotateY(-90deg); }
  50% { opacity: 1; transform: rotateY(-45deg); }
  100% { opacity: 1; transform: rotateY(0deg); }
}
.container:not(.animate-slideUp):not(.animate-radial):not(.animate-fadeIn) .link-item {
  opacity: 1;
  transform: translateY(0) scale(1);
}
@media (max-width: 768px) {
  .animate-slideUp .link-item { animation-duration: 0.4s; }
  .animate-radial .link-item { animation-duration: 0.4s; }
}
@media (max-width: 480px) {
  .animate-slideUp .link-item { animation-delay: 0s !important; }
  .animate-radial .link-item { animation-delay: 0s !important; }
}
@media (prefers-reduced-motion: reduce) {
  .animate-slideUp .link-item,
  .animate-radial .link-item {
    animation: none;
    opacity: 1;
    transform: none;
  }
}
</style>
