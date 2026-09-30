# Part 46: Nuxt Image Optimization

## ทำไมต้องทำ Image Optimization?

รูปภาพที่ไม่ได้รับการ optimize มักเป็นสาเหตุหลักที่ทำให้เว็บไซต์โหลดช้า `@nuxt/image` ช่วยแก้ปัญหานี้โดยอัตโนมัติ

## 1. ติดตั้ง @nuxt/image

```bash
npx nuxi module add image
# หรือ
npm install @nuxt/image
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  
  image: {
    // ขนาดที่ใช้บ่อย (สำหรับสร้าง srcset)
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280,
      xxl: 1536
    },
    
    // Format ที่รองรับ
    formats: ['webp', 'avif', 'jpeg'],
    
    // Quality เริ่มต้น
    quality: 80,
    
    // Providers
    providers: {
      cloudinary: {
        baseURL: 'https://res.cloudinary.com/my-cloud/image/upload/'
      }
    },
    
    // Domain ที่อนุญาต
    domains: ['images.example.com', 'cdn.example.com']
  }
})
```

## 2. NuxtImg Component

```vue
<!-- components/OptimizedImage.vue -->
<template>
  <div>
    <!-- รูปภาพพื้นฐาน -->
    <NuxtImg
      src="/images/hero.jpg"
      alt="Hero Image"
      width="1200"
      height="600"
    />
    
    <!-- พร้อม Lazy Loading -->
    <NuxtImg
      src="/images/product.jpg"
      alt="Product"
      width="400"
      height="300"
      loading="lazy"
      placeholder
    />
    
    <!-- Responsive Image -->
    <NuxtImg
      src="/images/banner.jpg"
      alt="Banner"
      sizes="sm:100vw md:50vw lg:800px"
      class="banner-image"
    />
    
    <!-- External Image (Cloudinary) -->
    <NuxtImg
      provider="cloudinary"
      src="samples/landscapes/nature-mountains.jpg"
      width="800"
      height="600"
      :modifiers="{ effect: 'auto_contrast', quality: 'auto' }"
    />
    
    <!-- ปรับแต่ง Format -->
    <NuxtImg
      src="/images/photo.jpg"
      format="webp"
      quality="85"
      width="600"
      height="400"
    />
  </div>
</template>
```

## 3. NuxtPicture Component

NuxtPicture รองรับหลาย format โดยอัตโนมัติ browser จะเลือก format ที่ดีที่สุด

```vue
<template>
  <div>
    <!-- Automatic format selection -->
    <NuxtPicture
      src="/images/photo.jpg"
      :imgAttrs="{ alt: 'Beautiful Photo', class: 'photo' }"
      width="800"
      height="600"
    />
    <!-- Output:
    <picture>
      <source type="image/avif" srcset="/images/photo.avif ...">
      <source type="image/webp" srcset="/images/photo.webp ...">
      <img src="/images/photo.jpg" alt="Beautiful Photo">
    </picture>
    -->
    
    <!-- Art Direction - รูปต่างกันตาม breakpoint -->
    <NuxtPicture
      src="/images/banner-desktop.jpg"
      :sources="[
        {
          media: '(max-width: 640px)',
          src: '/images/banner-mobile.jpg',
          width: 640,
          height: 400
        }
      ]"
      width="1280"
      height="400"
    />
  </div>
</template>
```

## 4. Image Providers

### Cloudinary Provider

```typescript
// nuxt.config.ts
image: {
  cloudinary: {
    baseURL: `https://res.cloudinary.com/${process.env.CLOUDINARY_CLOUD_NAME}/image/upload/`
  }
}
```

```vue
<template>
  <!-- Cloudinary Transformations -->
  <NuxtImg
    provider="cloudinary"
    src="products/shoe-001.jpg"
    width="400"
    height="400"
    :modifiers="{
      crop: 'fill',
      gravity: 'auto',
      effect: 'improve',
      quality: 'auto:good',
      fetchFormat: 'auto'
    }"
  />
</template>
```

### Imgix Provider

```typescript
// nuxt.config.ts
image: {
  imgix: {
    baseURL: 'https://your-domain.imgix.net'
  }
}
```

## 5. Responsive Images Best Practices

```vue
<!-- components/ResponsiveHero.vue -->
<template>
  <section class="hero">
    <NuxtPicture
      src="/images/hero-background.jpg"
      :imgAttrs="{
        alt: 'Hero Background',
        class: 'hero-bg',
        fetchpriority: 'high'
      }"
      sizes="100vw"
      :preload="true"
      width="1920"
      height="1080"
    />
    
    <div class="hero-content">
      <h1>{{ title }}</h1>
      <p>{{ subtitle }}</p>
    </div>
  </section>
