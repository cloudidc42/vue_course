# Part 81: Vue 3 Compiler Optimization

## Vue Compiler Architecture

Vue 3 มี compiler ที่แปลง template เป็น optimized render functions ก่อนถึง runtime ทำให้ application ทำงานเร็วขึ้นมาก

---

## 1. Vue Compiler Architecture

### Compilation Pipeline

```
Template String
      ↓
   Parsing (AST)
      ↓
   Transform
      ↓
   Code Generation
      ↓
Render Function (JavaScript)
```

### Template → Render Function

```html
<!-- Template -->
<div class="container">
  <h1>{{ title }}</h1>
  <p v-if="show">Hello</p>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </ul>
</div>
```

```javascript
// Generated render function (simplified)
import { createElementVNode as _createElementVNode, toDisplayString as _toDisplayString, renderList as _renderList, Fragment as _Fragment, openBlock as _openBlock, createElementBlock as _createElementBlock, createCommentVNode as _createCommentVNode, normalizeClass as _normalizeClass } from "vue"

// Static content ถูก hoist ออกไป
const _hoisted_1 = { class: "container" }

export function render(_ctx, _cache) {
  return (
    _openBlock(),
    _createElementBlock("div", _hoisted_1, [
      _createElementVNode("h1", null, _toDisplayString(_ctx.title), 1 /* TEXT */),
      _ctx.show
        ? _createElementVNode("p", null, "Hello")
        : _createCommentVNode("v-if", true),
      _createElementVNode("ul", null, [
        (_openBlock(true), _createElementBlock(_Fragment, null,
          _renderList(_ctx.items, (item) => {
            return _openBlock(), _createElementBlock("li", { key: item.id },
              _toDisplayString(item.name), 1 /* TEXT */
            )
          }), 128 /* KEYED_FRAGMENT */
        ))
      ])
    ])
  )
}
```

---

## 2. Static Hoisting

### อะไรคือ Static Hoisting?

```javascript
// ❌ ไม่มี hoisting - สร้างใหม่ทุก render
export function render() {
  return createElementVNode('div', null, [
    createElementVNode('h1', { class: 'title' }, 'Hello World'), // สร้างใหม่ทุกครั้ง!
    createElementVNode('p', null, _ctx.dynamicText)
  ])
}

// ✅ มี hoisting - สร้างครั้งเดียว
const _hoisted_1 = createElementVNode('h1', { class: 'title' }, 'Hello World')

export function render() {
  return createElementVNode('div', null, [
    _hoisted_1, // ใช้ reference เดิม!
    createElementVNode('p', null, _ctx.dynamicText)
  ])
}
```

### เงื่อนไขสำหรับ Hoisting

```vue
<template>
  <!-- ✅ Static - จะถูก hoist -->
  <div class="header">
    <img src="/logo.png" alt="Logo" />
    <span>My App</span>
  </div>
  
  <!-- ❌ Dynamic - ไม่ถูก hoist -->
  <div :class="dynamicClass">
    <span>{{ message }}</span>
  </div>
</template>
```

---

## 3. Patch Flags

Patch flags บอก Vue runtime ว่า node ไหนมีส่วนที่เปลี่ยนแปลงได้ เพื่อ skip การ diff ที่ไม่จำเป็น

```typescript
// Patch Flags constants
export const enum PatchFlags {
  TEXT = 1,           // Dynamic text content
  CLASS = 2,          // Dynamic class
  STYLE = 4,          // Dynamic style
  PROPS = 8,          // Dynamic props (ที่ไม่ใช่ class/style)
  FULL_PROPS = 16,    // Dynamic element with dynamic key props
  NEED_HYDRATION = 32,// An element that needs event listeners to be attached by vue
  STABLE_FRAGMENT = 64,
  KEYED_FRAGMENT = 128,
  UNKEYED_FRAGMENT = 256,
  NEED_PATCH = 512,   // An element that only needs non-props patching
  DYNAMIC_SLOTS = 1024,
  HOISTED = -1,       // Static - ไม่ต้อง diff
  BAIL = -2           // Indication to always diff full children
}
```

