<template>
  <div class="card-manage">
    <div class="card-header">
      <div class="header-content">
        <h2 class="page-title">管理网站导航卡片，支持主菜单和子菜单分类</h2>
      </div>
      <div class="card-add">
        <select v-model="selectedMenuId" class="input narrow" @change="onMenuChange">
          <option v-for="menu in menus" :value="menu.id" :key="menu.id">{{ menu.name }}</option>
        </select>
        <select v-model="selectedSubMenuId" class="input narrow" @change="onSubMenuChange">
          <option value="">主菜单</option>
          <option v-for="subMenu in currentSubMenus" :value="subMenu.id" :key="subMenu.id">{{ subMenu.name }}</option>
        </select>
        <input v-model="newCardTitle" placeholder="卡片标题" class="input narrow" />
        <input v-model="newCardUrl" placeholder="卡片链接" class="input wide" />
        <input v-model="newCardLogo" placeholder="logo链接(可选)" class="input wide" />
        <button class="btn" @click="addCard">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 5v14M5 12h14"/>
          </svg>
          添加卡片
        </button>
      </div>
    </div>
    <div class="card-card">
      <table class="card-table">
        <thead>
          <tr>
            <th>归属菜单</th>
            <th>归属子菜单</th>
            <th>标题</th>
            <th>网址</th>
            <th>Logo链接</th>
            <th>描述</th>
            <th>排序</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="card in cards" :key="card.id">
            <!-- 归属主菜单下拉框 -->
            <td>
              <select
                v-model="card.menu_id"
                @change="onCardMenuChange(card)"
                class="table-input归属-select"
              >
                <option v-for="menu in menus" :value="menu.id" :key="menu.id">{{ menu.name }}</option>
              </select>
            </td>
            <!-- 归属子菜单下拉框 -->
            <td>
              <select
                v-model="card.sub_menu_id"
                @change="onCardSubMenuChange(card)"
                class="table-input归属-select"
              >
                <option value="">无（主菜单）</option>
                <option
                  v-for="subMenu in getSubMenusForCard(card)"
                  :value="subMenu.id"
                  :key="subMenu.id"
                >{{ subMenu.name }}</option>
              </select>
            </td>
            <td><input v-model="card.title" @blur="updateCard(card)" class="table-input" /></td>
            <td><input v-model="card.url" @blur="updateCard(card)" class="table-input" /></td>
            <td><input v-model="card.logo_url" @blur="updateCard(card)" class="table-input" placeholder="logo链接(可选)" /></td>
            <td><input v-model="card.desc" @blur="updateCard(card)" class="table-input" placeholder="描述（可选）" /></td>
            <td><input v-model.number="card.order" type="number" @blur="updateCard(card)" class="table-input order-input" /></td>
            <td>
              <button class="btn btn-danger btn-icon" @click="deleteCard(card.id)" title="删除">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6"/>
                  <path d="M10 11v6M14 11v6"/>
                </svg>
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue';
import { 
  getMenus, 
  getCards, 
  addCard as apiAddCard, 
  updateCard as apiUpdateCard, 
  deleteCard as apiDeleteCard 
} from '../../api';

const menus = ref([]);
const cards = ref([]);
const selectedMenuId = ref();
const selectedSubMenuId = ref('');
const newCardTitle = ref('');
const newCardUrl = ref('');
const newCardLogo = ref('');

const currentSubMenus = computed(() => {
  if (!selectedMenuId.value) return [];
  const menu = menus.value.find(m => m.id === selectedMenuId.value);
  return menu?.subMenus || [];
});

onMounted(async () => {
  const res = await getMenus();
  menus.value = res.data;
  if (menus.value.length) {
    selectedMenuId.value = menus.value[0].id;
    selectedSubMenuId.value = '';
  }
});

watch(selectedMenuId, () => {
  selectedSubMenuId.value = '';
  loadCards();
});

watch(selectedSubMenuId, loadCards);

function onMenuChange() {
  selectedSubMenuId.value = '';
}

function onSubMenuChange() {
  loadCards();
}

// 获取卡片所属主菜单下的子菜单列表
function getSubMenusForCard(card) {
  if (!card.menu_id) return [];
  const menu = menus.value.find(m => m.id === card.menu_id);
  return menu?.subMenus || [];
}

// 当卡片的归属主菜单改变时
async function onCardMenuChange(card) {
  // 切换主菜单后，如果当前子菜单不属于新主菜单，则清空
  const newSubMenus = getSubMenusForCard(card);
  const hasSubMenu = newSubMenus.some(s => s.id === card.sub_menu_id);
  if (!hasSubMenu) {
    card.sub_menu_id = '';
  }
  await updateCard(card);
}

// 当卡片的归属子菜单改变时
async function onCardSubMenuChange(card) {
  await updateCard(card);
}

async function loadCards() {
  if (!selectedMenuId.value) return;
  const res = await getCards(selectedMenuId.value, selectedSubMenuId.value || null);
  cards.value = res.data;
}

async function addCard() {
  if (!newCardTitle.value || !newCardUrl.value) return;
  await apiAddCard({ 
    menu_id: selectedMenuId.value, 
    sub_menu_id: selectedSubMenuId.value || null,
    title: newCardTitle.value, 
    url: newCardUrl.value, 
    logo_url: newCardLogo.value 
  });
  newCardTitle.value = '';
  newCardUrl.value = '';
  newCardLogo.value = '';
  loadCards();
}

