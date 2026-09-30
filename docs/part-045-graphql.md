# Part 45: GraphQL กับ Vue/Nuxt.js

## GraphQL คืออะไร?

GraphQL เป็น query language สำหรับ API ที่พัฒนาโดย Facebook ช่วยให้ client สามารถขอข้อมูลที่ต้องการได้ตรงๆ ไม่ต้อง over-fetch หรือ under-fetch เหมือน REST API

## 1. GraphQL พื้นฐาน

### REST vs GraphQL

```
REST API (over-fetching):
GET /api/posts          → ได้ข้อมูลทั้งหมดของโพสต์
GET /api/posts/1/author → ต้องเรียก endpoint แยก

GraphQL (precise):
query {
  post(id: "1") {
    title
    author {
      name    ← ขอแค่ name ได้เลย
    }
  }
}
```

### GraphQL Schema Types

```graphql
# Schema Definition Language (SDL)

type Query {
  posts(page: Int, limit: Int, category: String): PostsResult!
  post(slug: String!): Post
  me: User
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  login(email: String!, password: String!): AuthResult!
}

type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
}

type Post {
  id: ID!
  title: String!
  slug: String!
  excerpt: String
  content: String!
  coverImage: String
  published: Boolean!
  publishedAt: String
  viewCount: Int!
  author: User!
  category: Category
  tags: [Tag!]!
  comments: [Comment!]!
  likesCount: Int!
  createdAt: String!
  updatedAt: String!
}

type User {
  id: ID!
  name: String!
  email: String!
  avatar: String
  bio: String
  role: Role!
  posts: [Post!]!
  postsCount: Int!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  color: String
  posts: [Post!]!
}

type Tag {
  id: ID!
  name: String!
  slug: String!
}

type Comment {
  id: ID!
  content: String!
  author: User!
  createdAt: String!
  replies: [Comment!]!
}

type PostsResult {
  posts: [Post!]!
  total: Int!
  page: Int!
  totalPages: Int!
}

type AuthResult {
  user: User!
  token: String!
}

input CreatePostInput {
  title: String!
  content: String!
  excerpt: String
  categoryId: ID
  tagIds: [ID!]
  published: Boolean
}

input UpdatePostInput {
  title: String
  content: String
  excerpt: String
  categoryId: ID
  tagIds: [ID!]
  published: Boolean
}

enum Role {
  ADMIN
  EDITOR
  AUTHOR
  VIEWER
}
```

## 2. Setup GraphQL Server กับ Nuxt

```bash
# ติดตั้ง packages
npm install graphql graphql-yoga @graphql-yoga/node
npm install @graphql-tools/schema @graphql-tools/merge
npm install -D @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-operations @graphql-codegen/typescript-vue-apollo
```

```typescript
// server/api/graphql.ts
import { createYoga, createSchema } from 'graphql-yoga'
import { typeDefs } from '../graphql/typeDefs'
import { resolvers } from '../graphql/resolvers'

const yoga = createYoga({
  schema: createSchema({ typeDefs, resolvers }),
  graphqlEndpoint: '/api/graphql',
  
  // Context สำหรับ authentication
  context: async ({ request }) => {
    const event = request as any
    
    // ดึง session
    let user = null
    try {
      const session = await getUserSession(event)
      user = session?.user || null
    } catch {}
    
    return {
      user,
      prisma
    }
  },
  
  plugins: [
    // logging
    {
      onExecute({ args }) {
        console.log('GraphQL Query:', args.document.loc?.source.body.substring(0, 100))
      }
    }
  ]
})

export default defineEventHandler(async (event) => {
  const { req, res } = event.node
  return yoga.handleNodeRequest(req, res)
})
```

