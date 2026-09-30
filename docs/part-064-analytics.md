# Part 64: Analytics Integration กับ Nuxt.js

## Google Analytics 4 กับ Nuxt

```bash
npm install @nuxtjs/google-analytics
# หรือใช้ gtag โดยตรง
npm install vue-gtag-next
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      script: [
        {
          src: `https://www.googletagmanager.com/gtag/js?id=${process.env.GA_MEASUREMENT_ID}`,
          async: true
        }
      ]
    }
  },
  
  runtimeConfig: {
    public: {
      gaMeasurementId: process.env.GA_MEASUREMENT_ID
    }
  }
})
```

```typescript
// plugins/analytics.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  const router = useRouter()
  
  // Initialize gtag
  window.dataLayer = window.dataLayer || []
  function gtag(...args: any[]) {
    window.dataLayer.push(args)
  }
  window.gtag = gtag
  
  gtag('js', new Date())
  gtag('config', config.public.gaMeasurementId, {
    send_page_view: false // Manual page view tracking
  })
  
  // Track page views
  router.afterEach((to) => {
    nextTick(() => {
      gtag('event', 'page_view', {
        page_title: document.title,
        page_location: window.location.href,
        page_path: to.fullPath
      })
    })
  })
})
```

## Custom Events

```typescript
// composables/useAnalytics.ts
type EventParams = Record<string, string | number | boolean>

interface AnalyticsEvent {
  action: string
  category?: string
  label?: string
  value?: number
  params?: EventParams
}

export function useAnalytics() {
  const config = useRuntimeConfig()
  const isEnabled = computed(() => 
    !!config.public.gaMeasurementId && 
    !import.meta.dev
  )

  const trackEvent = (event: AnalyticsEvent) => {
    if (!isEnabled.value || !window.gtag) return
    
    window.gtag('event', event.action, {
      event_category: event.category,
      event_label: event.label,
      value: event.value,
      ...event.params
    })
  }

  const trackPageView = (path?: string, title?: string) => {
    if (!isEnabled.value || !window.gtag) return
    
    window.gtag('event', 'page_view', {
      page_path: path || window.location.pathname,
      page_title: title || document.title,
      page_location: window.location.href
    })
  }

  const trackSearch = (query: string, resultsCount?: number) => {
    trackEvent({
      action: 'search',
      category: 'engagement',
      label: query,
      value: resultsCount,
      params: {
        search_term: query,
        results_count: resultsCount || 0
      }
    })
  }

  const trackSignup = (method: string) => {
    trackEvent({
      action: 'sign_up',
      params: { method }
    })
  }

  const trackLogin = (method: string) => {
    trackEvent({
      action: 'login',
      params: { method }
    })
  }

  const trackError = (error: string, fatal = false) => {
    if (!window.gtag) return
    window.gtag('event', 'exception', {
      description: error,
      fatal
    })
  }

  const setUser = (userId: string, properties?: Record<string, any>) => {
    if (!window.gtag) return
    window.gtag('config', config.public.gaMeasurementId, {
      user_id: userId
    })
    if (properties) {
      window.gtag('set', 'user_properties', properties)
    }
  }

  return {
    trackEvent,
    trackPageView,
    trackSearch,
    trackSignup,
    trackLogin,
    trackError,
    setUser
  }
}
```

## E-commerce Tracking

```typescript
// composables/useEcommerceAnalytics.ts
import { useAnalytics } from './useAnalytics'
import type { Product, CartItem, Order } from '~/types/ecommerce'

