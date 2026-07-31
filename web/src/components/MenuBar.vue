<template>
  <nav class="menu-bar">
    <div
      v-for="menu in menus"
      :key="menu.id"
      class="menu-item"
      :class="{ 'drop-zone': isAdmin && dropMenuId === menu.id }"
      @mouseenter="showSubMenu(menu.id)"
      @mouseleave="hideSubMenu(menu.id)"
      @dragover.prevent="onMenuDragOver(menu, null)"
      @dragleave="onMenuDragLeave(menu, null)"
      @drop.prevent="onMenuDrop(menu, null)"
    >
      <button
        @click="$emit('select', menu)"
        :class="{active: menu.id === activeId}"
      >
        {{ menu.name }}
      </button>

      <!-- 二级菜单 -->
      <div
        v-if="menu.subMenus && menu.subMenus.length > 0"
        class="sub-menu"
        :class="{ 'show': hoveredMenuId === menu.id }"
      >
        <button
          v-for="subMenu in menu.subMenus"
          :key="subMenu.id"
          @click="$emit('select', subMenu, menu)"
          :class="{active: subMenu.id === activeSubMenuId,
                   'drop-zone-sub': isAdmin && dropMenuId === menu.id && dropSubMenuId === subMenu.id}"
          class="sub-menu-item"
          @dragover.prevent="onMenuDragOver(menu, subMenu)"
          @dragleave="onMenuDragLeave(menu, subMenu)"
          @drop.prevent="onMenuDrop(menu, subMenu)"
        >
          {{ subMenu.name }}
        </button>
      </div>
    </div>
  </nav>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { updateCard } from '../api.js';

const props = defineProps({
  menus: Array,
  activeId: Number,
  activeSubMenuId: Number
});
const emit = defineEmits(['select', 'card-moved']);

/* ================= 新增：管理模式下标题可接收卡片 ================= */
const isAdmin = ref(!!localStorage.getItem('token'));
const dropMenuId = ref(null);
const dropSubMenuId = ref(null);

function checkAdmin() { isAdmin.value = !!localStorage.getItem('token'); }

function onMenuDragOver(menu, sub) {
  if (!isAdmin.value) return;
  dropMenuId.value = menu.id;
  dropSubMenuId.value = sub ? sub.id : null;
}
function onMenuDragLeave(menu, sub) {
  if (dropMenuId.value === menu.id && (!sub || dropSubMenuId.value === sub.id)) {
    dropMenuId.value = null;
    dropSubMenuId.value = null;
  }
}
async function onMenuDrop(menu, sub) {
  if (!isAdmin.value) return;
  dropMenuId.value = null;
  dropSubMenuId.value = null;
  const payload = window.__navCard;
  if (!payload || !payload.card) { showToast('⚠️ 请先拖动一个网站卡片到标题上'); return; }
  const { card } = payload;
  const newSubId = sub ? sub.id : null;
  try {
    const data = { ...card };
    delete data.id; // id 走 URL 参数
    await updateCard(card.id, { ...data, menuId: menu.id, subMenuId: newSubId });
    // 通知各网格刷新（移除已移动的卡片）
    window.dispatchEvent(new CustomEvent('nav:card-moved', {
      detail: { cardId: card.id, menuId: menu.id, subMenuId: newSubId }
    }));
    emit('card-moved', { card, menuId: menu.id, subMenuId: newSubId });
    showToast(`✅ 已移动到「${menu.name}${sub ? ' / ' + sub.name : ''}」`);
  } catch (err) {
    showToast('❌ 移动失败：' + (err.response?.data?.error || err.response?.data?.message || err.message));
  }
}
function showToast(msg) {
  let el = document.querySelector('.nav-toast');
  if (!el) {
    el = document.createElement('div');
    el.style.cssText = 'position:fixed;bottom:36px;left:50%;transform:translateX(-50%) translateY(80px);' +
      'background:#111827;color:#fff;padding:10px 22px;border-radius:99px;font-size:14px;z-index:99999;' +
      'opacity:0;transition:.3s;box-shadow:0 8px 24px rgba(0,0,0,.3);font-family:-apple-system,"Microsoft YaHei",sans-serif;';
    el.className = 'nav-toast';
    document.body.appendChild(el);
  }
  el.textContent = msg;
  el.style.opacity = '1';
  el.style.transform = 'translateX(-50%) translateY(0)';
  clearTimeout(el._t);
  el._t = setTimeout(() => {
    el.style.opacity = '0';
    el.style.transform = 'translateX(-50%) translateY(80px)';
  }, 2200);
}
onMounted(() => {
  checkAdmin();
  window.addEventListener('auth:unauthorized', checkAdmin);
  window.addEventListener('storage', checkAdmin);
});
onUnmounted(() => {
  window.removeEventListener('auth:unauthorized', checkAdmin);
  window.removeEventListener('storage', checkAdmin);
});
/* ================= 新增结束 ================= */