```typescript
// server/graphql/typeDefs.ts
import { gql } from 'graphql-tag'

export const typeDefs = gql`
  type Query {
    posts(page: Int, limit: Int, category: String, tag: String, search: String): PostsResult!
    post(slug: String!): Post
    me: User
    user(id: ID!): User
    categories: [Category!]!
    tags: [Tag!]!
  }
  
  type Mutation {
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!
    
    createComment(postId: ID!, content: String!, parentId: ID): Comment!
    deleteComment(id: ID!): Boolean!
    
    toggleLike(postId: ID!): LikeResult!
    
    login(email: String!, password: String!): AuthResult!
    logout: Boolean!
  }
  
  type Subscription {
    commentAdded(postId: ID!): Comment!
    postPublished: Post!
  }
  
  type Post {
    id: ID!
    title: String!
    slug: String!
    excerpt: String
    content: String!
    coverImage: String
    published: Boolean!
    publishedAt: String
    viewCount: Int!
    author: User!
    category: Category
    tags: [Tag!]!
    comments(limit: Int): [Comment!]!
    commentsCount: Int!
    likesCount: Int!
    isLiked: Boolean!
    createdAt: String!
    updatedAt: String!
  }
  
  type User {
    id: ID!
    name: String!
    email: String!
    avatar: String
    bio: String
    role: String!
    posts(limit: Int): [Post!]!
    postsCount: Int!
  }
  
  type Category {
    id: ID!
    name: String!
    slug: String!
    color: String
    postsCount: Int!
  }
  
  type Tag {
    id: ID!
    name: String!
    slug: String!
    postsCount: Int!
  }
  
  type Comment {
    id: ID!
    content: String!
    author: User!
    replies: [Comment!]!
    createdAt: String!
  }
  
  type PostsResult {
    posts: [Post!]!
    total: Int!
    page: Int!
    limit: Int!
    totalPages: Int!
    hasNext: Boolean!
    hasPrev: Boolean!
  }
  
  type AuthResult {
    user: User!
    token: String!
  }
  
  type LikeResult {
    liked: Boolean!
    count: Int!
  }
  
  input CreatePostInput {
    title: String!
    content: String!
    excerpt: String
    categoryId: ID
    tagIds: [ID!]
    published: Boolean
    coverImage: String
  }
  
  input UpdatePostInput {
    title: String
    content: String
    excerpt: String
    categoryId: ID
    tagIds: [ID!]
    published: Boolean
    coverImage: String
  }
`
```

## 3. Resolvers

```typescript
// server/graphql/resolvers/index.ts
import { postResolvers } from './posts'
import { userResolvers } from './users'
import { commentResolvers } from './comments'
import mergeWith from 'lodash/mergeWith'

export const resolvers = mergeWith(
  {},
  postResolvers,
  userResolvers,
  commentResolvers,
  (obj: any, src: any) => {
    if (Array.isArray(obj)) return obj.concat(src)
  }
)
```

