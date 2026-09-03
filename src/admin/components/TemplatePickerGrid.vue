<template>
  <div class="pollquest-template-picker">
    <!-- Toolbar: Categories & Search -->
    <div v-if="!loading" class="pollquest-template-toolbar">
      <div class="pollquest-template-tabs">
        <button
          v-for="cat in categories"
          :key="cat.key"
          type="button"
          class="pollquest-template-tab"
          :class="{ active: activeCategory === cat.key }"
          @click="activeCategory = cat.key"
        >
          <span>{{ cat.label }}</span>
          <span class="pollquest-template-tab-count">{{ cat.count }}</span>
        </button>
      </div>

      <div class="pollquest-template-search-wrap">
        <Search class="pollquest-template-search-icon" />
        <input
          v-model="searchQuery"
          type="text"
          class="pollquest-template-search-input"
          placeholder="Search templates..."
        />
        <button
          v-if="searchQuery"
          type="button"
          class="pollquest-template-search-clear"
          @click="searchQuery = ''"
          aria-label="Clear search"
        >
          <X />
        </button>
      </div>
    </div>

    <div v-if="loading" class="pollquest-template-loading">Loading templates…</div>

    <!-- Empty filtered state -->
    <div v-else-if="filteredTemplates.length === 0" class="pollquest-template-empty">
      <Inbox class="pollquest-template-empty-icon" />
      <p class="pollquest-template-empty-title">No templates found</p>
      <p class="pollquest-template-empty-desc">
        Try selecting a different category or clearing your search.
      </p>
      <button
        type="button"
        class="pollquest-btn pollquest-btn-secondary pollquest-btn-sm"
        @click="activeCategory = 'all'; searchQuery = ''"
      >
        View all templates
      </button>
    </div>

    <!-- Template Grid -->
    <div v-else class="pollquest-template-grid">
      <button
        v-for="template in filteredTemplates"
        :key="template.id"
        type="button"
        class="pollquest-template-card"
        :class="{
          'pollquest-template-card--selected': selectedId === template.id,
          'pollquest-template-card--locked': !template.is_available,
        }"
        :disabled="!template.is_available"
        @click="selectTemplate(template)"
      >
        <div class="pollquest-template-card-icon" :class="{ 'pollquest-template-card-icon--locked': !template.is_available }">
          <component :is="iconFor(template.icon)" />
          <span v-if="!template.is_available" class="pollquest-template-lock-badge">
            <Lock />
          </span>
        </div>

        <div class="pollquest-template-card-body">
          <div class="pollquest-template-card-title-row">
            <span class="pollquest-template-card-title">{{ template.title }}</span>
            <span v-if="template.is_pro" class="pollquest-template-pro-badge">Pro</span>
          </div>
          <p class="pollquest-template-card-desc">{{ template.description }}</p>
          <div class="pollquest-template-card-meta-row">
            <span class="pollquest-template-card-meta">
              {{ template.questions?.length || 0 }} questions
            </span>
            <span v-if="template.category" class="pollquest-template-category-badge">
              {{ categoryLabel(template.category) }}
            </span>
          </div>
        </div>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import {
  Lock,
  FilePlus,
  Globe,
  TrendingUp,
  ShoppingCart,
  Mail,
  Store,
  Headphones,
  LayoutTemplate,
  Search,
  X,
  Inbox,
} from 'lucide-vue-next';

const props = defineProps({
  selectedId: { type: String, default: '' },
  highlightIds: { type: Array, default: () => [] },
});

const emit = defineEmits(['select', 'loaded']);

const loading = ref(true);
const templates = ref([]);
const activeCategory = ref('all');
const searchQuery = ref('');

const categoryMap = {
  all: 'All',
  general: 'Feedback',
  nps: 'NPS & Ratings',
  leads: 'Lead Capture',
  ecommerce: 'eCommerce',
  content: 'Content',
};

function categoryLabel(cat) {
  return categoryMap[cat] || 'Feedback';
}

const categories = computed(() => {
  const counts = { all: templates.value.length };
  templates.value.forEach((t) => {
    const cat = t.category || 'general';
    counts[cat] = (counts[cat] || 0) + 1;
  });

  const list = [
    { key: 'all', label: 'All Templates', count: counts.all || 0 },
    { key: 'general', label: 'Feedback', count: counts.general || 0 },
    { key: 'nps', label: 'NPS & Ratings', count: counts.nps || 0 },
    { key: 'leads', label: 'Lead Capture', count: counts.leads || 0 },
    { key: 'ecommerce', label: 'eCommerce', count: counts.ecommerce || 0 },
    { key: 'content', label: 'Content', count: counts.content || 0 },
  ];

  return list.filter((c) => c.key === 'all' || c.count > 0);
});

const filteredTemplates = computed(() => {
  return templates.value.filter((t) => {
    const cat = t.category || 'general';
    const matchesCat = activeCategory.value === 'all' || cat === activeCategory.value;
    const query = searchQuery.value.trim().toLowerCase();
    const matchesQuery =
      !query ||
      (t.title && t.title.toLowerCase().includes(query)) ||
      (t.description && t.description.toLowerCase().includes(query));
    return matchesCat && matchesQuery;
  });
});

const iconMap = {
  'file-plus': FilePlus,
  globe: Globe,
  'trending-up': TrendingUp,
  'shopping-cart': ShoppingCart,
  mail: Mail,
  store: Store,
  headphones: Headphones,
};

function iconFor(name) {
  return iconMap[name] || LayoutTemplate;
}

function selectTemplate(template) {
  if (!template.is_available) {
    if (template.upgrade_url) {
      window.open(template.upgrade_url, '_blank', 'noopener,noreferrer');
    }
    return;
  }
  emit('select', template);
}

async function fetchTemplates() {
  const config = window.PollQuestAdminConfig || {};

  try {
    const res = await fetch(`${config.api_url}/survey-templates`, {
      headers: { 'X-WP-Nonce': config.nonce },
    });

    if (res.ok) {
      let data = await res.json();
      if (props.highlightIds.length > 0) {
        data = [...data].sort((a, b) => {
          const aHighlight = props.highlightIds.includes(a.id) ? 0 : 1;
          const bHighlight = props.highlightIds.includes(b.id) ? 0 : 1;
          return aHighlight - bHighlight;
        });
      }
      templates.value = data;
      emit('loaded', data);
    }
  } catch (err) {
    console.error('Failed to load templates', err);
  } finally {
    loading.value = false;
  }
}

onMounted(fetchTemplates);
</script>
