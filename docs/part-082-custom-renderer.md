# Part 82: Custom Renderer

## Vue Renderer API คืออะไร?

Vue 3 แยก core runtime ออกจาก DOM-specific code ทำให้เราสามารถสร้าง custom renderer ที่ render Vue components ไปยัง environment อื่นๆ เช่น Canvas, Native, Terminal ได้

---

## 1. Vue Renderer API

### Core Concepts

```typescript
// createRenderer API
import { createRenderer } from '@vue/runtime-core'

const { render, createApp } = createRenderer<Node, Element>({
  // Host Config - บอก Vue วิธีจัดการ elements ใน target environment
  patchProp,         // จัดการ props/attributes
  insert,            // เพิ่ม element ลงใน parent
  remove,            // ลบ element
  createElement,     // สร้าง element ใหม่
  createText,        // สร้าง text node
  createComment,     // สร้าง comment node
  setText,           // อัปเดต text content
  setElementText,    // อัปเดต element text
  parentNode,        // ได้ parent element
  nextSibling,       // ได้ next sibling
  querySelector,     // DOM query (optional)
  setScopeId,        // Scoped CSS support (optional)
  cloneNode,         // Clone node (optional)
  insertStaticContent // Insert static HTML (optional)
})
```

---

## 2. สร้าง Custom Renderer

### Base Renderer Template

```typescript
// renderers/base-renderer.ts
import { createRenderer, RendererOptions } from '@vue/runtime-core'

type HostNode = any
type HostElement = any

export function createCustomRenderer(
  rendererOptions: RendererOptions<HostNode, HostElement>
) {
  const { render, createApp } = createRenderer<HostNode, HostElement>(rendererOptions)

  return { render, createApp }
}
```

---

## 3. Vue กับ Canvas Renderer

### Canvas Renderer Implementation

