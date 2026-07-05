<template>
  <div
    class="vbwd-embed"
    :class="themeClass"
    data-testid="embed-landing1"
  >
    <Landing1View
      :embed-mode="true"
      :category="category"
      :theme="theme"
      :heading="heading"
      :subtitle="subtitle"
      :image-url="imageUrl"
      :features="features"
      :highlight-slug="highlightSlug"
      :badge="badge"
      :cta-label="ctaLabel"
      @plan-selected="onPlanSelected"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted } from 'vue';
import { useRoute } from 'vue-router';
import { useI18n } from 'vue-i18n';
import Landing1View from './Landing1View.vue';

const route = useRoute();
const { locale } = useI18n();

const allowedLocales = ['en', 'de'];
// Card themes understood by Landing1View. 'dark' additionally darkens the
// embed wrapper background; every other value keeps the light wrapper.
const allowedThemes = ['default', 'light', 'dark', 'teal', 'indigo', 'emerald'];

function q(key: string): string | undefined {
  const v = route.query[key];
  return typeof v === 'string' && v.trim() ? v : undefined;
}

const theme = computed(() => {
  const t = typeof route.query.theme === 'string' ? route.query.theme : 'light';
  return allowedThemes.includes(t) ? t : 'light';
});

const themeClass = computed(() =>
  theme.value === 'dark' ? 'vbwd-embed--dark' : 'vbwd-embed--light'
);

const category = computed(() =>
  typeof route.query.category === 'string' ? route.query.category : ''
);

const heading = computed(() => q('heading'));
const subtitle = computed(() => q('subtitle'));
const imageUrl = computed(() => q('image'));
const highlightSlug = computed(() => q('highlight'));
const badge = computed(() => q('badge'));
const ctaLabel = computed(() => q('cta'));
const features = computed<string[] | undefined>(() => {
  const raw = q('features');
  if (!raw) return undefined;
  const list = raw.split(',').map((s) => s.trim()).filter(Boolean);
  return list.length ? list : undefined;
});

onMounted(() => {
  const queryLocale = typeof route.query.locale === 'string' ? route.query.locale : 'en';
  if (allowedLocales.includes(queryLocale)) {
    locale.value = queryLocale;
  }
  initResizeObserver();
});

function onPlanSelected(plan: { slug: string; name: string; price: number; currency: string }) {
  if (window.parent !== window) {
    window.parent.postMessage({
      type: 'vbwd:plan-selected',
      payload: {
        planSlug: plan.slug,
        planName: plan.name,
        price: plan.price,
        currency: plan.currency,
      }
    }, '*');
  }
}

// Auto-resize: notify parent of content height changes
let resizeObserver: ResizeObserver | null = null;

function initResizeObserver() {
  if (window.parent === window) return;

  resizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
      const height = entry.borderBoxSize?.[0]?.blockSize ?? entry.target.scrollHeight;
      window.parent.postMessage({
        type: 'vbwd:resize',
        payload: { height: Math.ceil(height) }
      }, '*');
    }
  });

  const root = document.querySelector('.vbwd-embed');
  if (root) resizeObserver.observe(root);
}

onUnmounted(() => {
  resizeObserver?.disconnect();
});
</script>

<style scoped>
.vbwd-embed {
  background: #ffffff;
  min-height: 100vh;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.vbwd-embed--dark {
  background: #111827;
  color: #f3f4f6;
}
</style>