</template>

<script setup>
defineProps<{
  title: string
  subtitle: string
}>()
</script>

<style scoped>
.hero {
  position: relative;
  height: 100vh;
  overflow: hidden;
}

.hero-bg {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hero-content {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: white;
  text-align: center;
}
</style>
```

## 6. Lazy Loading

```vue
<!-- components/LazyImageGrid.vue -->
<template>
  <div class="image-grid">
    <div
      v-for="image in images"
      :key="image.id"
      class="grid-item"
    >
      <NuxtImg
        :src="image.src"
        :alt="image.alt"
        loading="lazy"
        width="400"
        height="300"
        :placeholder="[400, 300, 70, 5]"
        class="grid-image"
      />
      <p>{{ image.caption }}</p>
    </div>
  </div>
</template>

<script setup>
defineProps<{
  images: Array<{
    id: string
    src: string
    alt: string
    caption: string
  }>
}>()
</script>

<style scoped>
.image-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 16px;
}

.grid-image {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 8px;
  transition: transform 0.3s;
}

.grid-image:hover {
  transform: scale(1.02);
}
</style>
```

## 7. ตัวอย่าง: Portfolio กับ Optimized Images

```vue
<!-- pages/portfolio.vue -->
<template>
  <div class="portfolio">
    <!-- Hero Section -->
    <section class="hero-section">
      <NuxtPicture
        src="/portfolio/hero.jpg"
        :imgAttrs="{ alt: 'Portfolio Hero', class: 'hero-image' }"
        sizes="100vw"
        width="1920"
        height="1080"
        :preload="true"
      />
      <div class="hero-overlay">
        <h1>My Portfolio</h1>
        <p>Creative Work & Projects</p>
      </div>
    </section>
    
    <!-- Filter Buttons -->
    <div class="filter-bar">
      <button
        v-for="cat in categories"
        :key="cat"
        @click="activeCategory = cat"
        :class="{ active: activeCategory === cat }"
        class="filter-btn"
      >
        {{ cat }}
      </button>
    </div>
    
    <!-- Projects Grid -->
    <div class="projects-grid">
      <div
        v-for="project in filteredProjects"
        :key="project.id"
        class="project-card"
        @click="openProject(project)"
      >
        <div class="project-image-wrapper">
          <NuxtImg
            :src="project.thumbnail"
            :alt="project.title"
            loading="lazy"
            width="600"
            height="400"
            sizes="sm:100vw md:50vw lg:400px"
            :placeholder="[100, 67, 10, 5]"
            class="project-image"
          />
          
          <div class="project-overlay">
            <span class="project-category">{{ project.category }}</span>
            <h3>{{ project.title }}</h3>
            <p>{{ project.description }}</p>
            <div class="project-tech">
              <span v-for="tech in project.technologies" :key="tech">{{ tech }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Modal -->
    <Teleport to="body">
      <div v-if="selectedProject" class="modal-overlay" @click.self="closeProject">
        <div class="modal-content">
          <button class="modal-close" @click="closeProject">×</button>
          
          <!-- Project Gallery -->
          <div class="project-gallery">
            <NuxtImg
              :src="selectedProject.images[activeImageIndex]"
              :alt="selectedProject.title"
              width="1200"
              height="800"
              quality="90"
              class="gallery-main-image"
            />
            
            <div class="gallery-thumbs">
              <NuxtImg
                v-for="(img, idx) in selectedProject.images"
                :key="img"
                :src="img"
                :alt="`${selectedProject.title} ${idx + 1}`"
                width="100"
                height="67"
                loading="lazy"
                @click="activeImageIndex = idx"
                :class="{ active: activeImageIndex === idx }"
                class="gallery-thumb"
              />
            </div>
          </div>
          
          <div class="project-details">
            <h2>{{ selectedProject.title }}</h2>
            <p>{{ selectedProject.fullDescription }}</p>
            
            <div class="tech-stack">
              <h3>เทคโนโลยีที่ใช้</h3>
              <div class="tech-list">
                <span v-for="tech in selectedProject.technologies" :key="tech">
                  {{ tech }}
                </span>
              </div>
            </div>
            
            <div class="project-links">
              <a :href="selectedProject.liveUrl" target="_blank" class="btn-live">
                ดูโปรเจกต์
              </a>
              <a :href="selectedProject.githubUrl" target="_blank" class="btn-github">
                GitHub
              </a>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script setup>
useSeoMeta({
  title: 'Portfolio - Optimized Images Demo',
  description: 'ตัวอย่าง Portfolio พร้อม Image Optimization'
})

const categories = ['ทั้งหมด', 'Web App', 'Mobile', 'Design']
const activeCategory = ref('ทั้งหมด')
const selectedProject = ref(null)
const activeImageIndex = ref(0)

const projects = [
  {
    id: 1,
    title: 'E-Commerce Platform',
    category: 'Web App',
    description: 'แพลตฟอร์มซื้อขายออนไลน์',
    fullDescription: 'ระบบ E-Commerce ที่สร้างด้วย Nuxt 3 และ Prisma รองรับการชำระเงิน',
    thumbnail: '/portfolio/ecommerce-thumb.jpg',
    images: [
      '/portfolio/ecommerce-1.jpg',
      '/portfolio/ecommerce-2.jpg',
      '/portfolio/ecommerce-3.jpg'
    ],
    technologies: ['Nuxt 3', 'Prisma', 'PostgreSQL', 'Stripe'],
    liveUrl: 'https://example.com',
    githubUrl: 'https://github.com/example'
  }
]

const filteredProjects = computed(() => {
  if (activeCategory.value === 'ทั้งหมด') return projects
  return projects.filter(p => p.category === activeCategory.value)
})

const openProject = (project) => {
  selectedProject.value = project
  activeImageIndex.value = 0
}

const closeProject = () => {
  selectedProject.value = null
}
</script>

<style scoped>
.portfolio { max-width: 1400px; margin: 0 auto; }

.hero-section {
  position: relative;
  height: 80vh;
  overflow: hidden;
}

.hero-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hero-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
  text-align: center;
}

