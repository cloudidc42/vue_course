# Part 56: Accessibility (a11y) ใน Vue.js

## ความสำคัญของ Accessibility

Accessibility (a11y) คือการออกแบบและพัฒนาเว็บแอปพลิเคชันให้ผู้ใช้ทุกคนสามารถเข้าถึงได้ รวมถึงผู้พิการทางสายตา การได้ยิน และการเคลื่อนไหว การทำ Accessibility ไม่ใช่แค่เรื่องของ compliance แต่เป็นการสร้างประสบการณ์ที่ดีสำหรับทุกคน

## WCAG 2.1 Guidelines

WCAG (Web Content Accessibility Guidelines) คือมาตรฐานสำหรับ Accessibility ที่กำหนดโดย W3C มี 4 หลักการหลัก:

### POUR Principles
- **Perceivable** - ผู้ใช้ต้องรับรู้ข้อมูลได้
- **Operable** - ผู้ใช้ต้องใช้งาน interface ได้
- **Understandable** - ผู้ใช้ต้องเข้าใจข้อมูลและการทำงาน
- **Robust** - เนื้อหาต้องทนทานต่อเทคโนโลยีต่างๆ

### Conformance Levels
- **Level A** - ข้อกำหนดขั้นพื้นฐาน
- **Level AA** - มาตรฐานที่แนะนำสำหรับเว็บทั่วไป
- **Level AAA** - มาตรฐานสูงสุด

```vue
<!-- Good: ใช้ semantic HTML และ ARIA -->
<template>
  <main>
    <h1>หน้าแรก</h1>
    <nav aria-label="เมนูหลัก">
      <ul>
        <li><a href="/">หน้าแรก</a></li>
        <li><a href="/about">เกี่ยวกับเรา</a></li>
      </ul>
    </nav>
    
    <!-- Bad: ใช้ div แทน semantic elements -->
    <!-- <div class="nav">...</div> -->
  </main>
</template>
```

## Semantic HTML ใน Vue

การใช้ HTML elements ที่ถูกต้องช่วยให้ Screen Reader เข้าใจโครงสร้างของหน้าเว็บ

```vue
<!-- components/SemanticLayout.vue -->
<template>
  <div class="app">
    <!-- Skip navigation link -->
    <a href="#main-content" class="skip-link">
      ข้ามไปยังเนื้อหาหลัก
    </a>
    
    <!-- Header -->
    <header role="banner">
      <nav aria-label="เมนูหลัก">
        <ul role="list">
          <li v-for="item in navItems" :key="item.id">
            <RouterLink 
              :to="item.path"
              :aria-current="isCurrentRoute(item.path) ? 'page' : undefined"
            >
              {{ item.label }}
            </RouterLink>
          </li>
        </ul>
      </nav>
    </header>
    
    <!-- Main content -->
    <main id="main-content" tabindex="-1">
      <article v-if="article">
        <header>
          <h1>{{ article.title }}</h1>
          <time :datetime="article.publishedAt">
            {{ formatDate(article.publishedAt) }}
          </time>
        </header>
        
        <section aria-labelledby="content-heading">
          <h2 id="content-heading">เนื้อหา</h2>
          <div v-html="article.content"></div>
        </section>
        
        <aside aria-label="บทความที่เกี่ยวข้อง">
          <h2>บทความที่เกี่ยวข้อง</h2>
          <ul>
            <li v-for="related in relatedArticles" :key="related.id">
              <a :href="related.url">{{ related.title }}</a>
            </li>
          </ul>
        </aside>
      </article>
    </main>
    
    <!-- Footer -->
    <footer role="contentinfo">
      <p>&copy; 2024 Vue Course</p>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

const navItems = [
  { id: 1, label: 'หน้าแรก', path: '/' },
  { id: 2, label: 'บทเรียน', path: '/lessons' },
  { id: 3, label: 'ติดต่อ', path: '/contact' }
]

const isCurrentRoute = (path: string) => route.path === path

const formatDate = (dateStr: string) => {
  return new Date(dateStr).toLocaleDateString('th-TH')
}
</script>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
</style>
```

