# Part 15: Custom Directives

## บทนำ

Directives ใน Vue.js คือ special attributes ที่ขึ้นต้นด้วย `v-` ใช้สำหรับ manipulate DOM โดยตรง Custom Directives ช่วยให้เรา reuse DOM manipulation logic ได้

---

## 1. Built-in Directives ทบทวน

```vue
<template>
  <div>
    <!-- v-bind: ผูก attribute -->
    <img :src="imageUrl" :alt="title" />
    
    <!-- v-model: two-way binding -->
    <input v-model="text" />
    
    <!-- v-if / v-else-if / v-else: conditional rendering -->
    <p v-if="count > 10">มากกว่า 10</p>
    <p v-else-if="count > 5">มากกว่า 5</p>
    <p v-else>น้อยกว่าหรือเท่ากับ 5</p>
    
    <!-- v-show: toggle visibility -->
    <p v-show="isVisible">แสดง/ซ่อน</p>
    
    <!-- v-for: loop rendering -->
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
    
    <!-- v-on: event listener -->
    <button v-on:click="handleClick" @click.prevent="submit">คลิก</button>
    
    <!-- v-text / v-html -->
    <p v-text="message"></p>
    <div v-html="htmlContent"></div>
    
    <!-- v-once: render ครั้งเดียว -->
    <p v-once>{{ staticText }}</p>
    
    <!-- v-pre: ไม่ compile template -->
    <p v-pre>{{ ข้อความนี้จะไม่ถูก compile }}</p>
    
    <!-- v-memo: cache rendering -->
    <div v-memo="[count]">{{ heavyComputation }}</div>
  </div>
</template>
```

---

## 2. Custom Directive สร้างเอง

### Directive Object Format

```javascript
const myDirective = {
  // Called before the element's attributes or child component
  created(el, binding, vnode) {},
  
  // Called right before the element is inserted into the DOM
  beforeMount(el, binding, vnode) {},
  
  // Called when the element is inserted into the DOM
  mounted(el, binding, vnode) {},
  
  // Called before the parent component is updated
  beforeUpdate(el, binding, vnode, prevVnode) {},
  
  // Called after the parent component and all its children have updated
  updated(el, binding, vnode, prevVnode) {},
  
  // Called before the parent component is unmounted
  beforeUnmount(el, binding, vnode) {},
  
  // Called when the parent component is unmounted
  unmounted(el, binding, vnode) {}
}
```

### Binding Object

```javascript
// binding มี properties ดังนี้:
{
  value,      // ค่าที่ส่งมา: v-directive="value"
  oldValue,   // ค่าก่อนหน้า (ใน beforeUpdate/updated เท่านั้น)
  arg,        // argument: v-directive:arg
  modifiers,  // modifiers: v-directive.mod1.mod2 → { mod1: true, mod2: true }
  instance,   // component instance
  dir         // directive definition object
}
```

### การ Register Directive

```javascript
// main.js - Global directive
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Global directive
app.directive('focus', {
  mounted(el) {
    el.focus()
  }
})

app.mount('#app')
```

```vue
<!-- Local directive ใน component -->
<script setup>
// ใช้ camelCase ใน script, kebab-case ใน template
const vFocus = {
  mounted(el) {
    el.focus()
  }
}
</script>

<template>
  <!-- ใน template ใช้ v-focus -->
  <input v-focus type="text" />
</template>
```

---

## 3. v-focus

Directive สำหรับ auto focus element

```javascript
// directives/focus.js
export const vFocus = {
  // mounted เรียกเมื่อ element ถูก insert เข้า DOM แล้ว
  mounted(el, binding) {
    // ถ้า binding.value ไม่ใช่ false ให้ focus
    if (binding.value !== false) {
      // delay เล็กน้อยเพื่อให้ transition เสร็จก่อน
      const delay = binding.arg ? parseInt(binding.arg) : 0
      
      if (delay) {
        setTimeout(() => el.focus(), delay)
      } else {
        el.focus()
      }
    }
  }
}
```