```typescript
// server/graphql/resolvers/posts.ts
import { GraphQLError } from 'graphql'

export const postResolvers = {
  Query: {
    posts: async (_: any, args: any, ctx: any) => {
      const { page = 1, limit = 10, category, tag, search } = args
      const skip = (page - 1) * limit
      
      const where: any = { published: true }
      if (category) where.category = { slug: category }
      if (tag) where.tags = { some: { tag: { slug: tag } } }
      if (search) {
        where.OR = [
          { title: { contains: search, mode: 'insensitive' } },
          { excerpt: { contains: search, mode: 'insensitive' } }
        ]
      }
      
      const [posts, total] = await ctx.prisma.$transaction([
        ctx.prisma.post.findMany({ where, skip, take: limit, orderBy: { publishedAt: 'desc' } }),
        ctx.prisma.post.count({ where })
      ])
      
      return {
        posts,
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
        hasNext: skip + limit < total,
        hasPrev: page > 1
      }
    },
    
    post: async (_: any, { slug }: any, ctx: any) => {
      const post = await ctx.prisma.post.findUnique({
        where: { slug, published: true }
      })
      if (!post) throw new GraphQLError('ไม่พบโพสต์', { extensions: { code: 'NOT_FOUND' } })
      
      // Increment view count
      ctx.prisma.post.update({
        where: { id: post.id },
        data: { viewCount: { increment: 1 } }
      }).catch(() => {})
      
      return post
    }
  },
  
  Mutation: {
    createPost: async (_: any, { input }: any, ctx: any) => {
      if (!ctx.user) throw new GraphQLError('ยังไม่ได้เข้าสู่ระบบ', { extensions: { code: 'UNAUTHENTICATED' } })
      
      const { tagIds, ...data } = input
      const slug = generateSlug(data.title)
      
      return ctx.prisma.post.create({
        data: {
          ...data,
          slug,
          authorId: ctx.user.id,
          publishedAt: data.published ? new Date() : null,
          tags: tagIds ? {
            create: tagIds.map((tagId: string) => ({ tagId }))
          } : undefined
        }
      })
    },
    
    deletePost: async (_: any, { id }: any, ctx: any) => {
      if (!ctx.user) throw new GraphQLError('ยังไม่ได้เข้าสู่ระบบ', { extensions: { code: 'UNAUTHENTICATED' } })
      
      const post = await ctx.prisma.post.findUnique({ where: { id } })
      if (!post) throw new GraphQLError('ไม่พบโพสต์')
      if (post.authorId !== ctx.user.id && ctx.user.role !== 'admin') {
        throw new GraphQLError('ไม่มีสิทธิ์', { extensions: { code: 'FORBIDDEN' } })
      }
      
      await ctx.prisma.post.delete({ where: { id } })
      return true
    }
  },
  
  // Field resolvers
  Post: {
    author: (parent: any, _: any, ctx: any) =>
      ctx.prisma.user.findUnique({ where: { id: parent.authorId } }),
    
    category: (parent: any, _: any, ctx: any) =>
      parent.categoryId ? ctx.prisma.category.findUnique({ where: { id: parent.categoryId } }) : null,
    
    tags: async (parent: any, _: any, ctx: any) => {
      const postTags = await ctx.prisma.postTag.findMany({
        where: { postId: parent.id },
        include: { tag: true }
      })
      return postTags.map((pt: any) => pt.tag)
    },
    
    comments: (parent: any, { limit = 20 }: any, ctx: any) =>
      ctx.prisma.comment.findMany({
        where: { postId: parent.id, parentId: null },
        take: limit,
        orderBy: { createdAt: 'desc' }
      }),
    
    commentsCount: (parent: any, _: any, ctx: any) =>
      ctx.prisma.comment.count({ where: { postId: parent.id } }),
    
    likesCount: (parent: any, _: any, ctx: any) =>
      ctx.prisma.like.count({ where: { postId: parent.id } }),
    
    isLiked: async (parent: any, _: any, ctx: any) => {
      if (!ctx.user) return false
      const like = await ctx.prisma.like.findUnique({
        where: { userId_postId: { userId: ctx.user.id, postId: parent.id } }
      })
      return !!like
    }
  }
}

const generateSlug = (title: string) => {
  return title
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
    .slice(0, 100)
}
```

## 4. Apollo Client กับ Vue

```bash
npm install @apollo/client @vue/apollo-composable graphql
# หรือใช้ nuxt-apollo
npm install @nuxtjs/apollo
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/apollo'],
  
  apollo: {
    autoImports: true,
    authType: 'Bearer',
    authHeader: 'Authorization',
    tokenStorage: 'cookie',
    clients: {
      default: {
        httpEndpoint: 'http://localhost:3000/api/graphql',
        wsEndpoint: 'ws://localhost:3000/api/graphql', // สำหรับ subscriptions
      }
    }
  }
})
```

## 5. Queries และ Mutations ใน Vue

```typescript
// graphql/queries/posts.ts
import { gql } from '@apollo/client/core'

export const GET_POSTS = gql`
  query GetPosts($page: Int, $limit: Int, $category: String, $search: String) {
    posts(page: $page, limit: $limit, category: $category, search: $search) {
      posts {
        id
        title
        slug
        excerpt
        coverImage
        publishedAt
        viewCount
        author {
          id
          name
          avatar
        }
        category {
          id
          name
          slug
          color
        }
        tags {
          id
          name
          slug
        }
        commentsCount
        likesCount
        isLiked
      }
      total
      page
      totalPages
      hasNext
      hasPrev
    }
  }