export function useEcommerceAnalytics() {
  const config = useRuntimeConfig()

  const getGA4Item = (product: Product, quantity = 1, index = 0) => ({
    item_id: product.id,
    item_name: product.name,
    item_category: product.category.name,
    price: product.price,
    quantity,
    index
  })

  const trackViewItemList = (products: Product[], listName = 'Product List') => {
    if (!window.gtag) return
    window.gtag('event', 'view_item_list', {
      item_list_name: listName,
      items: products.map((p, i) => getGA4Item(p, 1, i))
    })
  }

  const trackViewItem = (product: Product) => {
    if (!window.gtag) return
    window.gtag('event', 'view_item', {
      currency: 'THB',
      value: product.price,
      items: [getGA4Item(product)]
    })
  }

  const trackAddToCart = (product: Product, quantity = 1) => {
    if (!window.gtag) return
    window.gtag('event', 'add_to_cart', {
      currency: 'THB',
      value: product.price * quantity,
      items: [getGA4Item(product, quantity)]
    })
  }

  const trackRemoveFromCart = (product: Product, quantity = 1) => {
    if (!window.gtag) return
    window.gtag('event', 'remove_from_cart', {
      currency: 'THB',
      value: product.price * quantity,
      items: [getGA4Item(product, quantity)]
    })
  }

  const trackBeginCheckout = (items: CartItem[], total: number) => {
    if (!window.gtag) return
    window.gtag('event', 'begin_checkout', {
      currency: 'THB',
      value: total,
      items: items.map((item, i) => getGA4Item(item.product, item.quantity, i))
    })
  }

  const trackPurchase = (order: Order) => {
    if (!window.gtag) return
    window.gtag('event', 'purchase', {
      transaction_id: order.orderNumber,
      value: order.total,
      tax: order.tax,
      shipping: order.shipping,
      currency: 'THB',
      coupon: order.coupon?.code,
      items: order.items.map((item, i) => getGA4Item(item.product, item.quantity, i))
    })
  }

  return {
    trackViewItemList,
    trackViewItem,
    trackAddToCart,
    trackRemoveFromCart,
    trackBeginCheckout,
    trackPurchase
  }
}
```

## Mixpanel Integration

```typescript
// plugins/mixpanel.client.ts
import mixpanel from 'mixpanel-browser'

export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  
  if (!config.public.mixpanelToken) return
  
  mixpanel.init(config.public.mixpanelToken, {
    debug: import.meta.dev,
    track_pageview: true,
    persistence: 'localStorage'
  })
  
  return {
    provide: {
      mixpanel
    }
  }
})
```

```typescript
// composables/useMixpanel.ts
export function useMixpanel() {
  const { $mixpanel } = useNuxtApp()
  
  if (!$mixpanel) {
    return {
      track: () => {},
      identify: () => {},
      setProfile: () => {},
      reset: () => {}
    }
  }
  
  const track = (event: string, properties?: Record<string, any>) => {
    $mixpanel.track(event, {
      ...properties,
      timestamp: new Date().toISOString()
    })
  }
  
  const identify = (userId: string) => {
    $mixpanel.identify(userId)
  }
  
  const setProfile = (properties: Record<string, any>) => {
    $mixpanel.people.set(properties)
  }
  
  const setOnce = (properties: Record<string, any>) => {
    $mixpanel.people.set_once(properties)
  }
  
  const increment = (property: string, value = 1) => {
    $mixpanel.people.increment(property, value)
  }
  
  const reset = () => {
    $mixpanel.reset()
  }
  
  return { track, identify, setProfile, setOnce, increment, reset }
}
```

## Privacy-first Analytics (Plausible)

Plausible คือ analytics ที่ไม่ใช้ cookies และเป็น privacy-friendly

```typescript
// plugins/plausible.client.ts
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  if (!config.public.plausibleDomain) return
  
  // Load Plausible script
  const script = document.createElement('script')
  script.defer = true
  script.src = 'https://plausible.io/js/script.js'
  script.setAttribute('data-domain', config.public.plausibleDomain)
  document.head.appendChild(script)
  
  const router = useRouter()
  
  // Track page views (Plausible does this automatically, but for SPA)
  router.afterEach((to) => {
    nextTick(() => {
      if (window.plausible) {
        window.plausible('pageview')
      }
    })
  })
  
  return {
    provide: {
      plausible: (event: string, options?: { props?: Record<string, any> }) => {
        if (window.plausible) {
          window.plausible(event, options)
        }
      }
    }
  }
})
```

## Analytics Composable

```typescript
// composables/useTracking.ts
// Unified analytics composable that works with multiple providers

