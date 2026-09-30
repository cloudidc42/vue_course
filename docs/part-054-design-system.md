# Part 54: Design System กับ Vue.js

## Design System คืออะไร?

Design System คือชุดมาตรฐานของ design decisions ที่รวม design language, component library, และ guidelines เข้าด้วยกัน ช่วยให้ทีมสร้าง UI ที่สอดคล้องกันและเร็วขึ้น

## 1. Tokens (Colors, Typography, Spacing)

```scss
/* styles/tokens.css */
:root {
  /* ============ Colors ============ */
  /* Brand */
  --color-primary-50: #e3f2fd;
  --color-primary-100: #bbdefb;
  --color-primary-200: #90caf9;
  --color-primary-300: #64b5f6;
  --color-primary-400: #42a5f5;
  --color-primary-500: #2196f3;  /* Base */
  --color-primary-600: #1e88e5;
  --color-primary-700: #1976d2;
  --color-primary-800: #1565c0;
  --color-primary-900: #0d47a1;
  
  /* Semantic Colors */
  --color-success: #4caf50;
  --color-success-light: #c8e6c9;
  --color-error: #f44336;
  --color-error-light: #ffcdd2;
  --color-warning: #ff9800;
  --color-warning-light: #ffe0b2;
  --color-info: #2196f3;
  --color-info-light: #e3f2fd;
  
  /* Neutral */
  --color-gray-50: #fafafa;
  --color-gray-100: #f5f5f5;
  --color-gray-200: #eeeeee;
  --color-gray-300: #e0e0e0;
  --color-gray-400: #bdbdbd;
  --color-gray-500: #9e9e9e;
  --color-gray-600: #757575;
  --color-gray-700: #616161;
  --color-gray-800: #424242;
  --color-gray-900: #212121;
  
  /* Text */
  --color-text-primary: var(--color-gray-900);
  --color-text-secondary: var(--color-gray-600);
  --color-text-disabled: var(--color-gray-400);
  --color-text-inverse: #ffffff;
  
  /* Background */
  --color-bg-primary: #ffffff;
  --color-bg-secondary: var(--color-gray-50);
  --color-bg-elevated: #ffffff;
  
  /* Border */
  --color-border: var(--color-gray-200);
  --color-border-strong: var(--color-gray-400);
  
  /* ============ Typography ============ */
  --font-family-sans: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-family-mono: 'JetBrains Mono', 'Fira Code', monospace;
  
  /* Font Sizes */
  --text-xs: 0.75rem;    /* 12px */
  --text-sm: 0.875rem;   /* 14px */
  --text-base: 1rem;     /* 16px */
  --text-lg: 1.125rem;   /* 18px */
  --text-xl: 1.25rem;    /* 20px */
  --text-2xl: 1.5rem;    /* 24px */
  --text-3xl: 1.875rem;  /* 30px */
  --text-4xl: 2.25rem;   /* 36px */
  --text-5xl: 3rem;      /* 48px */
  
  /* Font Weights */
  --font-thin: 100;
  --font-light: 300;
  --font-normal: 400;
  --font-medium: 500;
  --font-semibold: 600;
  --font-bold: 700;
  --font-extrabold: 800;
  
  /* Line Heights */
  --leading-none: 1;
  --leading-tight: 1.25;
  --leading-snug: 1.375;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
  --leading-loose: 2;
  
  /* ============ Spacing ============ */
  --space-1: 0.25rem;   /* 4px */
  --space-2: 0.5rem;    /* 8px */
  --space-3: 0.75rem;   /* 12px */
  --space-4: 1rem;      /* 16px */
  --space-5: 1.25rem;   /* 20px */
  --space-6: 1.5rem;    /* 24px */
  --space-8: 2rem;      /* 32px */
  --space-10: 2.5rem;   /* 40px */
  --space-12: 3rem;     /* 48px */
  --space-16: 4rem;     /* 64px */
  --space-20: 5rem;     /* 80px */
  --space-24: 6rem;     /* 96px */
  
  /* ============ Border Radius ============ */
  --radius-none: 0;
  --radius-sm: 0.125rem;  /* 2px */
  --radius-base: 0.25rem; /* 4px */
  --radius-md: 0.375rem;  /* 6px */
  --radius-lg: 0.5rem;    /* 8px */
  --radius-xl: 0.75rem;   /* 12px */
  --radius-2xl: 1rem;     /* 16px */
  --radius-full: 9999px;
  
  /* ============ Shadows ============ */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-base: 0 1px 3px rgba(0, 0, 0, 0.1), 0 1px 2px rgba(0, 0, 0, 0.06);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.07), 0 2px 4px rgba(0, 0, 0, 0.05);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1), 0 4px 6px rgba(0, 0, 0, 0.05);
  --shadow-xl: 0 20px 25px rgba(0, 0, 0, 0.1), 0 10px 10px rgba(0, 0, 0, 0.04);
  --shadow-2xl: 0 25px 50px rgba(0, 0, 0, 0.25);
  
  /* ============ Transitions ============ */
  --transition-fast: 150ms ease;
  --transition-base: 200ms ease;
  --transition-slow: 300ms ease;
  
  /* ============ Z-Index ============ */
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-overlay: 300;
  --z-modal: 400;
  --z-toast: 500;
  --z-tooltip: 600;
}

/* Dark Mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text-primary: #f5f5f5;
    --color-text-secondary: #9e9e9e;
    --color-bg-primary: #121212;
    --color-bg-secondary: #1e1e1e;
    --color-bg-elevated: #2d2d2d;
    --color-border: #333;
  }
}
```

