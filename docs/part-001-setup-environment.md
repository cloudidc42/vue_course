# Part 001: การติดตั้งและสภาพแวดล้อม (Setup & Environment)

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** 2-3 ชั่วโมง | **ขั้นตอน:** 1-30

---

## 🎯 สิ่งที่จะได้เรียนรู้

- ติดตั้ง Node.js และเครื่องมือที่จำเป็น
- สร้าง Vue.js Project ด้วย Vite
- เข้าใจโครงสร้างโปรเจกต์
- ตั้งค่า VS Code สำหรับการพัฒนา Vue.js
- รัน Development Server ครั้งแรก

---

## ขั้นตอนที่ 1: ติดตั้ง Node.js

### 1.1 ดาวน์โหลด Node.js

Node.js คือ JavaScript Runtime ที่จำเป็นสำหรับการพัฒนา Vue.js

```bash
# ตรวจสอบว่ามี Node.js อยู่แล้วหรือไม่
node --version
npm --version
```

ถ้าไม่มี ให้ไปดาวน์โหลดที่ https://nodejs.org (แนะนำ LTS version)

### 1.2 ติดตั้ง NVM (Node Version Manager) - แนะนำ

NVM ช่วยให้เราจัดการ Node.js หลายเวอร์ชันได้

**สำหรับ macOS / Linux:**
```bash
# ติดตั้ง nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Reload shell configuration
source ~/.bashrc  # หรือ source ~/.zshrc

# ติดตั้ง Node.js เวอร์ชันล่าสุด LTS
nvm install --lts
nvm use --lts

# ตรวจสอบเวอร์ชัน
node --version   # ควรเป็น v18+ หรือสูงกว่า
npm --version    # ควรเป็น v9+
```

**สำหรับ Windows:**
```powershell
# ดาวน์โหลด nvm-windows จาก GitHub
# https://github.com/coreybutler/nvm-windows

# หลังติดตั้งแล้ว เปิด Command Prompt ใหม่
nvm install lts
nvm use lts
```

### 1.3 ติดตั้ง Package Manager เพิ่มเติม

```bash
# ติดตั้ง pnpm (แนะนำ - เร็วกว่าและประหยัดพื้นที่)
npm install -g pnpm

# หรือติดตั้ง yarn
npm install -g yarn

# ตรวจสอบ
pnpm --version
```

---

## ขั้นตอนที่ 2: ติดตั้ง VS Code และ Extensions

### 2.1 ติดตั้ง VS Code

ดาวน์โหลดจาก https://code.visualstudio.com

### 2.2 Extensions ที่จำเป็นสำหรับ Vue.js

เปิด VS Code แล้วไปที่ Extensions (Ctrl+Shift+X) แล้วติดตั้ง:

**Extensions ที่จำเป็น:**
```
1. Vue - Official (ชื่อเดิม: Volar)
   Publisher: Vue
   ID: Vue.volar
   
2. TypeScript Vue Plugin (Volar)
   Publisher: Vue
   ID: Vue.vscode-typescript-vue-plugin

3. ESLint
   Publisher: Microsoft
   ID: dbaeumer.vscode-eslint

4. Prettier - Code formatter
   Publisher: Prettier
   ID: esbenp.prettier-vscode
```

**Extensions ที่แนะนำ:**
```
5. Auto Import - ES6, TS, JSX, TSX
   ID: NuclleCode.auto-import

6. Vue VSCode Snippets
   ID: sdras.vue-vscode-snippets

7. GitLens
   ID: eamodio.gitlens

8. Tailwind CSS IntelliSense
   ID: bradlc.vscode-tailwindcss

9. Error Lens
   ID: usernamehw.errorlens

10. Path Intellisense
    ID: christian-kohler.path-intellisense
```

### 2.3 ตั้งค่า VS Code Settings

เปิด Settings (Ctrl+Shift+P → "Open User Settings JSON"):

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[vue]": {
    "editor.defaultFormatter": "Vue.volar"
  },
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.wordWrap": "on",
  "files.autoSave": "onFocusChange",
  "explorer.confirmDelete": false,
  "typescript.preferences.importModuleSpecifier": "relative",
  "vue.inlayHints.missingProps": true,
  "vue.inlayHints.optionsWrapper": true
}
```

---

## ขั้นตอนที่ 3: สร้าง Vue.js Project แรก

### 3.1 สร้าง Project ด้วย Vite

```bash
# วิธีที่ 1: ใช้ create-vue (Official)
npm create vue@latest my-first-vue-app

# จะถามคำถามหลายข้อ:
# ✔ Project name: my-first-vue-app
# ✔ Add TypeScript? › Yes
# ✔ Add JSX Support? › No
# ✔ Add Vue Router for Single Page Application development? › Yes
# ✔ Add Pinia for state management? › Yes
# ✔ Add Vitest for Unit Testing? › Yes
# ✔ Add an End-to-End Testing Solution? › Playwright
# ✔ Add ESLint for code quality? › Yes
# ✔ Add Prettier for code formatting? › Yes
# ✔ Add Vue DevTools 7 extension for debugging? › Yes
```

```bash
# วิธีที่ 2: ใช้ Vite โดยตรง
npm create vite@latest my-vue-app -- --template vue