```vue
<!-- การใช้งาน -->
<template>
  <div>
    <!-- Auto focus ทันที -->
    <input v-focus placeholder="focus ทันที" />
    
    <!-- Focus หลัง 300ms (delay ผ่าน arg) -->
    <input v-focus:300 placeholder="focus หลัง 300ms" />
    
    <!-- Conditional focus -->
    <input v-focus="shouldFocus" placeholder="focus ถ้า shouldFocus = true" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { vFocus } from '../directives/focus.js'

const shouldFocus = ref(true)
</script>
```

---

## 4. v-click-outside

Directive สำหรับ detect การคลิกนอก element

```javascript
// directives/clickOutside.js
export const vClickOutside = {
  mounted(el, binding) {
    // บันทึก handler ไว้ใน element เพื่อลบออกใน unmounted
    el._clickOutsideHandler = (event) => {
      // ตรวจสอบว่า click อยู่นอก element หรือไม่
      if (!el.contains(event.target)) {
        // เรียก handler ที่ส่งมา
        if (typeof binding.value === 'function') {
          binding.value(event)
        }
      }
    }
    
    // ใช้ mousedown แทน click เพื่อ detect เร็วกว่า
    document.addEventListener('mousedown', el._clickOutsideHandler)
  },
  
  unmounted(el) {
    // ลบ event listener เมื่อ directive unmount
    if (el._clickOutsideHandler) {
      document.removeEventListener('mousedown', el._clickOutsideHandler)
      delete el._clickOutsideHandler
    }
  }
}
```

### การใช้งาน v-click-outside

```vue
<!-- components/Dropdown.vue -->
<template>
  <div class="dropdown" v-click-outside="closeDropdown">
    <button @click="toggleDropdown" class="dropdown-trigger">
      {{ selectedItem?.label || 'เลือก...' }}
      <span>{{ isOpen ? '▲' : '▼' }}</span>
    </button>
    
    <div v-if="isOpen" class="dropdown-menu">
      <button
        v-for="item in items"
        :key="item.value"
        @click="selectItem(item)"
        :class="['dropdown-item', { active: selectedItem?.value === item.value }]"
      >
        {{ item.label }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { vClickOutside } from '../directives/clickOutside.js'

const props = defineProps({
  items: Array,
  modelValue: [String, Number]
})

const emit = defineEmits(['update:modelValue'])

const isOpen = ref(false)
const selectedItem = ref(null)

function toggleDropdown() {
  isOpen.value = !isOpen.value
}

function closeDropdown() {
  isOpen.value = false
}

function selectItem(item) {
  selectedItem.value = item
  emit('update:modelValue', item.value)
  closeDropdown()
}
</script>
```

---

## 5. v-tooltip

Directive สำหรับแสดง tooltip

```javascript
// directives/tooltip.js
let tooltipElement = null

export const vTooltip = {
  mounted(el, binding) {
    const options = typeof binding.value === 'string'
      ? { content: binding.value }
      : binding.value
    
    const {
      content = '',
      placement = binding.arg || 'top',
      delay = 200
    } = options
    
    let showTimer = null
    let hideTimer = null
    
    function createTooltip() {
      tooltipElement = document.createElement('div')
      tooltipElement.className = `tooltip tooltip--${placement}`
      tooltipElement.textContent = content
      document.body.appendChild(tooltipElement)
      
      // Position tooltip
      positionTooltip(el, tooltipElement, placement)
    }
    
    function showTooltip() {
      if (hideTimer) clearTimeout(hideTimer)
      showTimer = setTimeout(() => {
        if (!tooltipElement) createTooltip()
        tooltipElement.style.opacity = '1'
        tooltipElement.style.visibility = 'visible'
      }, delay)
    }
    
    function hideTooltip() {
      if (showTimer) clearTimeout(showTimer)
      hideTimer = setTimeout(() => {
        if (tooltipElement) {
          tooltipElement.style.opacity = '0'
          tooltipElement.style.visibility = 'hidden'
        }
      }, 100)
    }
    
    el._tooltipShow = showTooltip
    el._tooltipHide = hideTooltip
    
    el.addEventListener('mouseenter', showTooltip)
    el.addEventListener('mouseleave', hideTooltip)
    el.addEventListener('focus', showTooltip)
    el.addEventListener('blur', hideTooltip)
  },
  
  updated(el, binding) {
    if (tooltipElement) {
      const options = typeof binding.value === 'string'
        ? { content: binding.value }
        : binding.value
      tooltipElement.textContent = options.content || binding.value
    }
  },
  
  unmounted(el) {
    el.removeEventListener('mouseenter', el._tooltipShow)
    el.removeEventListener('mouseleave', el._tooltipHide)
    el.removeEventListener('focus', el._tooltipShow)
    el.removeEventListener('blur', el._tooltipHide)
    
    if (tooltipElement) {
      tooltipElement.remove()
      tooltipElement = null
    }
  }
}

function positionTooltip(target, tooltip, placement) {
  const targetRect = target.getBoundingClientRect()
  const tooltipRect = tooltip.getBoundingClientRect()
  
  const positions = {
    top: {
      top: targetRect.top - tooltipRect.height - 8 + window.scrollY,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2 + window.scrollX
    },
    bottom: {
      top: targetRect.bottom + 8 + window.scrollY,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2 + window.scrollX
    },
    left: {
      top: targetRect.top + (targetRect.height - tooltipRect.height) / 2 + window.scrollY,
      left: targetRect.left - tooltipRect.width - 8 + window.scrollX
    },
    right: {
      top: targetRect.top + (targetRect.height - tooltipRect.height) / 2 + window.scrollY,
      left: targetRect.right + 8 + window.scrollX
    }
  }
  
  const pos = positions[placement] || positions.top
  tooltip.style.top = pos.top + 'px'
  tooltip.style.left = pos.left + 'px'
}
```