## 2. Component Library Architecture

```
design-system/
├── tokens/
│   ├── colors.ts
│   ├── typography.ts
│   ├── spacing.ts
│   └── index.ts
├── components/
│   ├── primitives/
│   │   ├── Button/
│   │   │   ├── Button.vue
│   │   │   ├── Button.test.ts
│   │   │   └── Button.stories.ts
│   │   ├── Input/
│   │   ├── Select/
│   │   └── Badge/
│   ├── composite/
│   │   ├── Card/
│   │   ├── Modal/
│   │   ├── Dropdown/
│   │   └── DataTable/
│   └── layout/
│       ├── Container/
│       ├── Grid/
│       └── Stack/
├── composables/
│   ├── useTheme.ts
│   └── useToast.ts
└── index.ts  # Main export
```

## 3. Button Component

```vue
<!-- components/ui/Button.vue -->
<template>
  <component
    :is="tag"
    class="btn"
    :class="[
      `btn--${variant}`,
      `btn--${size}`,
      {
        'btn--loading': loading,
        'btn--full': fullWidth,
        'btn--icon-only': iconOnly
      }
    ]"
    :disabled="disabled || loading"
    v-bind="$attrs"
  >
    <!-- Leading Icon -->
    <span v-if="$slots.icon || leadingIcon" class="btn__icon btn__icon--leading">
      <slot name="icon">
        <component :is="leadingIcon" v-if="leadingIcon" />
      </slot>
    </span>
    
    <!-- Loading Spinner -->
    <span v-if="loading" class="btn__spinner" aria-hidden="true">
      <svg viewBox="0 0 24 24" fill="none">
        <circle cx="12" cy="12" r="10" stroke="currentColor" stroke-width="3" stroke-dasharray="30" stroke-dashoffset="0"/>
      </svg>
    </span>
    
    <!-- Content -->
    <span v-if="!iconOnly" class="btn__content">
      <slot>{{ label }}</slot>
    </span>
    
    <!-- Trailing Icon -->
    <span v-if="$slots.trailingIcon || trailingIcon" class="btn__icon btn__icon--trailing">
      <slot name="trailingIcon">
        <component :is="trailingIcon" v-if="trailingIcon" />
      </slot>
    </span>
  </component>
</template>

<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger' | 'success'
  size?: 'xs' | 'sm' | 'md' | 'lg' | 'xl'
  tag?: string
  label?: string
  disabled?: boolean
  loading?: boolean
  fullWidth?: boolean
  iconOnly?: boolean
  leadingIcon?: any
  trailingIcon?: any
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  tag: 'button',
  disabled: false,
  loading: false,
  fullWidth: false,
  iconOnly: false
})
</script>

<style scoped>
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
  font-family: var(--font-family-sans);
  font-weight: var(--font-medium);
  border-radius: var(--radius-lg);
  border: 1px solid transparent;
  cursor: pointer;
  transition: all var(--transition-base);
  text-decoration: none;
  white-space: nowrap;
  user-select: none;
  
  &:focus-visible {
    outline: 2px solid var(--color-primary-500);
    outline-offset: 2px;
  }
  
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    pointer-events: none;
  }
}

/* Sizes */
.btn--xs {
  padding: var(--space-1) var(--space-2);
  font-size: var(--text-xs);
  min-height: 24px;
}

.btn--sm {
  padding: var(--space-1) var(--space-3);
  font-size: var(--text-sm);
  min-height: 32px;
}

.btn--md {
  padding: var(--space-2) var(--space-4);
  font-size: var(--text-base);
  min-height: 40px;
}

.btn--lg {
  padding: var(--space-3) var(--space-6);
  font-size: var(--text-lg);
  min-height: 48px;
}

.btn--xl {
  padding: var(--space-4) var(--space-8);
  font-size: var(--text-xl);
  min-height: 56px;
}

/* Variants */
.btn--primary {
  background: var(--color-primary-500);
  color: var(--color-text-inverse);
  
  &:hover:not(:disabled) {
    background: var(--color-primary-600);
  }
  
  &:active:not(:disabled) {
    background: var(--color-primary-700);
  }
}

.btn--secondary {
  background: var(--color-gray-100);
  color: var(--color-text-primary);
  border-color: var(--color-border);
  
  &:hover:not(:disabled) {
    background: var(--color-gray-200);
  }
}

.btn--outline {
  background: transparent;
  color: var(--color-primary-500);
  border-color: var(--color-primary-500);
  
  &:hover:not(:disabled) {
    background: var(--color-primary-50);
  }
}

.btn--ghost {
  background: transparent;
  color: var(--color-text-primary);
  
  &:hover:not(:disabled) {
    background: var(--color-gray-100);
  }
}

.btn--danger {
  background: var(--color-error);
  color: white;
  
  &:hover:not(:disabled) {
    background: #d32f2f;
  }
}

.btn--success {
  background: var(--color-success);
  color: white;
  
  &:hover:not(:disabled) {
    background: #388e3c;
  }
}

/* States */
.btn--loading {
  cursor: not-allowed;
  position: relative;
}

.btn--full {
  width: 100%;
}

.btn__spinner {
  width: 1em;
  height: 1em;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.btn__icon {
  display: inline-flex;
  align-items: center;
  width: 1.2em;
  height: 1.2em;
}
</style>
```