### ตัวอย่าง Patch Flags

```vue
<template>
  <!-- TEXT flag = 1 -->
  <span>{{ message }}</span>

  <!-- CLASS flag = 2 -->
  <div :class="className">static text</div>

  <!-- STYLE flag = 4 -->
  <div :style="styleObj">static text</div>

  <!-- PROPS flag = 8 (with tracked props) -->
  <Component :title="title" static-prop="value" />
  
  <!-- TEXT | CLASS = 3 (combined flags) -->
  <div :class="cls">{{ text }}</div>
</template>
```

---

## 4. Tree Shaking

### Vue 3 Tree Shaking Design

```javascript
// Vue 3 - Named imports (tree-shakeable)
import { ref, computed, watch } from 'vue'
// Build tool จะ include เฉพาะ functions ที่ใช้

// Vue 2 - Global Vue (ไม่ tree-shakeable)
import Vue from 'vue'
Vue.component('...') // ต้อง include ทั้ง Vue
```

### ตรวจสอบ Bundle

```javascript
// vite.config.ts
import { defineConfig } from 'vite'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    visualizer({
      open: true,
      filename: 'dist/stats.html',
      gzipSize: true,
      brotliSize: true
    })
  ]
})
```

---

## 5. Compile-time Optimizations

### v-once (Compile-time static)

```vue
<template>
  <!-- render ครั้งเดียว ไม่ update -->
  <div v-once>
    {{ expensiveComputation() }}
  </div>

  <!-- ดีกว่าใช้ useMemo ใน React -->
</template>
```

### v-memo

```vue
<template>
  <!-- Re-render เฉพาะเมื่อ item.id หรือ selected เปลี่ยน -->
  <div v-for="item in list" :key="item.id" v-memo="[item.id, selected === item.id]">
    <p>ID: {{ item.id }}</p>
    <p>Name: {{ item.name }}</p>
    <p>Selected: {{ selected === item.id }}</p>
  </div>
</template>
```

---

## 6. Custom Transforms

### สร้าง Vue Compiler Plugin

```typescript
// vite-plugin-vue-transform.ts
import type { Plugin } from 'vite'
import type { NodeTransform } from '@vue/compiler-core'
import { NodeTypes, createSimpleExpression } from '@vue/compiler-core'

// Transform ที่เพิ่ม data-testid อัตโนมัติจาก @test attribute
const addTestIdTransform: NodeTransform = (node) => {
  if (node.type !== NodeTypes.ELEMENT) return

  const testAttr = node.props.find(
    p => p.type === NodeTypes.ATTRIBUTE && p.name === '@test'
  )

  if (!testAttr) return

  // ลบ @test attribute
  node.props = node.props.filter(p => p !== testAttr)

  // เพิ่ม data-testid
  if (testAttr.type === NodeTypes.ATTRIBUTE && testAttr.value) {
    node.props.push({
      type: NodeTypes.ATTRIBUTE,
      name: 'data-testid',
      value: testAttr.value,
      loc: testAttr.loc
    })
  }
}

export function vueTestIdPlugin(): Plugin {
  return {
    name: 'vue-test-id-transform',
    enforce: 'pre',

    config(config) {
      // เพิ่ม transform ให้ Vue compiler
      return {
        ...config,
        plugins: [
          ...(config.plugins || []),
          {
            compilerOptions: {
              nodeTransforms: [
                process.env.NODE_ENV === 'test' ? addTestIdTransform : null
              ].filter(Boolean)
            }
          }
        ]
      }
    }
  }
}
```

### Custom Directive Compiler