### CSS สำหรับ Tooltip

```css
/* styles/tooltip.css */
.tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 12px;
  white-space: nowrap;
  pointer-events: none;
  z-index: 9999;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.2s;
}

.tooltip::before {
  content: '';
  position: absolute;
  border: 5px solid transparent;
}

.tooltip--top::before {
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  border-top-color: rgba(0, 0, 0, 0.8);
}

.tooltip--bottom::before {
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  border-bottom-color: rgba(0, 0, 0, 0.8);
}
```

### การใช้งาน v-tooltip

```vue
<template>
  <div>
    <!-- Tooltip แบบ string -->
    <button v-tooltip="'คลิกเพื่อบันทึก'">💾</button>
    
    <!-- Tooltip พร้อม options -->
    <button v-tooltip="{ content: 'ลบข้อมูล', placement: 'bottom' }">🗑️</button>
    
    <!-- Tooltip ด้วย arg -->
    <button v-tooltip:right="'ข้อมูลเพิ่มเติม'">ℹ️</button>
    
    <!-- Dynamic tooltip -->
    <span v-tooltip="dynamicTooltip">Hover เพื่อดู</span>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { vTooltip } from '../directives/tooltip.js'

const userRole = ref('admin')

const dynamicTooltip = computed(() => {
  return userRole.value === 'admin'
    ? 'คุณมีสิทธิ์ Admin'
    : 'คุณเป็นผู้ใช้ทั่วไป'
})
</script>
```

---

## 6. v-lazy-load

Directive สำหรับ lazy loading images

```javascript
// directives/lazyLoad.js
const defaultOptions = {
  loading: '/images/placeholder.svg',
  error: '/images/error.svg',
  threshold: 0.1 // 10% ของ element ต้องอยู่ใน viewport
}

export const vLazyLoad = {
  mounted(el, binding) {
    const options = { ...defaultOptions, ...(typeof binding.value === 'object' ? binding.value : {}) }
    const imageSrc = typeof binding.value === 'string' ? binding.value : binding.value?.src
    
    // แสดง placeholder ก่อน
    el.src = options.loading
    el.classList.add('lazy-loading')
    
    // สร้าง IntersectionObserver
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            loadImage(el, imageSrc, options)
            observer.unobserve(el)
          }
        })
      },
      { threshold: options.threshold }
    )
    
    observer.observe(el)
    el._lazyObserver = observer
  },
  
  updated(el, binding) {
    const imageSrc = typeof binding.value === 'string' ? binding.value : binding.value?.src
    
    if (imageSrc !== el._currentSrc) {
      const options = { ...defaultOptions, ...(typeof binding.value === 'object' ? binding.value : {}) }
      loadImage(el, imageSrc, options)
    }
  },
  
  unmounted(el) {
    if (el._lazyObserver) {
      el._lazyObserver.unobserve(el)
      el._lazyObserver.disconnect()
    }
  }
}

function loadImage(el, src, options) {
  el._currentSrc = src
  el.classList.add('lazy-loading')
  el.classList.remove('lazy-loaded', 'lazy-error')
  
  const img = new Image()
  
  img.onload = () => {
    el.src = src
    el.classList.remove('lazy-loading')
    el.classList.add('lazy-loaded')
  }
  
  img.onerror = () => {
    el.src = options.error
    el.classList.remove('lazy-loading')
    el.classList.add('lazy-error')
  }
  
  img.src = src
}
```