## 4. Dark Mode Theming

```typescript
// composables/useTheme.ts
export type ThemeMode = 'light' | 'dark' | 'system'

export const useTheme = () => {
  const colorMode = useColorMode()
  
  const mode = computed<ThemeMode>({
    get: () => colorMode.preference as ThemeMode,
    set: (value) => {
      colorMode.preference = value
    }
  })
  
  const isDark = computed(() => colorMode.value === 'dark')
  
  const toggle = () => {
    if (mode.value === 'light') mode.value = 'dark'
    else if (mode.value === 'dark') mode.value = 'system'
    else mode.value = 'light'
  }
  
  return { mode, isDark, toggle }
}
```

```vue
<!-- components/ui/ThemeToggle.vue -->
<template>
  <button @click="toggle" class="theme-toggle" :title="modeLabel">
    <span v-if="mode === 'light'">☀️</span>
    <span v-else-if="mode === 'dark'">🌙</span>
    <span v-else>💻</span>
    <span class="sr-only">{{ modeLabel }}</span>
  </button>
</template>

<script setup>
const { mode, toggle } = useTheme()

const modeLabel = computed(() => {
  const labels = { light: 'โหมดสว่าง', dark: 'โหมดมืด', system: 'ระบบ' }
  return labels[mode.value]
})
</script>

<style scoped>
.theme-toggle {
  background: none;
  border: none;
  cursor: pointer;
  padding: 8px;
  border-radius: var(--radius-lg);
  font-size: 1.2rem;
  transition: background var(--transition-base);
}

.theme-toggle:hover {
  background: var(--color-gray-100);
}
</style>
```