## ARIA Attributes

ARIA (Accessible Rich Internet Applications) ช่วยเพิ่มข้อมูล accessibility สำหรับ elements ที่ไม่มี semantic ที่ชัดเจน

```vue
<!-- components/AccessibleButton.vue -->
<template>
  <button
    :aria-expanded="isOpen"
    :aria-controls="panelId"
    :aria-label="ariaLabel"
    :disabled="disabled"
    :aria-busy="loading"
    @click="toggle"
  >
    <span v-if="loading" aria-hidden="true" class="spinner"></span>
    <slot>{{ label }}</slot>
  </button>
</template>

<script setup lang="ts">
defineProps<{
  isOpen?: boolean
  panelId?: string
  ariaLabel?: string
  disabled?: boolean
  loading?: boolean
  label?: string
}>()

const emit = defineEmits(['toggle'])

const toggle = () => {
  emit('toggle')
}
</script>
```

```vue
<!-- components/AccessibleModal.vue -->
<template>
  <Teleport to="body">
    <div
      v-if="isOpen"
      role="dialog"
      :aria-modal="true"
      :aria-labelledby="titleId"
      :aria-describedby="descriptionId"
      class="modal-overlay"
      @click.self="closeOnOverlay && close()"
      @keydown.escape="close"
    >
      <div class="modal-content" ref="modalRef">
        <header class="modal-header">
          <h2 :id="titleId">{{ title }}</h2>
          <button
            type="button"
            :aria-label="`ปิด ${title}`"
            class="close-btn"
            @click="close"
          >
            <span aria-hidden="true">&times;</span>
          </button>
        </header>
        
        <div :id="descriptionId" class="modal-body">
          <slot></slot>
        </div>
        
        <footer class="modal-footer">
          <slot name="footer">
            <button type="button" @click="close">ปิด</button>
          </slot>
        </footer>
      </div>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { ref, watch, nextTick, onMounted, onUnmounted } from 'vue'
import { useFocusTrap } from '@/composables/useFocusTrap'

const props = defineProps<{
  isOpen: boolean
  title: string
  closeOnOverlay?: boolean
}>()

const emit = defineEmits(['close'])

const titleId = `modal-title-${Math.random().toString(36).substr(2, 9)}`
const descriptionId = `modal-desc-${Math.random().toString(36).substr(2, 9)}`
const modalRef = ref<HTMLElement | null>(null)

const { activate, deactivate } = useFocusTrap(modalRef)

watch(() => props.isOpen, async (newVal) => {
  if (newVal) {
    await nextTick()
    activate()
    // Announce to screen readers
    document.title = `${props.title} - Dialog Open`
  } else {
    deactivate()
  }
})

const close = () => {
  emit('close')
}
</script>
```

## Focus Management

การจัดการ focus อย่างถูกต้องเป็นสิ่งสำคัญมากสำหรับผู้ใช้ที่ navigate ด้วยแป้นพิมพ์