```vue
<!-- การใช้งาน -->
<template>
  <div class="image-grid">
    <!-- Lazy load แบบ simple -->
    <img
      v-lazy-load="'/images/photo-1.jpg'"
      alt="ภาพ 1"
      class="grid-image"
    />
    
    <!-- Lazy load พร้อม options -->
    <img
      v-lazy-load="{
        src: product.imageUrl,
        loading: '/images/skeleton.gif',
        error: '/images/no-image.png'
      }"
      :alt="product.name"
    />
  </div>
</template>

<style>
.lazy-loading {
  opacity: 0.5;
  filter: blur(4px);
  transition: all 0.3s;
}

.lazy-loaded {
  opacity: 1;
  filter: none;
}

.lazy-error {
  opacity: 0.7;
}
</style>
```

---

## 7. v-permission

Directive สำหรับ role-based access control

```javascript
// directives/permission.js
import { useAuth } from '../composables/useAuth.js'

export const vPermission = {
  mounted(el, binding) {
    checkPermission(el, binding)
  },
  
  updated(el, binding) {
    checkPermission(el, binding)
  }
}

function checkPermission(el, binding) {
  // inject auth context (ต้อง call นอก composable)
  const { auth } = useAuth()
  
  const required = binding.value
  const modifier = binding.modifiers
  
  let hasPermission = false
  
  if (Array.isArray(required)) {
    // ต้องมีทุก permission (AND)
    if (modifier.all) {
      hasPermission = required.every(perm => 
        auth.user?.permissions?.includes(perm)
      )
    } else {
      // มีอย่างน้อยหนึ่ง permission (OR)
      hasPermission = required.some(perm =>
        auth.user?.permissions?.includes(perm)
      )
    }
  } else if (typeof required === 'string') {
    // ตรวจสอบ role
    if (binding.arg === 'role') {
      hasPermission = auth.user?.role === required
    } else {
      hasPermission = auth.user?.permissions?.includes(required) || false
    }
  }
  
  if (!hasPermission) {
    // ซ่อน element หรือ remove ออก
    if (binding.modifiers.hide) {
      el.style.display = 'none'
    } else {
      el.parentNode?.removeChild(el)
    }
  }
}
```

### การใช้งาน v-permission

```vue
<template>
  <div>
    <!-- แสดงเฉพาะผู้ที่มี permission 'create:post' -->
    <button v-permission="'create:post'" @click="createPost">
      สร้างโพสต์ใหม่
    </button>
    
    <!-- แสดงเฉพาะ admin role -->
    <div v-permission:role="'admin'">
      <h3>พื้นที่ Admin</h3>
      <!-- admin content -->
    </div>
    
    <!-- ต้องมีทั้ง 'edit:post' และ 'delete:post' -->
    <div v-permission.all="['edit:post', 'delete:post']">
      <button>แก้ไข</button>
      <button>ลบ</button>
    </div>
    
    <!-- ซ่อนแทนที่จะ remove (ใช้ v-if เพิ่ม) -->
    <button v-permission.hide="'delete:user'">ลบผู้ใช้</button>
  </div>
</template>
```

---

## 8. v-copy

Directive สำหรับ copy text ง่ายๆ