.hero-overlay h1 {
  font-size: 3rem;
  font-weight: bold;
  margin-bottom: 12px;
}

.filter-bar {
  display: flex;
  gap: 8px;
  justify-content: center;
  padding: 32px 16px;
  flex-wrap: wrap;
}

.filter-btn {
  padding: 8px 20px;
  border-radius: 20px;
  border: 2px solid #e0e0e0;
  background: white;
  cursor: pointer;
  transition: all 0.2s;
}

.filter-btn.active {
  background: #333;
  color: white;
  border-color: #333;
}

.projects-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 24px;
  padding: 0 24px 48px;
}

.project-card {
  cursor: pointer;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  transition: transform 0.3s, box-shadow 0.3s;
}

.project-card:hover {
  transform: translateY(-8px);
  box-shadow: 0 8px 40px rgba(0,0,0,0.2);
}

.project-image-wrapper {
  position: relative;
  overflow: hidden;
}

.project-image {
  width: 100%;
  height: 250px;
  object-fit: cover;
  transition: transform 0.5s;
}

.project-card:hover .project-image {
  transform: scale(1.05);
}

.project-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(to bottom, transparent 30%, rgba(0,0,0,0.85) 100%);
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  color: white;
  opacity: 0;
  transition: opacity 0.3s;
}

.project-card:hover .project-overlay { opacity: 1; }

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.9);
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

.modal-content {
  background: white;
  border-radius: 16px;
  max-width: 1000px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
}

.modal-close {
  position: absolute;
  top: 16px;
  right: 16px;
  background: white;
  border: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  font-size: 20px;
  cursor: pointer;
}

.gallery-main-image {
  width: 100%;
  height: 500px;
  object-fit: cover;
}

.gallery-thumbs {
  display: flex;
  gap: 8px;
  padding: 12px;
}

.gallery-thumb {
  width: 80px;
  height: 60px;
  object-fit: cover;
  border-radius: 4px;
  cursor: pointer;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.gallery-thumb.active, .gallery-thumb:hover { opacity: 1; }

.project-details {
  padding: 24px;
}
</style>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **@nuxt/image** - ติดตั้งและตั้งค่า module
2. **NuxtImg** - Component สำหรับรูปภาพที่ optimize แล้ว
3. **NuxtPicture** - รองรับหลาย format (WebP, AVIF)
4. **Image Providers** - Cloudinary, Imgix integration
5. **Responsive Images** - ใช้ `sizes` attribute
6. **Lazy Loading** - โหลดรูปเมื่อเข้าสู่ viewport
7. **Portfolio Demo** - ตัวอย่างการใช้งานจริง