```typescript
// composables/useFocusTrap.ts
import { ref, onUnmounted } from 'vue'

export function useFocusTrap(containerRef: Ref<HTMLElement | null>) {
  const previousFocus = ref<HTMLElement | null>(null)
  
  const focusableElements = [
    'a[href]',
    'button:not([disabled])',
    'input:not([disabled])',
    'select:not([disabled])',
    'textarea:not([disabled])',
    '[tabindex]:not([tabindex="-1"])',
    'details > summary'
  ].join(', ')
  
  const getFocusableElements = () => {
    if (!containerRef.value) return []
    return Array.from(
      containerRef.value.querySelectorAll<HTMLElement>(focusableElements)
    ).filter(el => !el.closest('[hidden]'))
  }
  
  const handleKeyDown = (event: KeyboardEvent) => {
    if (event.key !== 'Tab') return
    
    const focusable = getFocusableElements()
    if (focusable.length === 0) return
    
    const firstElement = focusable[0]
    const lastElement = focusable[focusable.length - 1]
    
    if (event.shiftKey) {
      // Shift + Tab: ย้อนกลับ
      if (document.activeElement === firstElement) {
        event.preventDefault()
        lastElement.focus()
      }
    } else {
      // Tab: ไปข้างหน้า
      if (document.activeElement === lastElement) {
        event.preventDefault()
        firstElement.focus()
      }
    }
  }
  
  const activate = () => {
    previousFocus.value = document.activeElement as HTMLElement
    document.addEventListener('keydown', handleKeyDown)
    
    // Focus first element
    const focusable = getFocusableElements()
    if (focusable.length > 0) {
      focusable[0].focus()
    } else {
      containerRef.value?.focus()
    }
  }
  
  const deactivate = () => {
    document.removeEventListener('keydown', handleKeyDown)
    previousFocus.value?.focus()
    previousFocus.value = null
  }
  
  onUnmounted(() => {
    document.removeEventListener('keydown', handleKeyDown)
  })
  
  return { activate, deactivate }
}
```

```typescript
// composables/useFocusReturn.ts
import { ref } from 'vue'

export function useFocusReturn() {
  const savedFocus = ref<HTMLElement | null>(null)
  
  const saveFocus = () => {
    savedFocus.value = document.activeElement as HTMLElement
  }
  
  const restoreFocus = () => {
    if (savedFocus.value) {
      savedFocus.value.focus()
      savedFocus.value = null
    }
  }
  
  const focusElement = (selector: string) => {
    const element = document.querySelector<HTMLElement>(selector)
    if (element) {
      element.focus()
    }
  }
  
  return { saveFocus, restoreFocus, focusElement }
}
```

## Keyboard Navigation