const hoveredMenuId = ref(null);

function showSubMenu(menuId) {
  hoveredMenuId.value = menuId;
}
function hideSubMenu(menuId) {
  setTimeout(() => {
    if (hoveredMenuId.value === menuId) {
      hoveredMenuId.value = null;
    }
  }, 100);
}
</script>

<style scoped>
.menu-bar {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  padding: 0 1rem;
  position: relative;
}
.menu-item { position: relative; }
.menu-bar button {
  background: transparent;
  border: none;
  color: #fff;
  font-size: 16px;
  font-weight: 500;
  padding: 0.8rem 2rem;
  cursor: pointer;
  transition: all 0.3s ease;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  box-shadow: none;
  border-radius: 8px;
  position: relative;
  overflow: hidden;
}
.menu-bar button::before {
  content: '';
  position: absolute;
  bottom: 0;
  left: 50%;
  width: 0;
  height: 2px;
  background: #399dff;
  transition: all 0.3s ease;
  transform: translateX(-50%);
}
.menu-bar button:hover {
  color: #399dff;
  transform: translateY(-1px);
}
.menu-bar button.active { color: #399dff; }
.menu-bar button.active::before { width: 60%; }

/* ===== 新增：拖放高亮 ===== */
.menu-item.drop-zone > button {
  background: rgba(57, 157, 255, 0.3) !important;
  outline: 2px dashed #399dff;
  outline-offset: -2px;
}
.sub-menu-item.drop-zone-sub {
  background: rgba(57, 157, 255, 0.35) !important;
  color: #399dff !important;
}

/* ===== 以下为原有二级菜单样式（原样保留） ===== */
.sub-menu {
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: #5c595900;
  backdrop-filter: blur(8px);
  border-radius: 6px;
  min-width: 120px;
  opacity: 0;
  visibility: hidden;
  transition: all 0.2s ease;
  z-index: 1000;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.15);
  margin-top: -2px;
}
.sub-menu.show {
  opacity: 1;
  visibility: visible;
  transform: translateX(-50%) translateY(2px);
}
.sub-menu-item {
  display: block !important;
  width: 100% !important;
  text-align: center !important;
  padding: 0.4rem 1rem !important;
  border: none !important;
  background: transparent !important;
  color: #fff !important;
  font-size: 14px !important;
  font-weight: 400 !important;
  cursor: pointer !important;
  transition: all 0.2s ease !important;
  border-radius: 0 !important;
  text-shadow: none !important;
  line-height: 1.5 !important;
}
.sub-menu-item:hover {
  background: rgba(57, 157, 255, 0.25) !important;
  color: #399dff !important;
  transform: none !important;
}
.sub-menu-item.active {
  background: rgba(57, 157, 255, 0.35) !important;
  color: #399dff !important;
  font-weight: 500 !important;
}
.sub-menu-item::before { display: none; }
@media (max-width: 768px) {
  .menu-bar { gap: 0.2rem; }
  .menu-bar button { font-size: 14px; padding: .4rem .8rem; }
  .sub-menu { min-width: 100px; }
  .sub-menu-item { font-size: 8px !important; padding: 0.2rem 0.8rem !important; }
}
</style>