# หรือด้วย TypeScript
npm create vite@latest my-vue-app -- --template vue-ts
```

### 3.2 เข้า Directory และติดตั้ง Dependencies

```bash
cd my-first-vue-app

# ติดตั้ง dependencies
npm install
# หรือ
pnpm install
# หรือ
yarn install
```

### 3.3 รัน Development Server

```bash
npm run dev
# หรือ
pnpm dev
```

คุณจะเห็นข้อความประมาณนี้:
```
  VITE v5.0.0  ready in 423 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help
```

เปิด Browser แล้วไปที่ `http://localhost:5173` คุณจะเห็นหน้าเว็บ Vue.js แรกของคุณ!

---

## ขั้นตอนที่ 4: ทำความเข้าใจโครงสร้างโปรเจกต์

### 4.1 โครงสร้างไฟล์

```
my-first-vue-app/
├── public/                  # Static assets (favicon, etc.)
│   └── favicon.ico
├── src/                     # Source code ทั้งหมดอยู่ที่นี่
│   ├── assets/              # Images, fonts, styles
│   │   └── main.css
│   ├── components/          # Vue Components
│   │   └── HelloWorld.vue
│   ├── router/              # Vue Router configuration
│   │   └── index.ts
│   ├── stores/              # Pinia stores
│   │   └── counter.ts
│   ├── views/               # Page Components
│   │   ├── HomeView.vue
│   │   └── AboutView.vue
│   ├── App.vue              # Root Component
│   └── main.ts              # Entry Point
├── index.html               # HTML Template
├── vite.config.ts           # Vite Configuration
├── tsconfig.json            # TypeScript Configuration
├── package.json             # Dependencies
└── .eslintrc.cjs            # ESLint Configuration
```

### 4.2 ไฟล์สำคัญ

**`src/main.ts` - Entry Point:**
```typescript
import './assets/main.css'

import { createApp } from 'vue'
import { createPinia } from 'pinia'

import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

**`src/App.vue` - Root Component:**
```vue
<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />

      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>
    </div>
  </header>

  <RouterView />
</template>
```

---

## ขั้นตอนที่ 5: สร้างโปรเจกต์ Vue.js อย่างง่าย

มาเริ่มเขียนโค้ดแรกกัน! เราจะสร้าง Counter App อย่างง่าย

### 5.1 แก้ไข App.vue

แทนที่เนื้อหาใน `src/App.vue` ด้วย:

```vue
<script setup lang="ts">
import { ref } from 'vue'

// สร้างตัวแปร reactive
const count = ref(0)
const message = ref('สวัสดี Vue.js!')

// Function สำหรับเพิ่มค่า
function increment() {
  count.value++
}

// Function สำหรับลดค่า
function decrement() {
  count.value--
}

// Function สำหรับ Reset
function reset() {
  count.value = 0
}
</script>

<template>
  <div class="app">
    <h1>{{ message }}</h1>
    
    <div class="counter">
      <p class="count">{{ count }}</p>
      
      <div class="buttons">
        <button @click="decrement" :disabled="count <= 0">-</button>
        <button @click="reset">Reset</button>
        <button @click="increment">+</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
  font-family: Arial, sans-serif;
}

h1 {
  color: #42b883;
  margin-bottom: 2rem;
}

.counter {
  text-align: center;
  padding: 2rem;
  border: 2px solid #42b883;
  border-radius: 12px;
  min-width: 200px;
}

.count {
  font-size: 4rem;
  font-weight: bold;
  color: #35495e;
  margin: 1rem 0;
}