## 5. ตัวอย่าง: Mini Design System

```vue
<!-- pages/design-system.vue -->
<template>
  <div class="ds-page">
    <header class="ds-header">
      <h1>Design System</h1>
      <ThemeToggle />
    </header>
    
    <!-- Colors -->
    <section class="ds-section">
      <h2>Colors</h2>
      <div class="color-grid">
        <div v-for="(color, name) in colors" :key="name" class="color-swatch">
          <div class="swatch" :style="{ background: color }"></div>
          <div class="swatch-info">
            <span class="swatch-name">{{ name }}</span>
            <span class="swatch-value">{{ color }}</span>
          </div>
        </div>
      </div>
    </section>
    
    <!-- Typography -->
    <section class="ds-section">
      <h2>Typography</h2>
      <div class="type-scale">
        <div v-for="size in typeSizes" :key="size.name" class="type-sample">
          <p :style="{ fontSize: size.value }">
            สวัสดี Vue.js ({{ size.name }} - {{ size.value }})
          </p>
        </div>
      </div>
    </section>
    
    <!-- Buttons -->
    <section class="ds-section">
      <h2>Buttons</h2>
      
      <div class="component-showcase">
        <h3>Variants</h3>
        <div class="button-row">
          <Button v-for="variant in buttonVariants" :key="variant" :variant="variant">
            {{ variant }}
          </Button>
        </div>
        
        <h3>Sizes</h3>
        <div class="button-row">
          <Button v-for="size in buttonSizes" :key="size" :size="size">
            {{ size }}
          </Button>
        </div>
        
        <h3>States</h3>
        <div class="button-row">
          <Button>Normal</Button>
          <Button :loading="true">Loading</Button>
          <Button :disabled="true">Disabled</Button>
        </div>
      </div>
    </section>
    
    <!-- Form Elements -->
    <section class="ds-section">
      <h2>Form Elements</h2>
      <div class="form-showcase">
        <Input label="Text Input" placeholder="กรอกข้อความ" />
        <Input label="Error State" placeholder="กรอกข้อความ" :error="'กรุณากรอกข้อมูล'" />
        <Select label="Select" :options="selectOptions" />
        <Checkbox label="Checkbox option" />
        <RadioGroup :options="radioOptions" v-model="selectedRadio" />
        <Textarea label="Textarea" placeholder="กรอกข้อความยาวๆ" />
      </div>
    </section>
  </div>
</template>

<script setup>
const colors = {
  'primary-500': '#2196f3',
  'primary-700': '#1976d2',
  'success': '#4caf50',
  'error': '#f44336',
  'warning': '#ff9800',
  'gray-200': '#eeeeee',
  'gray-600': '#757575',
  'gray-900': '#212121'
}

const typeSizes = [
  { name: 'xs', value: 'var(--text-xs)' },
  { name: 'sm', value: 'var(--text-sm)' },
  { name: 'base', value: 'var(--text-base)' },
  { name: 'lg', value: 'var(--text-lg)' },
  { name: 'xl', value: 'var(--text-xl)' },
  { name: '2xl', value: 'var(--text-2xl)' },
  { name: '3xl', value: 'var(--text-3xl)' }
]

const buttonVariants = ['primary', 'secondary', 'outline', 'ghost', 'danger', 'success']
const buttonSizes = ['xs', 'sm', 'md', 'lg', 'xl']

const selectOptions = [
  { value: 'option1', label: 'ตัวเลือก 1' },
  { value: 'option2', label: 'ตัวเลือก 2' }
]

const radioOptions = [
  { value: 'a', label: 'ตัวเลือก A' },
  { value: 'b', label: 'ตัวเลือก B' }
]

const selectedRadio = ref('a')
</script>

<style scoped>
.ds-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: var(--space-8);
}

.ds-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-12);
  padding-bottom: var(--space-6);
  border-bottom: 1px solid var(--color-border);
}

.ds-section {
  margin-bottom: var(--space-16);
}

.ds-section h2 {
  font-size: var(--text-2xl);
  font-weight: var(--font-semibold);
  margin-bottom: var(--space-6);
  padding-bottom: var(--space-2);
  border-bottom: 2px solid var(--color-primary-500);
}

.color-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: var(--space-4);
}

.color-swatch {
  border-radius: var(--radius-xl);
  overflow: hidden;
  border: 1px solid var(--color-border);
}

.swatch {
  height: 80px;
}

.swatch-info {
  padding: var(--space-2) var(--space-3);
  background: var(--color-bg-elevated);
}

.swatch-name {
  display: block;
  font-size: var(--text-sm);
  font-weight: var(--font-medium);
}

.swatch-value {
  display: block;
  font-size: var(--text-xs);
  color: var(--color-text-secondary);
  font-family: var(--font-family-mono);
}

.button-row {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-3);
  margin-bottom: var(--space-6);
  align-items: center;
}

.form-showcase {
  max-width: 400px;
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
}
</style>
```