```typescript
// compiler-plugins/lazy-load-transform.ts
import { NodeTransform, NodeTypes, createSimpleExpression } from '@vue/compiler-core'

// Transform v-lazy directive เป็น intersection observer code
export const lazyLoadTransform: NodeTransform = (node, context) => {
  if (node.type !== NodeTypes.ELEMENT) return
  if (node.tag !== 'img') return

  const lazyAttr = node.props.find(
    p => p.type === NodeTypes.DIRECTIVE && p.name === 'lazy'
  )

  if (!lazyAttr) return

  // แปลง :src เป็น data-src
  const srcAttr = node.props.find(
    p => p.type === NodeTypes.DIRECTIVE && p.name === 'bind' && 
         p.arg?.type === NodeTypes.SIMPLE_EXPRESSION && p.arg.content === 'src'
  )

  if (srcAttr && srcAttr.type === NodeTypes.DIRECTIVE) {
    // เปลี่ยน :src เป็น :data-src
    srcAttr.arg = createSimpleExpression('data-src', true)
  }

  // ลบ v-lazy directive
  node.props = node.props.filter(p => p !== lazyAttr)

  // เพิ่ม class สำหรับ JS lazy loader
  node.props.push({
    type: NodeTypes.ATTRIBUTE,
    name: 'class',
    value: { type: NodeTypes.TEXT, content: 'lazy-image' },
    loc: node.loc
  })
}
```

---

## 7. ตัวอย่าง: Writing Compiler Plugins

### Performance Monitoring Plugin

```typescript
// vite-plugin-component-perf.ts
import type { Plugin } from 'vite'
import { NodeTypes, NodeTransform } from '@vue/compiler-core'

const DEV_ONLY_TRANSFORM: NodeTransform = (node, context) => {
  if (node.type !== NodeTypes.ELEMENT) return
  if (!node.tag.includes('-') && node.tag[0] !== node.tag[0].toUpperCase()) return

  // ใน development mode เท่านั้น
  if (process.env.NODE_ENV !== 'development') return

  // เพิ่ม performance mark
  return () => {
    // This runs after children are processed
    const componentName = node.tag
    
    // Wrap component in performance tracking
    // Implementation depends on what you need to track
  }
}

export function componentPerfPlugin(): Plugin {
  return {
    name: 'component-perf',
    apply: 'build',

    transform(code, id) {
      if (!id.endsWith('.vue')) return
      if (process.env.NODE_ENV === 'production') return

      // Add performance timing to setup functions
      return {
        code: code.replace(
          /setup\(props, ctx\) \{/g,
          `setup(props, ctx) {
  const __perfStart = performance.now();
  const __cleanup = () => {
    const duration = performance.now() - __perfStart;
    if (duration > 16) console.warn('[Perf] Slow component setup:', duration.toFixed(2) + 'ms');
  };
  `
        ),
        map: null
      }
    }
  }
}
```

### Build Analysis

```typescript
// analyze-bundle.ts
import { build } from 'vite'
import { visualizer } from 'rollup-plugin-visualizer'

async function analyzeBundle() {
  const result = await build({
    plugins: [
      visualizer({
        open: false,
        filename: 'bundle-analysis.html',
        template: 'treemap',
        gzipSize: true,
        brotliSize: true
      })
    ],
    build: {
      rollupOptions: {
        output: {
          manualChunks(id) {
            // Analyze which chunk each module goes into
            if (id.includes('node_modules')) {
              if (id.includes('vue')) return 'vue'
              if (id.includes('pinia')) return 'pinia'
              return 'vendor'
            }
          }
        }
      }
    }
  })

  console.log('Bundle analysis complete: bundle-analysis.html')
}
```

---

## สรุป

Vue 3 Compiler Optimization ช่วยให้:
1. **Static hoisting** - ไม่ต้องสร้าง static nodes ซ้ำ
2. **Patch flags** - Skip unnecessary diffing
3. **Tree shaking** - Bundle เฉพาะ code ที่ใช้จริง
4. **v-memo / v-once** - Manual memoization
5. **Custom transforms** - Extend compiler behavior