```vue
<!-- components/AccessibleDropdown.vue -->
<template>
  <div class="dropdown" ref="dropdownRef">
    <button
      type="button"
      :id="buttonId"
      :aria-expanded="isOpen"
      :aria-haspopup="'listbox'"
      :aria-controls="listId"
      class="dropdown-trigger"
      @click="toggleDropdown"
      @keydown="handleTriggerKeydown"
    >
      {{ selectedItem?.label || placeholder }}
      <span aria-hidden="true" class="arrow">▼</span>
    </button>
    
    <ul
      v-show="isOpen"
      :id="listId"
      role="listbox"
      :aria-labelledby="buttonId"
      class="dropdown-list"
      ref="listRef"
    >
      <li
        v-for="(item, index) in items"
        :key="item.value"
        role="option"
        :aria-selected="selectedItem?.value === item.value"
        :class="{ focused: focusedIndex === index, selected: selectedItem?.value === item.value }"
        @click="selectItem(item)"
        @keydown="handleOptionKeydown($event, index)"
        tabindex="-1"
        :ref="el => setOptionRef(el as HTMLElement, index)"
      >
        {{ item.label }}
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface DropdownItem {
  value: string
  label: string
}

const props = defineProps<{
  items: DropdownItem[]
  modelValue?: string
  placeholder?: string
}>()

const emit = defineEmits(['update:modelValue'])

const dropdownRef = ref<HTMLElement | null>(null)
const listRef = ref<HTMLElement | null>(null)
const isOpen = ref(false)
const focusedIndex = ref(-1)
const optionRefs = ref<HTMLElement[]>([])

const buttonId = `dropdown-btn-${Math.random().toString(36).substr(2, 9)}`
const listId = `dropdown-list-${Math.random().toString(36).substr(2, 9)}`

const selectedItem = computed(() => 
  props.items.find(item => item.value === props.modelValue)
)

const setOptionRef = (el: HTMLElement, index: number) => {
  optionRefs.value[index] = el
}

const openDropdown = () => {
  isOpen.value = true
  const selectedIndex = props.items.findIndex(item => item.value === props.modelValue)
  focusedIndex.value = selectedIndex >= 0 ? selectedIndex : 0
}

const closeDropdown = () => {
  isOpen.value = false
  focusedIndex.value = -1
}

const toggleDropdown = () => {
  if (isOpen.value) {
    closeDropdown()
  } else {
    openDropdown()
  }
}

const selectItem = (item: DropdownItem) => {
  emit('update:modelValue', item.value)
  closeDropdown()
  // Return focus to trigger
  const trigger = dropdownRef.value?.querySelector<HTMLElement>('.dropdown-trigger')
  trigger?.focus()
}

const handleTriggerKeydown = (event: KeyboardEvent) => {
  switch (event.key) {
    case 'ArrowDown':
    case 'Enter':
    case ' ':
      event.preventDefault()
      if (!isOpen.value) {
        openDropdown()
      }
      break
    case 'ArrowUp':
      event.preventDefault()
      if (!isOpen.value) {
        openDropdown()
      }
      break
    case 'Escape':
      closeDropdown()
      break
  }
}

const handleOptionKeydown = (event: KeyboardEvent, index: number) => {
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault()
      if (index < props.items.length - 1) {
        focusedIndex.value = index + 1
        optionRefs.value[focusedIndex.value]?.focus()
      }
      break
    case 'ArrowUp':
      event.preventDefault()
      if (index > 0) {
        focusedIndex.value = index - 1
        optionRefs.value[focusedIndex.value]?.focus()
      }
      break
    case 'Enter':
    case ' ':
      event.preventDefault()
      selectItem(props.items[index])
      break
    case 'Escape':
      closeDropdown()
      break
    case 'Home':
      event.preventDefault()
      focusedIndex.value = 0
      optionRefs.value[0]?.focus()
      break
    case 'End':
      event.preventDefault()
      focusedIndex.value = props.items.length - 1
      optionRefs.value[focusedIndex.value]?.focus()
      break
  }
}

watch(focusedIndex, (newIndex) => {
  if (newIndex >= 0 && optionRefs.value[newIndex]) {
    optionRefs.value[newIndex].focus()
  }
})

// Close on outside click
const handleClickOutside = (event: MouseEvent) => {
  if (dropdownRef.value && !dropdownRef.value.contains(event.target as Node)) {
    closeDropdown()
  }
}

import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  document.addEventListener('click', handleClickOutside)
})

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside)
})
</script>
```

## Screen Reader Support

```vue
<!-- components/LiveRegion.vue -->
<template>
  <!-- Live region สำหรับแจ้งเตือน status changes -->
  <div class="sr-only">
    <div 
      role="status" 
      aria-live="polite" 
      aria-atomic="true"
      ref="statusRef"
    >
      {{ statusMessage }}
    </div>
    
    <div 
      role="alert" 
      aria-live="assertive" 
      aria-atomic="true"
      ref="alertRef"
    >
      {{ alertMessage }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const statusMessage = ref('')
const alertMessage = ref('')
const statusRef = ref<HTMLElement | null>(null)
const alertRef = ref<HTMLElement | null>(null)

const announce = (message: string, type: 'polite' | 'assertive' = 'polite') => {
  if (type === 'polite') {
    statusMessage.value = ''
    setTimeout(() => {
      statusMessage.value = message
    }, 100)
  } else {
    alertMessage.value = ''
    setTimeout(() => {
      alertMessage.value = message
    }, 100)
  }
}

defineExpose({ announce })
</script>

<style>
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
</style>
```