export function useTracking() {
  const { trackEvent: trackGA } = useAnalytics()
  const { track: trackMixpanel } = useMixpanel()
  const { $plausible } = useNuxtApp()
  
  const track = (
    eventName: string,
    properties?: Record<string, string | number | boolean>
  ) => {
    // Track to all providers
    trackGA({
      action: eventName,
      params: properties
    })
    
    trackMixpanel(eventName, properties)
    
    if ($plausible) {
      $plausible(eventName, { props: properties })
    }
  }
  
  const trackClick = (element: string, properties?: Record<string, any>) => {
    track('click', { element, ...properties })
  }
  
  const trackFormSubmit = (formName: string, success: boolean) => {
    track('form_submit', { form: formName, success })
  }
  
  const trackFeatureUse = (feature: string) => {
    track('feature_use', { feature })
  }
  
  const trackError = (error: string, context?: string) => {
    track('error', { error, context: context || 'unknown' })
  }
  
  const trackTiming = (category: string, variable: string, timeMs: number) => {
    if (!window.gtag) return
    window.gtag('event', 'timing_complete', {
      name: variable,
      value: timeMs,
      event_category: category
    })
  }
  
  return {
    track,
    trackClick,
    trackFormSubmit,
    trackFeatureUse,
    trackError,
    trackTiming
  }
}
```

```vue
<!-- components/analytics/TrackableLink.vue -->
<template>
  <NuxtLink
    :to="to"
    v-bind="$attrs"
    @click="handleClick"
  >
    <slot></slot>
  </NuxtLink>
</template>

<script setup lang="ts">
const props = defineProps<{
  to: string
  trackingName?: string
  trackingCategory?: string
}>()

const { trackClick } = useTracking()

const handleClick = () => {
  if (props.trackingName) {
    trackClick(props.trackingName, {
      category: props.trackingCategory || 'navigation',
      destination: props.to
    })
  }
}
</script>
```

## Cookie Consent

```vue
<!-- components/CookieConsent.vue -->
<template>
  <Transition name="slide-up">
    <div v-if="showBanner" class="cookie-consent">
      <div class="cookie-content">
        <p>
          เราใช้ cookies เพื่อปรับปรุงประสบการณ์การใช้งาน
          <NuxtLink to="/privacy-policy">นโยบายความเป็นส่วนตัว</NuxtLink>
        </p>
        <div class="cookie-actions">
          <button @click="acceptAll" class="btn-accept">ยอมรับทั้งหมด</button>
          <button @click="acceptNecessary" class="btn-necessary">เฉพาะที่จำเป็น</button>
          <button @click="showSettings = true" class="btn-settings">ตั้งค่า</button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const showBanner = ref(false)
const showSettings = ref(false)
const CONSENT_KEY = 'cookie_consent'

onMounted(() => {
  try {
    const saved = localStorage.getItem(CONSENT_KEY)
    if (!saved) showBanner.value = true
  } catch {}
})

const acceptAll = () => {
  const consent = { analytics: true, marketing: true, necessary: true }
  try {
    localStorage.setItem(CONSENT_KEY, JSON.stringify(consent))
  } catch {}
  showBanner.value = false
  initAnalytics()
}

const acceptNecessary = () => {
  const consent = { analytics: false, marketing: false, necessary: true }
  try {
    localStorage.setItem(CONSENT_KEY, JSON.stringify(consent))
  } catch {}
  showBanner.value = false
}

const initAnalytics = () => {
  if (window.gtag) {
    window.gtag('consent', 'update', {
      analytics_storage: 'granted',
      ad_storage: 'granted'
    })
  }
}
</script>
```

## สรุป

Analytics Integration ต้องคำนึงถึง:
1. Privacy compliance (PDPA, GDPR) - ขอ consent ก่อน
2. Cookie consent ก่อน tracking ทุกครั้ง
3. Multiple provider support ด้วย unified composable
4. E-commerce tracking สำหรับ conversion measurement
5. Custom events ที่มีความหมายต่อ business
6. Privacy-first alternatives (Plausible, Umami)
7. Server-side analytics เพื่อหลีกเลี่ยง ad blockers