## 7. Animation Tokens

```css
/* assets/css/tokens/animations.css */
:root {
  /* Duration */
  --duration-fast: 100ms;
  --duration-normal: 200ms;
  --duration-slow: 300ms;
  --duration-slower: 500ms;

  /* Easing */
  --ease-linear: linear;
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-bounce: cubic-bezier(0.34, 1.56, 0.64, 1);

  /* Common transitions */
  --transition-base: all var(--duration-normal) var(--ease-in-out);
  --transition-colors: color var(--duration-fast) var(--ease-in-out),
                       background-color var(--duration-fast) var(--ease-in-out),
                       border-color var(--duration-fast) var(--ease-in-out);
  --transition-transform: transform var(--duration-normal) var(--ease-out);
  --transition-opacity: opacity var(--duration-normal) var(--ease-in-out);
}
```

## 8. Design Token Documentation Component

```vue
<!-- components/design-system/TokenDoc.vue -->
<template>
  <section class="token-doc">
    <h2>{{ title }}</h2>
    <div class="token-grid">
      <div
        v-for="token in tokens"
        :key="token.name"
        class="token-item"
      >
        <!-- Color preview -->
        <div
          v-if="type === 'color'"
          class="color-swatch"
          :style="{ background: `var(${token.name})` }"
        />
        <!-- Spacing preview -->
        <div
          v-else-if="type === 'spacing'"
          class="spacing-swatch"
          :style="{ width: `var(${token.name})`, height: '16px', background: 'var(--color-primary-500)' }"
        />
        <div class="token-info">
          <code class="token-name">{{ token.name }}</code>
          <span class="token-value">{{ token.value }}</span>
          <span v-if="token.description" class="token-desc">{{ token.description }}</span>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
interface Token {
  name: string
  value: string
  description?: string
}

defineProps<{
  title: string
  tokens: Token[]
  type: 'color' | 'spacing' | 'typography' | 'other'
}>()
</script>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Design Tokens** - Colors, Typography, Spacing แบบ CSS Variables
2. **Component Architecture** - โครงสร้างที่ scalable
3. **Button Component** - Component ที่มี variants, sizes, states
4. **Dark Mode** - Theming system ด้วย @nuxtjs/color-mode
5. **Design System Page** - Showcase ทุก components
6. **CSS Variables** - ใช้ tokens ใน components
7. **Animation Tokens** - Duration, Easing, Transition tokens
8. **Token Documentation** - Component สำหรับ document design tokens