```typescript
// composables/useAnnouncer.ts
import { ref, onMounted, onUnmounted } from 'vue'

let announcer: HTMLElement | null = null

const createAnnouncer = () => {
  const el = document.createElement('div')
  el.setAttribute('role', 'status')
  el.setAttribute('aria-live', 'polite')
  el.setAttribute('aria-atomic', 'true')
  el.style.cssText = `
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  `
  document.body.appendChild(el)
  return el
}

export function useAnnouncer() {
  onMounted(() => {
    if (!announcer) {
      announcer = createAnnouncer()
    }
  })
  
  const announce = (message: string) => {
    if (!announcer) return
    
    announcer.textContent = ''
    setTimeout(() => {
      if (announcer) {
        announcer.textContent = message
      }
    }, 100)
  }
  
  return { announce }
}
```

## Color Contrast

ต้องมี contrast ratio ที่เพียงพอ: 4.5:1 สำหรับ text ปกติ และ 3:1 สำหรับ large text

```typescript
// utils/colorContrast.ts
export function getLuminance(r: number, g: number, b: number): number {
  const [rs, gs, bs] = [r, g, b].map(c => {
    c = c / 255
    return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4)
  })
  return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs
}

export function hexToRgb(hex: string): [number, number, number] {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
  if (!result) throw new Error('Invalid hex color')
  return [
    parseInt(result[1], 16),
    parseInt(result[2], 16),
    parseInt(result[3], 16)
  ]
}

export function getContrastRatio(color1: string, color2: string): number {
  const [r1, g1, b1] = hexToRgb(color1)
  const [r2, g2, b2] = hexToRgb(color2)
  
  const lum1 = getLuminance(r1, g1, b1)
  const lum2 = getLuminance(r2, g2, b2)
  
  const lighter = Math.max(lum1, lum2)
  const darker = Math.min(lum1, lum2)
  
  return (lighter + 0.05) / (darker + 0.05)
}

export function meetsWCAG(
  color1: string, 
  color2: string, 
  level: 'AA' | 'AAA' = 'AA',
  isLargeText = false
): boolean {
  const ratio = getContrastRatio(color1, color2)
  
  if (level === 'AA') {
    return isLargeText ? ratio >= 3 : ratio >= 4.5
  } else {
    return isLargeText ? ratio >= 4.5 : ratio >= 7
  }
}
```

## axe-core Testing

```typescript
// tests/accessibility/axe-setup.ts
import { mount } from '@vue/test-utils'
import { configureAxe, toHaveNoViolations } from 'jest-axe'
import { expect } from 'vitest'

expect.extend(toHaveNoViolations)

const axe = configureAxe({
  rules: {
    'color-contrast': { enabled: true },
    'document-title': { enabled: true },
    'html-has-lang': { enabled: true },
    'landmark-one-main': { enabled: true }
  }
})

export { axe }
export async function checkAccessibility(component: any, props?: Record<string, any>) {
  const wrapper = mount(component, { props })
  const results = await axe(wrapper.element)
  expect(results).toHaveNoViolations()
  return wrapper
}
```

```typescript
// tests/accessibility/AccessibleForm.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import { axe, toHaveNoViolations } from 'jest-axe'
import AccessibleForm from '@/components/AccessibleForm.vue'

expect.extend(toHaveNoViolations)

describe('AccessibleForm', () => {
  it('has no accessibility violations', async () => {
    const wrapper = mount(AccessibleForm, {
      global: {
        stubs: { RouterLink: true }
      }
    })
    
    const results = await axe(wrapper.element)
    expect(results).toHaveNoViolations()
  })
  
  it('has proper form labels', () => {
    const wrapper = mount(AccessibleForm)
    const inputs = wrapper.findAll('input')
    
    inputs.forEach(input => {
      const id = input.attributes('id')
      const label = wrapper.find(`label[for="${id}"]`)
      expect(label.exists()).toBe(true)
    })
  })
  
  it('announces errors to screen readers', async () => {
    const wrapper = mount(AccessibleForm)
    await wrapper.find('form').trigger('submit')
    
    const errorRegion = wrapper.find('[role="alert"]')
    expect(errorRegion.exists()).toBe(true)
  })
})
```