`

export const GET_POST = gql`
  query GetPost($slug: String!) {
    post(slug: $slug) {
      id
      title
      slug
      content
      excerpt
      coverImage
      publishedAt
      viewCount
      author {
        id
        name
        avatar
        bio
        postsCount
      }
      category {
        id
        name
        slug
        color
      }
      tags {
        id
        name
        slug
      }
      comments {
        id
        content
        createdAt
        author {
          id
          name
          avatar
        }
        replies {
          id
          content
          createdAt
          author {
            id
            name
            avatar
          }
        }
      }
      commentsCount
      likesCount
      isLiked
    }
  }
`

export const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
      slug
      published
    }
  }
`

export const TOGGLE_LIKE = gql`
  mutation ToggleLike($postId: ID!) {
    toggleLike(postId: $postId) {
      liked
      count
    }
  }
`
```

## 6. ใช้งาน Queries ใน Components

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-page">
    <h1>บทความทั้งหมด</h1>
    
    <!-- Loading -->
    <div v-if="loading" class="loading-grid">
      <PostSkeleton v-for="i in 6" :key="i" />
    </div>
    
    <!-- Error -->
    <div v-else-if="error" class="error-state">
      <p>เกิดข้อผิดพลาด: {{ error.message }}</p>
      <button @click="refetch">ลองใหม่</button>
    </div>
    
    <!-- Posts Grid -->
    <div v-else class="posts-grid">
      <PostCard
        v-for="post in posts?.posts"
        :key="post.id"
        :post="post"
        @like="handleLike"
      />
    </div>
    
    <!-- Pagination -->
    <Pagination
      v-if="posts"
      :page="currentPage"
      :total-pages="posts.totalPages"
      :has-next="posts.hasNext"
      :has-prev="posts.hasPrev"
      @change="changePage"
    />
  </div>
</template>

<script setup>
import { GET_POSTS, TOGGLE_LIKE } from '~/graphql/queries/posts'

const route = useRoute()
const currentPage = ref(Number(route.query.page) || 1)
const categoryFilter = ref(route.query.category || '')
const searchQuery = ref(route.query.search || '')

// Query
const { result: postsResult, loading, error, refetch } = useQuery(
  GET_POSTS,
  computed(() => ({
    page: currentPage.value,
    limit: 12,
    category: categoryFilter.value || undefined,
    search: searchQuery.value || undefined
  })),
  {
    fetchPolicy: 'cache-and-network'
  }
)

const posts = computed(() => postsResult.value?.posts)

// Mutation
const { mutate: toggleLike } = useMutation(TOGGLE_LIKE)

const handleLike = async (postId) => {
  await toggleLike(
    { postId },
    {
      // Optimistic update
      optimisticResponse: {
        toggleLike: {
          __typename: 'LikeResult',
          liked: true,
          count: 0
        }
      }
    }
  )
}

const changePage = (newPage) => {
  currentPage.value = newPage
  window.scrollTo(0, 0)
}
</script>
```

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article v-if="post" class="post-article">
    <header class="post-header">
      <img
        v-if="post.coverImage"
        :src="post.coverImage"
        :alt="post.title"
        class="cover-image"
      />
      <div class="post-meta">
        <span class="category-badge" v-if="post.category">
          {{ post.category.name }}
        </span>
        <div class="tags">
          <span v-for="tag in post.tags" :key="tag.id" class="tag">
            #{{ tag.name }}
          </span>
        </div>
      </div>
      <h1>{{ post.title }}</h1>
      <div class="author-info">
        <img :src="post.author.avatar" :alt="post.author.name" />
        <div>
          <strong>{{ post.author.name }}</strong>
          <time>{{ formatDate(post.publishedAt) }}</time>
        </div>
      </div>
    </header>
    
    <div class="post-content" v-html="renderedContent"></div>
    
    <!-- Like Button -->
    <div class="post-actions">
      <button @click="handleLike" class="like-btn" :class="{ liked: post.isLiked }">
        {{ post.isLiked ? '❤️' : '🤍' }} {{ post.likesCount }}
      </button>
    </div>
    
    <!-- Comments -->
    <section class="comments-section">
      <h2>{{ post.commentsCount }} ความคิดเห็น</h2>
      
      <CommentForm @submit="submitComment" />
      
      <div class="comments-list">
        <CommentItem
          v-for="comment in post.comments"
          :key="comment.id"
          :comment="comment"
          @reply="handleReply"
        />
      </div>
    </section>
  </article>