async function updateCard(card) {
  await apiUpdateCard(card.id, {
    menu_id: card.menu_id,
    sub_menu_id: card.sub_menu_id || null,
    title: card.title,
    url: card.url,
    logo_url: card.logo_url,
    desc: card.desc,
    order: card.order
  });
  loadCards();
}

async function deleteCard(id) {
  await apiDeleteCard(id);
  loadCards();
}
</script>

<style scoped>
.card-manage {
  max-width: 1200px;
  width: 95%;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.card-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 20px;
  color: white;
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.3);
  width: 95%;
  text-align: center;
}

.header-content {
  margin-bottom: 15px;
  text-align: center;
}

.page-title {
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0 0 8px 0;
  letter-spacing: -0.5px;
}

.card-add {
  margin: 0 auto;
  display: flex;
  gap: 5px;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
}

.card-card {
  background: var(--admin-card-bg, white);
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  overflow-x: auto; /* 允许横向滚动 */
  width: 100%;
}

.card-table {
  width: 100%;
  border-collapse: collapse;
  padding: 24px;
  min-width: 1000px; /* 保证表格在小屏幕上也能横向滚动 */
}

.card-table th,
.card-table td {
  padding: 8px 12px;
  text-align: left;
  border-bottom: 1px solid var(--admin-border, #e5e7eb);
}

.card-table th {
  background: var(--admin-card-header-bg, #f9fafb);
  font-weight: 600;
  color: var(--admin-text, #222);
}

/* 表格列宽度设置 */
.card-table th:nth-child(1), /* 归属菜单列 */
.card-table td:nth-child(1) {
  width: 14%;
}

.card-table th:nth-child(2), /* 归属子菜单列 */
.card-table td:nth-child(2) {
  width: 14%;
}

.card-table th:nth-child(3), /* 标题列 */
.card-table td:nth-child(3) {
  width: 10%;
}

.card-table th:nth-child(4), /* 网址列 */
.card-table td:nth-child(4) {
  width: 20%;
}

.card-table th:nth-child(5), /* Logo链接列 */
.card-table td:nth-child(5) {
  width: 15%;
}

.card-table th:nth-child(6), /* 描述列 */
.card-table td:nth-child(6) {
  width: 10%;
}

.card-table th:nth-child(7), /* 排序列 */
.card-table td:nth-child(7) {
  width: 6%;
}

.card-table th:nth-child(8), /* 操作列 */
.card-table td:nth-child(8) {
  width: 10%;
  text-align: center;
}

.input {
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid var(--admin-input-border, #d0d7e2);
  background: var(--admin-input-bg, #fff);
  color: var(--admin-text, #222);
  font-size: 0.9rem;
  transition: all 0.2s ease;
}

/* 窄输入框 */
.input.narrow {
  width: 140px;
}

/* 宽输入框 */
.input.wide {
  width: 200px;
}

/* 表格内输入框 */
.table-input {
  width: 100%;
  padding: 8px 4px;
  border-radius: 6px;
  border: 1px solid var(--admin-border, #e2e8f0);
  background: var(--admin-input-bg, #fff);
  color: var(--admin-text, #222);
  font-size: 0.85rem;
  transition: all 0.2s ease;
}

.table-input:focus {
  outline: none;
  border-color: #399dff;
  box-shadow: 0 0 0 2px rgba(57, 157, 255, 0.1);
}

.input:focus {
  outline: none;
  border-color: #399dff;
  box-shadow: 0 0 0 3px rgba(57, 157, 255, 0.1);
}

.order-input {
  width: 60px;
}

/* 归属下拉框样式 */
.归属-select {
  width: 100%;
  padding: 6px 4px;
  border-radius: 6px;
  border: 1px solid var(--admin-border, #e2e8f0);
  background: var(--admin-input-bg, #fff);
  color: var(--admin-text, #222);
  font-size: 0.85rem;
  transition: all 0.2s ease;
  cursor: pointer;
}

.归属-select:focus {
  outline: none;
  border-color: #399dff;
  box-shadow: 0 0 0 2px rgba(57, 157, 255, 0.1);
}

.btn {
  padding: 10px 8px;
  border: none;
  border-radius: 8px;
  background: #399dff;
  color: white;
  cursor: pointer;
  font-weight: 500;
  font-size: 0.9rem;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.btn-icon {
  width: 32px;
  height: 32px;
  padding: 0;
  justify-content: center;
  border-radius: 6px;
}

.btn:hover {
  background: #2d7dd2;
  transform: translateY(-1px);
}

.btn-danger {
  background: #ef4444;
}

.btn-danger:hover {
  background: #dc2626;
}

@media (max-width: 768px) {
  .card-manage {
    width: 94%;
    padding: 16px;
  }
  
  .card-card {
    padding: 16px 12px;
  }
  
  .card-add {
    flex-direction: column;
    align-items: stretch;
    gap: 8px;
  }
  
  .input.narrow,
  .input.wide {
    width: 100%;
  }
  
  .order-input {
    width: 60px;
  }
}
</style>
