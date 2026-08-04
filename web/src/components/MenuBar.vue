<template>
  <nav class="menu-bar">
    <div
      v-for="menu in menus"
      :key="menu.id"
      class="menu-item"
      @mouseenter="showSubMenu(menu.id)"
      @mouseleave="hideSubMenu(menu.id)"
    >
      <button
        :data-menu-id="menu.id"
        :data-sub-id="null"
        @click="$emit('select', menu)"
        :class="{active: menu.id === activeId}"
      >
        {{ menu.name }}
      </button>

      <!-- 二级菜单 -->
      <div v-if="menu.subMenus && menu.subMenus.length > 0"
           class="sub-menu" :class="{ 'show': hoveredMenuId === menu.id }">
        <button
          v-for="subMenu in menu.subMenus"
          :key="subMenu.id"
          :data-menu-id="menu.id"
          :data-sub-id="subMenu.id"
          class="sub-menu-item"
          @click="$emit('select', subMenu, menu)"
          :class="{active: subMenu.id === activeSubMenuId}"
        >
          {{ subMenu.name }}
        </button>
      </div>
    </div>
  </nav>
</template>

<script setup>
import { ref } from 'vue';

defineProps({
  menus: Array,
  activeId: Number,
  activeSubMenuId: Number
});
defineEmits(['select']);

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
  gap: 4px;
}
.menu-item { position: relative; }
.menu-bar button {
  background: transparent;
  border: none;
  color: #fff;
  font-size: 16px;
  font-weight: 500;
  padding: 0.8rem 1.4rem;
  cursor: pointer;
  white-space: nowrap;   /* ① 新增：防止手机端文字被挤成竖排 */
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

/* 二级菜单 */
.sub-menu {
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(92, 89, 89, 0.35);
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
  padding: 4px 0;
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
  white-space: nowrap !important;   /* ① 新增 */
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
  .menu-bar { gap: 0; padding: 0; }
  .menu-bar button { font-size: 13px; padding: .4rem .6rem; }
  .sub-menu { min-width: 90px; }
  .sub-menu-item { font-size: 12px !important; padding: 0.3rem 0.8rem !important; }
}
</style>