</template>

<script setup>
import { GET_POST, TOGGLE_LIKE } from '~/graphql/queries/posts'
import { ADD_COMMENT } from '~/graphql/mutations/comments'
import { marked } from 'marked'

const route = useRoute()

const { result, loading, error } = useQuery(
  GET_POST,
  { slug: route.params.slug },
  { fetchPolicy: 'cache-and-network' }
)

const post = computed(() => result.value?.post)
const renderedContent = computed(() => post.value ? marked(post.value.content) : '')

const { mutate: toggleLike } = useMutation(TOGGLE_LIKE)
const { mutate: addComment } = useMutation(ADD_COMMENT)

const handleLike = () => toggleLike({ postId: post.value.id })

const submitComment = async (content, parentId = null) => {
  await addComment({
    postId: post.value.id,
    content,
    parentId
  }, {
    refetchQueries: [{ query: GET_POST, variables: { slug: route.params.slug } }]
  })
}

const formatDate = (date) => {
  return new Date(date).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

// SEO
useHead({
  title: computed(() => post.value?.title || 'Loading...'),
  meta: [
    { name: 'description', content: computed(() => post.value?.excerpt || '') }
  ]
})
</script>
```

## 7. Subscriptions

```typescript
// graphql/subscriptions/comments.ts
import { gql } from '@apollo/client/core'

export const COMMENT_ADDED = gql`
  subscription OnCommentAdded($postId: ID!) {
    commentAdded(postId: $postId) {
      id
      content
      createdAt
      author {
        id
        name
        avatar
      }
    }
  }
`
```

```typescript
// server/graphql/resolvers/subscriptions.ts
import { PubSub } from 'graphql-subscriptions'

export const pubsub = new PubSub()

export const subscriptionResolvers = {
  Subscription: {
    commentAdded: {
      subscribe: (_: any, { postId }: any) => {
        return pubsub.asyncIterator([`COMMENT_ADDED_${postId}`])
      }
    },
    postPublished: {
      subscribe: () => pubsub.asyncIterator(['POST_PUBLISHED'])
    }
  }
}

// เรียกใน mutation
export const publishCommentEvent = (postId: string, comment: any) => {
  pubsub.publish(`COMMENT_ADDED_${postId}`, { commentAdded: comment })
}
```

```vue
<!-- ใช้ Subscription -->
<script setup>
import { COMMENT_ADDED } from '~/graphql/subscriptions/comments'

const { result: commentResult } = useSubscription(
  COMMENT_ADDED,
  { postId: postId.value }
)

// เมื่อมี comment ใหม่
watch(commentResult, (newData) => {
  if (newData?.commentAdded) {
    comments.value.unshift(newData.commentAdded)
  }
})
</script>
```

## 8. Code Generation

```typescript
// codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli'

const config: CodegenConfig = {
  schema: 'http://localhost:3000/api/graphql',
  documents: ['graphql/**/*.ts', 'pages/**/*.vue', 'components/**/*.vue'],
  generates: {
    './types/graphql.ts': {
      plugins: ['typescript', 'typescript-operations']
    },
    './composables/graphql.ts': {
      plugins: ['typescript-vue-apollo']
    }
  }
}

export default config
```

```bash
# รัน code generation
npx graphql-codegen
# หรือ watch mode
npx graphql-codegen --watch
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GraphQL พื้นฐาน** - Schema, Queries, Mutations, Subscriptions
2. **Apollo Client** - ตั้งค่าและใช้งานกับ Vue/Nuxt
3. **nuxt-apollo** - Module สำหรับ Nuxt
4. **Resolvers** - การเขียน Server-side logic
5. **Subscriptions** - Real-time data ด้วย WebSocket
6. **Code Generation** - สร้าง TypeScript types อัตโนมัติ
7. **Blog API** - ตัวอย่างสมบูรณ์