```typescript
// renderers/canvas-renderer.ts
import { createRenderer } from '@vue/runtime-core'

// ประเภทของ "elements" บน Canvas
type CanvasElementType = 'rect' | 'circle' | 'text' | 'group' | 'image' | 'line'

interface CanvasElement {
  type: CanvasElementType
  props: Record<string, any>
  children: CanvasElement[]
  parent: CanvasElement | null
  _canvas?: HTMLCanvasElement
  _ctx?: CanvasRenderingContext2D
}

// Root container
interface CanvasRoot extends CanvasElement {
  type: 'group'
  _canvas: HTMLCanvasElement
  _ctx: CanvasRenderingContext2D
  _dirty: boolean
  _rafId?: number
}

function createCanvasElement(type: CanvasElementType): CanvasElement {
  return {
    type,
    props: {},
    children: [],
    parent: null
  }
}

function renderCanvas(root: CanvasRoot) {
  const { _ctx: ctx, _canvas: canvas } = root

  // Clear canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // Render all elements
  renderElement(ctx, root)
}

function renderElement(ctx: CanvasRenderingContext2D, element: CanvasElement) {
  ctx.save()

  // Apply transforms
  const { x = 0, y = 0, rotation = 0, scaleX = 1, scaleY = 1, opacity = 1 } = element.props

  ctx.globalAlpha = opacity
  ctx.translate(x, y)
  ctx.rotate((rotation * Math.PI) / 180)
  ctx.scale(scaleX, scaleY)

  switch (element.type) {
    case 'rect':
      drawRect(ctx, element.props)
      break
    case 'circle':
      drawCircle(ctx, element.props)
      break
    case 'text':
      drawText(ctx, element.props)
      break
    case 'image':
      drawImage(ctx, element.props)
      break
    case 'line':
      drawLine(ctx, element.props)
      break
    case 'group':
      // Groups just apply transforms to children
      break
  }

  // Render children
  for (const child of element.children) {
    renderElement(ctx, child)
  }

  ctx.restore()
}

function drawRect(ctx: CanvasRenderingContext2D, props: Record<string, any>) {
  const { width = 100, height = 50, fill = 'black', stroke, strokeWidth = 1, radius = 0 } = props

  ctx.beginPath()
  if (radius > 0) {
    ctx.roundRect(-width/2, -height/2, width, height, radius)
  } else {
    ctx.rect(-width/2, -height/2, width, height)
  }

  if (fill) {
    ctx.fillStyle = fill
    ctx.fill()
  }
  if (stroke) {
    ctx.strokeStyle = stroke
    ctx.lineWidth = strokeWidth
    ctx.stroke()
  }
}

function drawCircle(ctx: CanvasRenderingContext2D, props: Record<string, any>) {
  const { radius = 50, fill = 'black', stroke, strokeWidth = 1 } = props

  ctx.beginPath()
  ctx.arc(0, 0, radius, 0, Math.PI * 2)

  if (fill) {
    ctx.fillStyle = fill
    ctx.fill()
  }
  if (stroke) {
    ctx.strokeStyle = stroke
    ctx.lineWidth = strokeWidth
    ctx.stroke()
  }
}

function drawText(ctx: CanvasRenderingContext2D, props: Record<string, any>) {
  const {
    text = '',
    fill = 'black',
    fontSize = 16,
    fontFamily = 'Arial',
    fontWeight = 'normal',
    align = 'center',
    baseline = 'middle'
  } = props

  ctx.font = `${fontWeight} ${fontSize}px ${fontFamily}`
  ctx.fillStyle = fill
  ctx.textAlign = align as CanvasTextAlign
  ctx.textBaseline = baseline as CanvasTextBaseline
  ctx.fillText(text, 0, 0)
}

function drawLine(ctx: CanvasRenderingContext2D, props: Record<string, any>) {
  const { x1 = 0, y1 = 0, x2 = 100, y2 = 0, stroke = 'black', strokeWidth = 1, dash } = props

  ctx.beginPath()
  ctx.moveTo(x1, y1)
  ctx.lineTo(x2, y2)
  ctx.strokeStyle = stroke
  ctx.lineWidth = strokeWidth
  if (dash) ctx.setLineDash(dash)
  ctx.stroke()
}

function drawImage(ctx: CanvasRenderingContext2D, props: Record<string, any>) {
  const { src, width = 100, height = 100 } = props
  const img = new Image()
  img.src = src
  ctx.drawImage(img, -width/2, -height/2, width, height)
}

// Canvas Renderer
export const { render: canvasRender, createApp: createCanvasApp } = createRenderer<
  CanvasElement,
  CanvasElement
>({
  createElement(type) {
    return createCanvasElement(type as CanvasElementType)
  },

  createText(text) {
    return {
      type: 'text',
      props: { text },
      children: [],
      parent: null
    }
  },

  createComment() {
    return createCanvasElement('group') // Canvas ไม่มี comments
  },

  insert(child, parent, anchor) {
    const index = anchor ? parent.children.indexOf(anchor) : -1
    if (index === -1) {
      parent.children.push(child)
    } else {
      parent.children.splice(index, 0, child)
    }
    child.parent = parent

    // Trigger re-render
    const root = findRoot(parent)
    if (root) scheduleRender(root as CanvasRoot)
  },

  remove(child) {
    if (child.parent) {
      const index = child.parent.children.indexOf(child)
      if (index !== -1) child.parent.children.splice(index, 1)
      child.parent = null

      const root = findRoot(child) 
      if (root) scheduleRender(root as CanvasRoot)
    }
  },

  patchProp(el, key, prevValue, nextValue) {
    el.props[key] = nextValue

    const root = findRoot(el)
    if (root) scheduleRender(root as CanvasRoot)
  },

  setText(node, text) {
    node.props.text = text
    const root = findRoot(node)
    if (root) scheduleRender(root as CanvasRoot)
  },

  setElementText(el, text) {
    el.props.text = text
  },

  parentNode(node) {
    return node.parent
  },

  nextSibling(node) {
    if (!node.parent) return null
    const index = node.parent.children.indexOf(node)
    return node.parent.children[index + 1] || null
  }
})

function findRoot(element: CanvasElement): CanvasRoot | null {
  let current: CanvasElement | null = element
  while (current) {
    if ('_canvas' in current) return current as CanvasRoot
    current = current.parent
  }
  return null
}

function scheduleRender(root: CanvasRoot) {
  if (root._rafId) cancelAnimationFrame(root._rafId)
  root._rafId = requestAnimationFrame(() => {
    renderCanvas(root)
    root._rafId = undefined
  })
}
```

### Vue Canvas Components

```vue
<!-- components/canvas/CanvasRect.vue -->
<script setup lang="ts">
defineProps<{
  x?: number
  y?: number
  width?: number
  height?: number
  fill?: string
  stroke?: string
  radius?: number
  opacity?: number
}>()
</script>

<template>
  <rect v-bind="$props" />
</template>

<!-- components/canvas/CanvasCircle.vue -->
<script setup lang="ts">
defineProps<{
  x?: number
  y?: number
  radius?: number
  fill?: string
  stroke?: string
}>()
</script>

<template>
  <circle v-bind="$props" />
</template>
```

### Canvas App Entry