## ตัวอย่าง: Accessible Form

```vue
<!-- components/AccessibleForm.vue -->
<template>
  <form 
    @submit.prevent="handleSubmit"
    novalidate
    aria-label="แบบฟอร์มสมัครสมาชิก"
  >
    <!-- Error summary -->
    <div 
      v-if="Object.keys(errors).length > 0"
      role="alert"
      aria-live="assertive"
      class="error-summary"
      tabindex="-1"
      ref="errorSummaryRef"
    >
      <h2>กรุณาแก้ไขข้อผิดพลาดดังนี้:</h2>
      <ul>
        <li v-for="(error, field) in errors" :key="field">
          <a :href="`#${field}`">{{ error }}</a>
        </li>
      </ul>
    </div>
    
    <!-- Name field -->
    <div class="form-group">
      <label for="name" class="required">
        ชื่อ-นามสกุล
        <span aria-hidden="true" class="required-indicator">*</span>
      </label>
      <input
        id="name"
        v-model="form.name"
        type="text"
        :aria-required="true"
        :aria-invalid="!!errors.name"
        :aria-describedby="errors.name ? 'name-error' : 'name-hint'"
        autocomplete="name"
        @blur="validateField('name')"
      />
      <p id="name-hint" class="field-hint">
        กรอกชื่อ-นามสกุลจริงของคุณ
      </p>
      <p 
        v-if="errors.name" 
        id="name-error" 
        class="field-error"
        role="alert"
      >
        {{ errors.name }}
      </p>
    </div>
    
    <!-- Email field -->
    <div class="form-group">
      <label for="email" class="required">
        อีเมล
        <span aria-hidden="true" class="required-indicator">*</span>
      </label>
      <input
        id="email"
        v-model="form.email"
        type="email"
        :aria-required="true"
        :aria-invalid="!!errors.email"
        :aria-describedby="errors.email ? 'email-error' : undefined"
        autocomplete="email"
        @blur="validateField('email')"
      />
      <p 
        v-if="errors.email" 
        id="email-error" 
        class="field-error"
        role="alert"
      >
        {{ errors.email }}
      </p>
    </div>
    
    <!-- Password field -->
    <div class="form-group">
      <label for="password" class="required">
        รหัสผ่าน
        <span aria-hidden="true" class="required-indicator">*</span>
      </label>
      <div class="password-wrapper">
        <input
          id="password"
          v-model="form.password"
          :type="showPassword ? 'text' : 'password'"
          :aria-required="true"
          :aria-invalid="!!errors.password"
          :aria-describedby="['password-requirements', errors.password ? 'password-error' : ''].filter(Boolean).join(' ')"
          autocomplete="new-password"
          @blur="validateField('password')"
        />
        <button
          type="button"
          :aria-label="showPassword ? 'ซ่อนรหัสผ่าน' : 'แสดงรหัสผ่าน'"
          :aria-pressed="showPassword"
          class="toggle-password"
          @click="showPassword = !showPassword"
        >
          <span aria-hidden="true">{{ showPassword ? '🙈' : '👁️' }}</span>
        </button>
      </div>
      <ul id="password-requirements" class="requirements">
        <li :class="{ met: passwordChecks.length }">
          <span aria-hidden="true">{{ passwordChecks.length ? '✓' : '✗' }}</span>
          อย่างน้อย 8 ตัวอักษร
        </li>
        <li :class="{ met: passwordChecks.uppercase }">
          <span aria-hidden="true">{{ passwordChecks.uppercase ? '✓' : '✗' }}</span>
          มีตัวพิมพ์ใหญ่
        </li>
        <li :class="{ met: passwordChecks.number }">
          <span aria-hidden="true">{{ passwordChecks.number ? '✓' : '✗' }}</span>
          มีตัวเลข
        </li>
      </ul>
      <p 
        v-if="errors.password" 
        id="password-error" 
        class="field-error"
        role="alert"
      >
        {{ errors.password }}
      </p>
    </div>
    
    <!-- Terms checkbox -->
    <div class="form-group">
      <div class="checkbox-wrapper">
        <input
          id="terms"
          v-model="form.acceptTerms"
          type="checkbox"
          :aria-required="true"
          :aria-invalid="!!errors.acceptTerms"
          :aria-describedby="errors.acceptTerms ? 'terms-error' : undefined"
        />
        <label for="terms">
          ฉันยอมรับ
          <a href="/terms" target="_blank">
            ข้อกำหนดการใช้งาน
            <span class="sr-only">(เปิดในหน้าต่างใหม่)</span>
          </a>
        </label>
      </div>
      <p 
        v-if="errors.acceptTerms" 
        id="terms-error" 
        class="field-error"
        role="alert"
      >
        {{ errors.acceptTerms }}
      </p>
    </div>
    
    <!-- Submit button -->
    <button 
      type="submit"
      :disabled="isSubmitting"
      :aria-busy="isSubmitting"
      class="submit-btn"
    >
      <span v-if="isSubmitting">
        <span aria-hidden="true" class="spinner"></span>
        กำลังสมัครสมาชิก...
      </span>
      <span v-else>สมัครสมาชิก</span>
    </button>
  </form>