```javascript
// directives/copy.js
export const vCopy = {
  mounted(el, binding) {
    el.style.cursor = 'pointer'
    el._copyHandler = async () => {
      const text = binding.value || el.textContent
      
      try {
        await navigator.clipboard.writeText(text)
        
        // Visual feedback
        const originalText = el.textContent
        const originalTitle = el.title
        
        el.textContent = binding.arg === 'icon' ? '✅' : '✅ คัดลอกแล้ว!'
        el.classList.add('copied')
        
        if (typeof binding.modifiers !== 'undefined' && binding.modifiers.notify) {
          showNotification('คัดลอกแล้ว!')
        }
        
        setTimeout(() => {
          if (binding.arg === 'icon') {
            el.textContent = '📋'
          } else {
            el.textContent = originalText
          }
          el.title = originalTitle
          el.classList.remove('copied')
        }, 2000)
        
      } catch (err) {
        console.error('Copy failed:', err)
      }
    }
    
    el.addEventListener('click', el._copyHandler)
    
    // เพิ่ม title attribute
    el.title = 'คลิกเพื่อคัดลอก'
  },
  
  updated(el, binding) {
    // อัปเดตค่าที่จะ copy เมื่อ binding เปลี่ยน
    el._copyValue = binding.value
  },
  
  unmounted(el) {
    el.removeEventListener('click', el._copyHandler)
  }
}

function showNotification(message) {
  const notification = document.createElement('div')
  notification.textContent = message
  notification.style.cssText = `
    position: fixed;
    bottom: 20px;
    right: 20px;
    background: #4CAF50;
    color: white;
    padding: 10px 20px;
    border-radius: 4px;
    z-index: 9999;
    animation: slideIn 0.3s ease;
  `
  document.body.appendChild(notification)
  setTimeout(() => notification.remove(), 2000)
}
```

### การใช้งาน v-copy

```vue
<template>
  <div>
    <!-- คัดลอก text ของ element -->
    <code v-copy>npm install vue@next</code>
    
    <!-- คัดลอกค่าที่กำหนด -->
    <button v-copy="'https://vuejs.org'">คัดลอก URL</button>
    
    <!-- คัดลอกพร้อม notification -->
    <span v-copy.notify="secretKey">คลิกเพื่อคัดลอก API Key</span>
    
    <!-- คัดลอกโค้ดแบบ icon -->
    <div class="code-block">
      <pre>{{ codeExample }}</pre>
      <span v-copy:icon="codeExample">📋</span>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { vCopy } from '../directives/copy.js'

const secretKey = ref('sk-12345-abcdef')
const codeExample = ref(`const app = createApp(App)
app.mount('#app')`)
</script>
```

---

## 9. Plugin: Register ทุก Directives พร้อมกัน

```javascript
// directives/index.js
import { vFocus } from './focus.js'
import { vClickOutside } from './clickOutside.js'
import { vTooltip } from './tooltip.js'
import { vLazyLoad } from './lazyLoad.js'
import { vPermission } from './permission.js'
import { vCopy } from './copy.js'

// สร้าง Plugin
export const DirectivesPlugin = {
  install(app) {
    app.directive('focus', vFocus)
    app.directive('click-outside', vClickOutside)
    app.directive('tooltip', vTooltip)
    app.directive('lazy-load', vLazyLoad)
    app.directive('permission', vPermission)
    app.directive('copy', vCopy)
    
    console.log('✅ Custom directives registered')
  }
}
```

```javascript
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import { DirectivesPlugin } from './directives/index.js'

const app = createApp(App)
app.use(DirectivesPlugin)
app.mount('#app')
```

---

## สรุป

| Directive | ใช้สำหรับ | Hooks ที่ใช้ |
|-----------|-----------|-------------|
| `v-focus` | Auto focus input | `mounted` |
| `v-click-outside` | Detect click นอก element | `mounted`, `unmounted` |
| `v-tooltip` | แสดง tooltip | `mounted`, `updated`, `unmounted` |
| `v-lazy-load` | Lazy load images | `mounted`, `updated`, `unmounted` |
| `v-permission` | Access control | `mounted`, `updated` |
| `v-copy` | Copy to clipboard | `mounted`, `updated`, `unmounted` |

### Best Practices

```javascript
// ✅ ทำ cleanup ใน unmounted เสมอ
const vMyDirective = {
  mounted(el, binding) {
    el._handler = () => { /* ... */ }
    document.addEventListener('click', el._handler)
  },
  unmounted(el) {
    document.removeEventListener('click', el._handler)
    delete el._handler
  }
}

// ✅ ใช้ el._xxx สำหรับเก็บ state ใน element
el._timeout = setTimeout(() => {}, 1000)
el._observer = new IntersectionObserver(...)
```