.buttons {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

button {
  padding: 0.5rem 1.5rem;
  font-size: 1.2rem;
  border: none;
  border-radius: 8px;
  background-color: #42b883;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #35a776;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>
```

### 5.2 บันทึกและดูผลลัพธ์

บันทึกไฟล์ (Ctrl+S) และ Browser จะ reload อัตโนมัติ คุณจะเห็น Counter App ที่ทำงานได้จริง!

---

## ขั้นตอนที่ 6: ทำความเข้าใจ Composition API vs Options API

Vue.js มี 2 วิธีในการเขียน Component:

### 6.1 Options API (แบบเก่า Vue 2)

```vue
<script>
export default {
  name: 'CounterComponent',
  data() {
    return {
      count: 0,
      message: 'สวัสดี!'
    }
  },
  computed: {
    doubleCount() {
      return this.count * 2
    }
  },
  methods: {
    increment() {
      this.count++
    },
    decrement() {
      this.count--
    }
  },
  mounted() {
    console.log('Component mounted!')
  }
}
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <p>Count: {{ count }}</p>
    <p>Double: {{ doubleCount }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
  </div>
</template>
```

### 6.2 Composition API (แบบใหม่ Vue 3) - แนะนำ

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'

// Data
const count = ref(0)
const message = ref('สวัสดี!')

// Computed
const doubleCount = computed(() => count.value * 2)

// Methods
function increment() {
  count.value++
}

function decrement() {
  count.value--
}

// Lifecycle
onMounted(() => {
  console.log('Component mounted!')
})
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <p>Count: {{ count }}</p>
    <p>Double: {{ doubleCount }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
  </div>
</template>
```

**ทำไมถึงแนะนำ Composition API?**
- โค้ดอ่านง่ายกว่าในโปรเจกต์ขนาดใหญ่
- สามารถ reuse logic ได้ง่ายกว่า (Composables)
- TypeScript support ดีกว่า
- ประสิทธิภาพดีกว่าเล็กน้อย

---

## ขั้นตอนที่ 7: ตั้งค่า Git

### 7.1 Initialize Git Repository

```bash
# Initialize git
git init

# เพิ่มไฟล์ทั้งหมด
git add .

# Commit ครั้งแรก
git commit -m "Initial Vue.js project setup"
```

### 7.2 .gitignore

สร้างไฟล์ `.gitignore` (create-vue จะสร้างให้อัตโนมัติ):

```gitignore
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

# Node modules
node_modules
.DS_Store
dist
dist-ssr
coverage
*.local

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
```

---

## ขั้นตอนที่ 8: Scripts ที่สำคัญใน package.json

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test:unit": "vitest",
    "test:e2e": "playwright test",
    "lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs,.ts,.tsx,.cts,.mts --fix --ignore-path .gitignore",
    "format": "prettier --write src/"
  }
}
```

**คำอธิบาย:**
- `npm run dev` - รัน Development Server (Hot Reload)
- `npm run build` - Build สำหรับ Production
- `npm run preview` - Preview Production Build
- `npm run test:unit` - รัน Unit Tests
- `npm run lint` - ตรวจสอบและแก้ไข Code Style
- `npm run format` - Format โค้ดด้วย Prettier

---

## ขั้นตอนที่ 9: ตั้งค่า Vite Configuration

แก้ไข `vite.config.ts`:

```typescript
import { fileURLToPath, URL } from 'node:url'

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'

export default defineConfig({
  plugins: [
    vue(),
    vueJsx(),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
  server: {
    port: 3000,          // เปลี่ยน Port
    open: true,          // เปิด Browser อัตโนมัติ
    cors: true,
    proxy: {
      // Proxy สำหรับ API calls
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia']
        }
      }
    }
  }
})
```

---

## ขั้นตอนที่ 10: ตั้งค่า ESLint และ Prettier

### 10.1 .eslintrc.cjs

```javascript
/* eslint-env node */
require('@rushstack/eslint-patch/modern-module-resolution')

module.exports = {
  root: true,
  'extends': [
    'plugin:vue/vue3-essential',
    'eslint:recommended',
    '@vue/eslint-config-typescript',
    '@vue/eslint-config-prettier/skip-formatting'
  ],
  parserOptions: {
    ecmaVersion: 'latest'
  },
  rules: {
    // Vue specific rules
    'vue/multi-word-component-names': 'off',
    'vue/component-name-in-template-casing': ['error', 'PascalCase'],
    'vue/prop-name-casing': ['error', 'camelCase'],
    
    // TypeScript rules
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/explicit-function-return-type': 'off',
    
    // General rules
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'warn' : 'off'
  }
}
```

### 10.2 .prettierrc.json

```json
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "trailingComma": "none",
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "vueIndentScriptAndStyle": true
}
```

---

## สรุป Part 001

ในบทเรียนนี้คุณได้เรียนรู้:

✅ ติดตั้ง Node.js และ Package Manager (npm/pnpm/yarn)  
✅ ตั้งค่า VS Code พร้อม Extensions สำหรับ Vue.js  
✅ สร้าง Vue.js Project ด้วย Vite  
✅ ทำความเข้าใจโครงสร้างโปรเจกต์  
✅ เขียน Counter App อย่างง่าย  
✅ เปรียบเทียบ Options API vs Composition API  
✅ ตั้งค่า Git, ESLint, และ Prettier  

---

## แบบฝึกหัด

1. **ระดับง่าย:** สร้าง Counter ที่มีปุ่ม +5 และ -5
2. **ระดับกลาง:** เพิ่ม Input สำหรับใส่ข้อความที่ต้องการแสดง
3. **ระดับยาก:** สร้าง To-Do List อย่างง่ายโดยใช้ `ref` และ `computed`

---

## แหล่งข้อมูลเพิ่มเติม

- [Vue.js Official Docs](https://vuejs.org)
- [Vite Official Docs](https://vitejs.dev)
- [Vue DevTools](https://devtools.vuejs.org)

---

**ต่อไป → [Part 002: Vue.js Template Syntax](./part-002-template-syntax.md)**
