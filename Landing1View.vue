<template>
  <div
    class="landing1"
    :class="themeClass"
    data-testid="landing1-root"
  >
    <div class="landing1-header">
      <h1 data-testid="landing1-title">
        {{ headingText }}
      </h1>
      <p class="subtitle">
        {{ subtitleText }}
      </p>
    </div>

    <!-- Loading State -->
    <div
      v-if="loading"
      class="loading-state"
      data-testid="landing1-loading"
    >
      <div class="spinner" />
      <p>{{ $t('common.loading') }}</p>
    </div>

    <!-- Error State -->
    <div
      v-else-if="error"
      class="error-state"
      data-testid="landing1-error"
    >
      <p>{{ error }}</p>
      <button
        class="retry-btn"
        @click="loadPlans"
      >
        {{ $t('common.retry') }}
      </button>
    </div>

    <!-- Empty State -->
    <div
      v-else-if="plans.length === 0"
      class="empty-state"
      data-testid="landing1-empty"
    >
      <p>{{ $t('landing1.noPlans') }}</p>
    </div>

    <!-- Plan Cards Grid -->
    <div
      v-else
      class="plans-grid"
      data-testid="landing1-plans"
    >
      <div
        v-for="plan in plans"
        :key="plan.id"
        class="plan-card"
        :class="{ 'plan-card--featured': isFeatured(plan) }"
        :data-testid="`plan-card-${plan.slug}`"
      >
        <div
          v-if="isFeatured(plan)"
          class="plan-card__badge"
          data-testid="plan-card-badge"
        >
          {{ badgeText }}
        </div>

        <img
          v-if="imageUrl"
          class="plan-card__image"
          :src="imageUrl"
          :alt="plan.name"
        >

        <h2 class="plan-name">
          {{ plan.name }}
        </h2>
        <div class="plan-price">
          <span class="plan-price__amount">{{ formatPrice(plan.display_price, plan.display_currency) }}</span>
          <span
            v-if="plan.billing_period"
            class="billing-period"
          >/{{ formatBillingPeriod(plan.billing_period) }}</span>
        </div>
        <p
          v-if="plan.description"
          class="plan-description"
        >
          {{ plan.description }}
        </p>

        <ul
          v-if="features.length"
          class="plan-features"
          data-testid="plan-features"
        >
          <li
            v-for="(feature, i) in features"
            :key="i"
            class="plan-features__item"
          >
            <svg
              class="plan-features__check"
              viewBox="0 0 20 20"
              aria-hidden="true"
            >
              <path
                d="M7.5 13.5 4 10l-1.2 1.2L7.5 16 17 6.5 15.8 5.3z"
                fill="currentColor"
              />
            </svg>
            <span>{{ feature }}</span>
          </li>
        </ul>

        <button
          class="choose-plan-btn"
          :data-testid="`choose-plan-${plan.slug}`"
          @click="choosePlan(plan)"
        >
          {{ ctaText }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import { formatMoney } from 'vbwd-view-component';

export interface TarifPlan {
  id: string;
  name: string;
  slug: string;
  description?: string;
  display_price: number;
  display_currency: string;
  billing_period?: string;
  is_active?: boolean;
}

const ALLOWED_THEMES = ['default', 'light', 'dark', 'teal', 'indigo', 'emerald'];

const props = defineProps<{
  embedMode?: boolean;
  category?: string;
  planSlugs?: string[];
  heading?: string;
  subtitle?: string;
  theme?: string;
  imageUrl?: string;
  features?: string[];
  highlightSlug?: string;
  badge?: string;
  ctaLabel?: string;
}>();

const emit = defineEmits<{
  (e: 'plan-selected', plan: { slug: string; name: string; price: number; currency: string }): void;
}>();

const router = useRouter();
const { t } = useI18n();

const plans = ref<TarifPlan[]>([]);
const loading = ref(false);
const error = ref<string | null>(null);

const themeClass = computed(() => {
  const theme = props.theme && ALLOWED_THEMES.includes(props.theme) ? props.theme : 'default';
  return `landing1--${theme}`;
});

const headingText = computed(() => props.heading?.trim() || t('landing1.title'));
const subtitleText = computed(() => props.subtitle?.trim() || t('landing1.subtitle'));
const ctaText = computed(() => props.ctaLabel?.trim() || t('landing1.choosePlan'));
const badgeText = computed(() => props.badge?.trim() || t('landing1.popular'));
const features = computed<string[]>(() =>
  (props.features ?? []).map((f) => String(f).trim()).filter(Boolean)
);

function isFeatured(plan: TarifPlan): boolean {
  return !!props.highlightSlug && plan.slug === props.highlightSlug;
}

async function loadPlans() {
  loading.value = true;
  error.value = null;
  try {
    const url = props.category
      ? `/api/v1/tarif-plans?category=${encodeURIComponent(props.category)}`
      : '/api/v1/tarif-plans';
    const response = await fetch(url);
    if (!response.ok) throw new Error('Failed to load plans');
    const data = await response.json();
    const all = (data.plans || []).filter((p: TarifPlan) => p.is_active !== false);
    plans.value = props.planSlugs?.length
      ? all.filter((p: TarifPlan) => props.planSlugs!.includes(p.slug))
      : all;
  } catch (e) {
    error.value = (e as Error).message || t('common.errors.generic');
  } finally {
    loading.value = false;
  }
}

function choosePlan(plan: TarifPlan) {
  if (props.embedMode) {
    emit('plan-selected', {
      slug: plan.slug,
      name: plan.name,
      price: plan.display_price,
      currency: plan.display_currency,
    });
  } else {
    router.push({ path: '/checkout', query: { tarif_plan_id: plan.slug } });
  }
}

function formatPrice(price: number, currency?: string): string {
  // Resolve: the value's own currency, else the operating currency (S99).
  return formatMoney(price, { currency });
}

function formatBillingPeriod(period?: string): string {
  if (!period) return 'month';
  const periodMap: Record<string, string> = {
    monthly: 'month',
    yearly: 'year',
    annual: 'year',
    weekly: 'week',
    MONTHLY: 'month',
    YEARLY: 'year',
  };
  return periodMap[period] || period.toLowerCase();
}

onMounted(() => {
  loadPlans();
});
</script>

<style>
/* Non-scoped so CMS page styles and NativePricingPlans widget CSS can override these rules. */
.landing1 {
  /* Theme tokens — overridden per .landing1--<theme> below. Fall back to the
     shared fe-user vbwd-* tokens so the widget still tracks the active CMS style. */
  --l1-primary: var(--vbwd-color-primary, #3498db);
  --l1-primary-hover: var(--vbwd-color-primary-hover, #2980b9);
  --l1-card-bg: var(--vbwd-card-bg, #ffffff);
  --l1-text-heading: var(--vbwd-text-heading, #1f2937);
  --l1-text-muted: var(--vbwd-text-muted, #6b7280);
  --l1-border: var(--vbwd-border-light, #e5e7eb);
  --l1-check: var(--l1-primary);
  --l1-card-shadow: var(--vbwd-card-shadow, 0 2px 10px rgba(0, 0, 0, 0.08));

  max-width: 1200px;
  margin: 0 auto;
  padding: 40px 20px;
}

/* ── Themes ──────────────────────────────────────────────────────────────── */
.landing1--light {
  --l1-card-bg: #ffffff;
  --l1-text-heading: #1f2937;
  --l1-text-muted: #6b7280;
  --l1-border: #e5e7eb;
}

.landing1--dark {
  --l1-primary: #3b82f6;
  --l1-primary-hover: #2563eb;
  --l1-card-bg: #1f2937;
  --l1-text-heading: #f9fafb;
  --l1-text-muted: #9ca3af;
  --l1-border: #374151;
  --l1-card-shadow: 0 2px 14px rgba(0, 0, 0, 0.5);
}

.landing1--teal {
  --l1-primary: #14b8a6;
  --l1-primary-hover: #0d9488;
  --l1-check: #14b8a6;
}

.landing1--indigo {
  --l1-primary: #6366f1;
  --l1-primary-hover: #4f46e5;
  --l1-check: #6366f1;
}

.landing1--emerald {
  --l1-primary: #10b981;
  --l1-primary-hover: #059669;
  --l1-check: #10b981;
}

.landing1-header {
  text-align: center;
  margin-bottom: 40px;
}

.landing1-header h1 {
  font-size: 2rem;
  color: var(--l1-text-heading);
  margin-bottom: 10px;
}

.landing1 .subtitle {
  color: var(--l1-text-muted);
  font-size: 1.1rem;
}

.landing1 .loading-state,
.landing1 .error-state,
.landing1 .empty-state {
  text-align: center;
  padding: 60px 20px;
  color: var(--l1-text-muted);
}

.landing1 .spinner {
  width: 40px;
  height: 40px;
  border: 3px solid var(--l1-border);
  border-top: 3px solid var(--l1-primary);
  border-radius: 50%;
  animation: landing1-spin 1s linear infinite;
  margin: 0 auto 15px;
}

@keyframes landing1-spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.landing1 .retry-btn {
  margin-top: 15px;
  padding: 10px 20px;
  background: var(--l1-primary);
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.landing1 .plans-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 24px;
  align-items: stretch;
}

.landing1 .plan-card {
  position: relative;
  display: flex;
  flex-direction: column;
  background: var(--l1-card-bg);
  padding: 32px 28px;
  border-radius: 16px;
  box-shadow: var(--l1-card-shadow);
  text-align: center;
  transition: transform 0.2s, box-shadow 0.2s, border-color 0.2s;
  border: 2px solid transparent;
}

.landing1 .plan-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 30px rgba(0, 0, 0, 0.14);
  border-color: var(--l1-primary);
}

/* ── Emphasized / "popular" plan ─────────────────────────────────────────── */
.landing1 .plan-card--featured {
  border-color: var(--l1-primary);
  box-shadow: 0 16px 40px rgba(0, 0, 0, 0.16);
  z-index: 1;
}

@media (min-width: 900px) {
  .landing1 .plan-card--featured {
    transform: scale(1.05);
  }
  .landing1 .plan-card--featured:hover {
    transform: scale(1.05) translateY(-4px);
  }
}

.landing1 .plan-card__badge {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translate(-50%, -50%);
  background: var(--l1-primary);
  color: #fff;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  padding: 5px 14px;
  border-radius: 999px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.18);
  white-space: nowrap;
}

.landing1 .plan-card__image {
  display: block;
  width: 72px;
  height: 72px;
  object-fit: contain;
  margin: 4px auto 16px;
}

.landing1 .plan-name {
  font-size: 1.35rem;
  font-weight: 700;
  color: var(--l1-text-heading);
  margin-bottom: 14px;
}

.landing1 .plan-price {
  margin-bottom: 18px;
}

.landing1 .plan-price__amount {
  font-size: 2.25rem;
  font-weight: 800;
  color: var(--l1-primary);
}

.landing1 .billing-period {
  font-size: 0.9rem;
  font-weight: 400;
  color: var(--l1-text-muted);
}

.landing1 .plan-description {
  color: var(--l1-text-muted);
  font-size: 0.9rem;
  margin-bottom: 18px;
  line-height: 1.5;
}

.landing1 .plan-features {
  list-style: none;
  padding: 0;
  margin: 0 0 24px;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.landing1 .plan-features__item {
  display: flex;
  align-items: center;
  gap: 10px;
  color: var(--l1-text-muted);
  font-size: 0.95rem;
}

.landing1 .plan-features__check {
  flex-shrink: 0;
  width: 18px;
  height: 18px;
  color: var(--l1-check);
}

.landing1 .choose-plan-btn {
  width: 100%;
  margin-top: auto;
  padding: 14px;
  background-color: var(--l1-primary);
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s;
}

.landing1 .choose-plan-btn:hover {
  background-color: var(--l1-primary-hover);
}

.landing1 .plan-card--featured .choose-plan-btn {
  background-image: linear-gradient(135deg, var(--l1-primary), var(--l1-primary-hover));
}
</style>