```typescript
// canvas-app.ts
import { createCanvasApp } from './renderers/canvas-renderer'
import App from './App.vue'

export function mountCanvasApp(canvas: HTMLCanvasElement) {
  const ctx = canvas.getContext('2d')!

  // สร้าง root container
  const root = {
    type: 'group' as const,
    props: {},
    children: [],
    parent: null,
    _canvas: canvas,
    _ctx: ctx,
    _dirty: false
  }

  const app = createCanvasApp(App)
  app.mount(root as any)

  return app
}
```

---

## 4. Terminal Renderer

### Ink-like Terminal Renderer

```typescript
// renderers/terminal-renderer.ts
import { createRenderer } from '@vue/runtime-core'

interface TerminalNode {
  type: string
  text?: string
  style?: {
    color?: string
    bold?: boolean
    underline?: boolean
  }
  children: TerminalNode[]
}

function renderToString(node: TerminalNode): string {
  if (node.type === 'text') return node.text || ''

  let result = ''
  const { style = {} } = node

  let prefix = ''
  let suffix = ''

  if (style.bold) { prefix += '\x1b[1m'; suffix = '\x1b[0m' + suffix }
  if (style.underline) { prefix += '\x1b[4m'; suffix = '\x1b[0m' + suffix }
  if (style.color) {
    const colors: Record<string, string> = {
      red: '\x1b[31m', green: '\x1b[32m',
      yellow: '\x1b[33m', blue: '\x1b[34m',
      cyan: '\x1b[36m', white: '\x1b[37m'
    }
    prefix += colors[style.color] || ''
    suffix = '\x1b[0m' + suffix
  }

  result += prefix
  result += node.children.map(renderToString).join('')
  result += suffix

  if (node.type === 'box') result += '\n'

  return result
}

export const { createApp: createTerminalApp } = createRenderer<TerminalNode, TerminalNode>({
  createElement(type) {
    return { type, children: [] }
  },
  createText(text) {
    return { type: 'text', text, children: [] }
  },
  createComment() {
    return { type: 'comment', children: [] }
  },
  insert(child, parent) {
    parent.children.push(child)
    // Re-render terminal
    process.stdout.write('\x1bc') // Clear screen
    console.log(renderToString(parent))
  },
  remove(child) {
    // implementation
  },
  patchProp(el, key, _, value) {
    if (key === 'style') el.style = value
    else (el as any)[key] = value
  },
  setText(node, text) { node.text = text },
  setElementText(el, text) { el.text = text },
  parentNode(node) { return null },
  nextSibling(node) { return null }
})
```

---

## 5. ตัวอย่าง: Canvas App

```vue
<!-- App.vue (Canvas) -->
<template>
  <group>
    <!-- Background -->
    <CanvasRect
      :x="400" :y="300"
      :width="800" :height="600"
      fill="#1a1a2e"
    />

    <!-- Animated circles -->
    <CanvasCircle
      v-for="particle in particles"
      :key="particle.id"
      :x="particle.x"
      :y="particle.y"
      :radius="particle.radius"
      :fill="particle.color"
      :opacity="particle.opacity"
    />

    <!-- Title -->
    <canvas-text
      :x="400" :y="60"
      text="Vue Canvas Demo"
      fill="white"
      :fontSize="32"
      fontWeight="bold"
    />
  </group>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const particles = ref<Array<{
  id: number
  x: number
  y: number
  radius: number
  color: string
  opacity: number
  vx: number
  vy: number
}>>([])

// Generate initial particles
for (let i = 0; i < 50; i++) {
  particles.value.push({
    id: i,
    x: Math.random() * 800,
    y: Math.random() * 600,
    radius: Math.random() * 5 + 2,
    color: `hsl(${Math.random() * 360}, 70%, 60%)`,
    opacity: Math.random() * 0.8 + 0.2,
    vx: (Math.random() - 0.5) * 2,
    vy: (Math.random() - 0.5) * 2
  })
}

let animationId: number

function animate() {
  particles.value = particles.value.map(p => ({
    ...p,
    x: (p.x + p.vx + 800) % 800,
    y: (p.y + p.vy + 600) % 600
  }))
  animationId = requestAnimationFrame(animate)
}

onMounted(() => animate())
onUnmounted(() => cancelAnimationFrame(animationId))
</script>
```

---

## สรุป

Custom Renderer ช่วยให้:
1. **ใช้ Vue ได้ทุก environment** - Canvas, Terminal, Native, WebGL
2. **Reuse business logic** - Components เหมือนกัน ต่างแค่ renderer
3. **Reactive updates** - Vue's reactivity ทำงานได้ทุก renderer
4. **Familiar API** - Developer ที่รู้ Vue แล้วสามารถใช้ได้ทันที