</template>

<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'

interface FormData {
  name: string
  email: string
  password: string
  acceptTerms: boolean
}

const form = ref<FormData>({
  name: '',
  email: '',
  password: '',
  acceptTerms: false
})

const errors = ref<Partial<Record<keyof FormData, string>>>({})
const isSubmitting = ref(false)
const showPassword = ref(false)
const errorSummaryRef = ref<HTMLElement | null>(null)

const passwordChecks = computed(() => ({
  length: form.value.password.length >= 8,
  uppercase: /[A-Z]/.test(form.value.password),
  number: /\d/.test(form.value.password)
}))

const validateField = (field: keyof FormData) => {
  switch (field) {
    case 'name':
      if (!form.value.name.trim()) {
        errors.value.name = 'กรุณากรอกชื่อ-นามสกุล'
      } else {
        delete errors.value.name
      }
      break
    case 'email':
      if (!form.value.email) {
        errors.value.email = 'กรุณากรอกอีเมล'
      } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.value.email)) {
        errors.value.email = 'รูปแบบอีเมลไม่ถูกต้อง'
      } else {
        delete errors.value.email
      }
      break
    case 'password':
      if (!form.value.password) {
        errors.value.password = 'กรุณากรอกรหัสผ่าน'
      } else if (form.value.password.length < 8) {
        errors.value.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
      } else {
        delete errors.value.password
      }
      break
    case 'acceptTerms':
      if (!form.value.acceptTerms) {
        errors.value.acceptTerms = 'กรุณายอมรับข้อกำหนดการใช้งาน'
      } else {
        delete errors.value.acceptTerms
      }
      break
  }
}

const handleSubmit = async () => {
  // Validate all fields
  ;(['name', 'email', 'password', 'acceptTerms'] as const).forEach(validateField)
  
  if (Object.keys(errors.value).length > 0) {
    await nextTick()
    errorSummaryRef.value?.focus()
    return
  }
  
  isSubmitting.value = true
  try {
    // Submit form...
    await new Promise(resolve => setTimeout(resolve, 1000))
    console.log('Form submitted:', form.value)
  } finally {
    isSubmitting.value = false
  }
}
</script>
```

## ตัวอย่าง: Accessible Navigation

```vue
<!-- components/AccessibleNavigation.vue -->
<template>
  <nav aria-label="เมนูหลัก">
    <button
      v-if="isMobile"
      type="button"
      :aria-expanded="isMenuOpen"
      aria-controls="main-menu"
      class="menu-toggle"
      @click="toggleMenu"
    >
      <span class="sr-only">{{ isMenuOpen ? 'ปิดเมนู' : 'เปิดเมนู' }}</span>
      <span aria-hidden="true" class="hamburger">☰</span>
    </button>
    
    <ul
      :id="mainMenuId"
      :hidden="isMobile && !isMenuOpen"
      class="nav-list"
    >
      <li v-for="item in navigationItems" :key="item.id">
        <!-- Item without children -->
        <RouterLink
          v-if="!item.children"
          :to="item.path"
          :aria-current="isCurrentRoute(item.path) ? 'page' : undefined"
          class="nav-link"
        >
          {{ item.label }}
        </RouterLink>
        
        <!-- Item with dropdown -->
        <template v-else>
          <button
            type="button"
            :aria-expanded="openSubmenu === item.id"
            :aria-controls="`submenu-${item.id}`"
            class="nav-link has-submenu"
            @click="toggleSubmenu(item.id)"
            @keydown="handleMenuKeydown($event, item.id)"
          >
            {{ item.label }}
            <span aria-hidden="true" class="arrow">▼</span>
          </button>
          
          <ul
            :id="`submenu-${item.id}`"
            v-show="openSubmenu === item.id"
            class="submenu"
            role="menu"
          >
            <li 
              v-for="child in item.children" 
              :key="child.id"
              role="none"
            >
              <RouterLink
                :to="child.path"
                role="menuitem"
                :aria-current="isCurrentRoute(child.path) ? 'page' : undefined"
                class="submenu-link"
                @keydown="handleSubmenuKeydown($event, item.id)"
              >
                {{ child.label }}
              </RouterLink>
            </li>
          </ul>
        </template>
      </li>
    </ul>
  </nav>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const isMenuOpen = ref(false)
const openSubmenu = ref<string | null>(null)
const mainMenuId = 'main-menu'

const isMobile = computed(() => window.innerWidth < 768)

const navigationItems = [
  { id: '1', label: 'หน้าแรก', path: '/' },
  { id: '2', label: 'บทเรียน', path: '/lessons', children: [
    { id: '2-1', label: 'Vue Basics', path: '/lessons/vue-basics' },
    { id: '2-2', label: 'Nuxt.js', path: '/lessons/nuxtjs' }
  ]},
  { id: '3', label: 'ติดต่อ', path: '/contact' }
]

const isCurrentRoute = (path: string) => route.path === path

const toggleMenu = () => {
  isMenuOpen.value = !isMenuOpen.value
}

const toggleSubmenu = (id: string) => {
  openSubmenu.value = openSubmenu.value === id ? null : id
}

const handleMenuKeydown = (event: KeyboardEvent, itemId: string) => {
  if (event.key === 'Escape') {
    openSubmenu.value = null
  }
}

const handleSubmenuKeydown = (event: KeyboardEvent, parentId: string) => {
  if (event.key === 'Escape') {
    openSubmenu.value = null
    // Return focus to parent button
    const parentBtn = document.querySelector<HTMLElement>(`[aria-controls="submenu-${parentId}"]`)
    parentBtn?.focus()
  }
}
</script>
```

## สรุป

การทำ Accessibility ที่ดีต้องใช้หลายเทคนิคร่วมกัน:
1. ใช้ Semantic HTML ให้ถูกต้อง
2. เพิ่ม ARIA attributes เมื่อจำเป็น
3. จัดการ keyboard navigation
4. ทดสอบด้วย Screen Reader จริง
5. ตรวจสอบ Color Contrast
6. ใช้ axe-core ในการ test อัตโนมัติ

ทรัพยากรเพิ่มเติม:
- [WCAG 2.1](https://www.w3.org/WAI/WCAG21/quickref/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/TR/wai-aria-practices-1.1/)
- [Vue A11y](https://vue-a11y.com/)
