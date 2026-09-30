# Part 61: Payment Gateway Integration ใน Nuxt.js

## Payment Gateway Options

ตัวเลือก Payment Gateway สำหรับเว็บไทยและสากล:

| Gateway | ประเทศ | ค่าธรรมเนียม | Features |
|---------|--------|-------------|---------|
| Stripe | สากล | 3.4% + ฿12 | Card, Promptpay, Apple/Google Pay |
| Omise | ไทย | 3.65% | PromptPay, Internet Banking |
| 2C2P | ไทย/เอเชีย | ตามข้อตกลง | ครบครัน |
| SCB Easy API | ไทย | ตามข้อตกลง | QR PromptPay |
| PayPal | สากล | 4.4% + fixed | International |

## Stripe Integration

### Frontend Setup

```bash
npm install @stripe/stripe-js
```

```typescript
// plugins/stripe.client.ts
import { loadStripe } from '@stripe/stripe-js'

export default defineNuxtPlugin(async () => {
  const config = useRuntimeConfig()
  const stripe = await loadStripe(config.public.stripePublicKey)
  
  return {
    provide: {
      stripe
    }
  }
})
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
    public: {
      stripePublicKey: process.env.STRIPE_PUBLIC_KEY
    }
  }
})
```

### Create Payment Intent

```typescript
// server/api/payments/create-intent.post.ts
import Stripe from 'stripe'

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const stripe = new Stripe(config.stripeSecretKey, {
    apiVersion: '2023-10-16'
  })
  
  const body = await readBody(event)
  const { amount, currency = 'thb', orderId, customer } = body
  
  // Validate amount
  if (!amount || amount < 20) {
    throw createError({ statusCode: 400, message: 'Invalid amount' })
  }
  
  try {
    // Find or create Stripe customer
    let stripeCustomerId: string | undefined
    if (customer?.email) {
      const customers = await stripe.customers.list({ email: customer.email, limit: 1 })
      if (customers.data.length > 0) {
        stripeCustomerId = customers.data[0].id
      } else {
        const newCustomer = await stripe.customers.create({
          email: customer.email,
          name: customer.name,
          metadata: { userId: customer.id }
        })
        stripeCustomerId = newCustomer.id
      }
    }
    
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Convert to satang (smallest unit)
      currency,
      customer: stripeCustomerId,
      metadata: {
        orderId,
        userId: customer?.id || 'guest'
      },
      payment_method_types: ['card', 'promptpay'],
      receipt_email: customer?.email
    })
    
    return {
      clientSecret: paymentIntent.client_secret,
      paymentIntentId: paymentIntent.id
    }
  } catch (error: any) {
    throw createError({
      statusCode: 400,
      message: error.message
    })
  }
})
```

### Payment Form Component

```vue
<!-- components/payment/StripePaymentForm.vue -->
<template>
  <div class="payment-form">
    <div class="payment-methods">
      <button
        v-for="method in paymentMethods"
        :key="method.id"
        :class="['method-btn', { active: selectedMethod === method.id }]"
        @click="selectedMethod = method.id"
        type="button"
      >
        <img :src="method.icon" :alt="method.name" />
        <span>{{ method.name }}</span>
      </button>
    </div>
    
    <!-- Stripe Payment Element -->
    <div id="payment-element" ref="paymentElementRef" class="stripe-element"></div>
    
    <!-- PromptPay QR (shown when PromptPay selected) -->
    <div v-if="promptPayQR" class="promptpay-qr">
      <img :src="promptPayQR" alt="QR PromptPay" />
      <p>สแกน QR เพื่อชำระเงิน</p>
      <div class="qr-countdown">
        หมดอายุใน: {{ qrCountdown }} วินาที
      </div>
    </div>
    
    <!-- Error message -->
    <div v-if="errorMessage" class="payment-error" role="alert">
      <span>⚠️ {{ errorMessage }}</span>
    </div>
    
    <!-- Submit Button -->
    <button
      type="button"
      :disabled="!isStripeReady || isProcessing"
      :aria-busy="isProcessing"
      @click="handlePayment"
      class="pay-button"
    >
      <template v-if="isProcessing">
        <span class="spinner" aria-hidden="true"></span>
        กำลังดำเนินการ...
      </template>
      <template v-else>
        ชำระเงิน {{ formatAmount(amount) }}
      </template>
    </button>
    
    <!-- Security badges -->
    <div class="security-badges">
      <span>🔒 SSL Secured</span>
      <span>Powered by Stripe</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'
import type { Stripe, StripeElements, StripePaymentElementOptions } from '@stripe/stripe-js'

const props = defineProps<{
  amount: number
  orderId: string
  customerEmail?: string
}>()

const emit = defineEmits<{
  success: [paymentIntentId: string]
  error: [message: string]
  cancel: []
}>()

const { $stripe } = useNuxtApp()
const paymentElementRef = ref<HTMLElement | null>(null)
const isStripeReady = ref(false)
const isProcessing = ref(false)
const errorMessage = ref('')
const selectedMethod = ref('card')
const promptPayQR = ref<string | null>(null)
const qrCountdown = ref(0)

let stripe: Stripe | null = null
let elements: StripeElements | null = null
let countdownTimer: NodeJS.Timeout | null = null

const paymentMethods = [
  { id: 'card', name: 'บัตรเครดิต/เดบิต', icon: '/icons/credit-card.svg' },
  { id: 'promptpay', name: 'PromptPay', icon: '/icons/promptpay.svg' }
]

const formatAmount = (amount: number) => {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency: 'THB'
  }).format(amount)
}

onMounted(async () => {
  await initializeStripe()
})

onUnmounted(() => {
  if (countdownTimer) clearInterval(countdownTimer)
})

const initializeStripe = async () => {
  try {
    // Create payment intent
    const { clientSecret } = await $fetch('/api/payments/create-intent', {
      method: 'POST',
      body: {
        amount: props.amount,
        orderId: props.orderId,
        customer: { email: props.customerEmail }
      }
    })
    
    stripe = $stripe
    if (!stripe) throw new Error('Stripe not loaded')
    
    elements = stripe.elements({
      clientSecret,
      appearance: {
        theme: 'flat',
        variables: {
          colorPrimary: '#7c3aed',
          colorBackground: '#ffffff',
          colorText: '#1f2937',
          fontFamily: 'Sarabun, sans-serif'
        }
      },
      locale: 'th'
    })
    
    const paymentElementOptions: StripePaymentElementOptions = {
      layout: {
        type: 'tabs',
        defaultCollapsed: false
      }
    }
    
    const paymentElement = elements.create('payment', paymentElementOptions)
    paymentElement.mount(paymentElementRef.value!)
    
    paymentElement.on('ready', () => {
      isStripeReady.value = true
    })
    
    paymentElement.on('change', (event) => {
      errorMessage.value = event.complete ? '' : ''
    })
  } catch (error: any) {
    errorMessage.value = error.message || 'ไม่สามารถโหลด Payment Form ได้'
  }
}

const handlePayment = async () => {
  if (!stripe || !elements) return
  
  isProcessing.value = true
  errorMessage.value = ''
  
  try {
    const { error, paymentIntent } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/payment/success?orderId=${props.orderId}`,
        receipt_email: props.customerEmail
      },
      redirect: 'if_required'
    })
    
    if (error) {
      if (error.type === 'card_error' || error.type === 'validation_error') {
        errorMessage.value = error.message || 'เกิดข้อผิดพลาด'
      } else {
        errorMessage.value = 'เกิดข้อผิดพลาดที่ไม่คาดคิด กรุณาลองใหม่'
      }
      emit('error', errorMessage.value)
      
      // Handle PromptPay QR
    } else if (paymentIntent) {
      if (paymentIntent.status === 'requires_action' && 
          paymentIntent.next_action?.type === 'promptpay_display_qr_code') {
        const qrCodeUrl = paymentIntent.next_action.promptpay_display_qr_code?.image_url_svg
        if (qrCodeUrl) {
          promptPayQR.value = qrCodeUrl
          startQRCountdown()
          pollPaymentStatus(paymentIntent.id)
        }
      } else if (paymentIntent.status === 'succeeded') {
        emit('success', paymentIntent.id)
      }
    }
  } finally {
    isProcessing.value = false
  }
}

const startQRCountdown = () => {
  qrCountdown.value = 3600 // 1 hour for PromptPay
  countdownTimer = setInterval(() => {
    qrCountdown.value--
    if (qrCountdown.value <= 0) {
      clearInterval(countdownTimer!)
      promptPayQR.value = null
      errorMessage.value = 'QR Code หมดอายุแล้ว กรุณาทำรายการใหม่'
    }
  }, 1000)
}

const pollPaymentStatus = async (paymentIntentId: string) => {
  const maxAttempts = 60
  let attempts = 0
  
  const checkStatus = async () => {
    if (attempts >= maxAttempts) return
    attempts++
    
    const { status } = await $fetch(`/api/payments/status/${paymentIntentId}`)
    
    if (status === 'succeeded') {
      promptPayQR.value = null
      if (countdownTimer) clearInterval(countdownTimer)
      emit('success', paymentIntentId)
    } else if (status === 'requires_payment_method') {
      errorMessage.value = 'การชำระเงินไม่สำเร็จ'
    } else {
      setTimeout(checkStatus, 3000) // Check every 3 seconds
    }
  }
  
  setTimeout(checkStatus, 3000)
}
</script>
```

## SCB Easy Pay (ไทย)

```typescript
// server/api/payments/scb/create-qr.post.ts
import crypto from 'crypto'

interface SCBQRPayload {
  amount: number
  ref1: string
  ref2?: string
}

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const body = await readBody<SCBQRPayload>(event)
  
  const timestamp = new Date().toISOString()
  const uuid = crypto.randomUUID()
  
  // Get SCB OAuth token
  const tokenResponse = await $fetch<{ access_token: string }>(
    `${config.scbApiUrl}/v1/oauth/token`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'resourceOwnerId': config.scbApiKey,
        'requestUId': uuid,
        'accept-language': 'EN'
      },
      body: {
        applicationKey: config.scbApiKey,
        applicationSecret: config.scbApiSecret
      }
    }
  )
  
  // Create QR Code
  const qrResponse = await $fetch(
    `${config.scbApiUrl}/v1/payment/qrcode/create`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${tokenResponse.access_token}`,
        'resourceOwnerId': config.scbApiKey,
        'requestUId': uuid,
        'accept-language': 'EN'
      },
      body: {
        qrType: 'PP',
        ppType: 'BILLERID',
        ppId: config.scbBillerId,
        amount: body.amount.toFixed(2),
        ref1: body.ref1,
        ref2: body.ref2 || 'WEB',
        ref3: 'SCB'
      }
    }
  )
  
  return {
    qrImage: qrResponse.data?.qrImage,
    qrRawData: qrResponse.data?.qrRawData,
    uuid
  }
})
```

## PromptPay QR

```typescript
// utils/promptpay.ts
export function generatePromptPayPayload(
  phoneOrTaxId: string,
  amount?: number
): string {
  const isPhone = phoneOrTaxId.length === 10 || phoneOrTaxId.length === 13
  
  // Format phone: 0812345678 -> 0066812345678
  const formatted = isPhone
    ? `0066${phoneOrTaxId.slice(1)}`
    : phoneOrTaxId.replace(/-/g, '')
  
  const idType = formatted.length === 13 ? '02' : '01'
  
  const buildTLV = (tag: string, value: string): string => {
    const length = value.length.toString().padStart(2, '0')
    return `${tag}${length}${value}`
  }
  
  const merchantAccountInfo = buildTLV(
    '00', 'A000000677010111'
  ) + buildTLV(
    idType, formatted
  )
  
  let payload = [
    '000201', // Payload Format Indicator
    '010212', // Point of Initiation Method
    buildTLV('29', merchantAccountInfo), // Merchant Account Info
    '5303764', // Transaction Currency (THB)
    '5802TH', // Country Code
  ]
  
  if (amount) {
    payload.push(buildTLV('54', amount.toFixed(2)))
  }
  
  const payloadStr = payload.join('') + '6304'
  const crc = calculateCRC16(payloadStr)
  
  return payloadStr + crc
}

function calculateCRC16(data: string): string {
  let crc = 0xFFFF
  for (let i = 0; i < data.length; i++) {
    crc ^= data.charCodeAt(i) << 8
    for (let j = 0; j < 8; j++) {
      if (crc & 0x8000) {
        crc = (crc << 1) ^ 0x1021
      } else {
        crc <<= 1
      }
    }
  }
  return (crc & 0xFFFF).toString(16).toUpperCase().padStart(4, '0')
}
```

```vue
<!-- components/payment/PromptPayQR.vue -->
<template>
  <div class="promptpay-qr">
    <div class="qr-header">
      <img src="/images/promptpay-logo.png" alt="PromptPay" class="promptpay-logo" />
      <h3>ชำระด้วย PromptPay</h3>
    </div>
    
    <div class="qr-code" v-if="qrDataUrl">
      <img :src="qrDataUrl" alt="QR Code PromptPay" />
      <p class="qr-amount">฿{{ amount.toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}</p>
    </div>
    
    <div v-else class="qr-loading">
      <div class="spinner"></div>
      <p>กำลังสร้าง QR Code...</p>
    </div>
    
    <div class="qr-steps">
      <p>วิธีชำระเงิน:</p>
      <ol>
        <li>เปิดแอปธนาคาร</li>
        <li>เลือก "สแกน QR"</li>
        <li>สแกน QR Code ด้านบน</li>
        <li>ยืนยันการชำระเงิน</li>
      </ol>
    </div>
    
    <div v-if="countdown > 0" class="qr-timer">
      <div class="timer-bar" :style="{ width: `${(countdown / 900) * 100}%` }"></div>
      <p>QR หมดอายุใน {{ formatCountdown }}</p>
    </div>
    
    <div v-else class="qr-expired">
      <p>QR Code หมดอายุแล้ว</p>
      <button @click="$emit('refresh')">สร้าง QR ใหม่</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'
import QRCode from 'qrcode'
import { generatePromptPayPayload } from '@/utils/promptpay'

const props = defineProps<{
  phoneNumber: string
  amount: number
  orderId: string
}>()

defineEmits(['success', 'expired', 'refresh'])

const qrDataUrl = ref<string | null>(null)
const countdown = ref(900) // 15 minutes
let timer: NodeJS.Timeout | null = null

const formatCountdown = computed(() => {
  const minutes = Math.floor(countdown.value / 60)
  const seconds = countdown.value % 60
  return `${minutes}:${seconds.toString().padStart(2, '0')}`
})

onMounted(async () => {
  await generateQR()
  startCountdown()
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
})

const generateQR = async () => {
  const payload = generatePromptPayPayload(props.phoneNumber, props.amount)
  qrDataUrl.value = await QRCode.toDataURL(payload, {
    width: 300,
    margin: 2,
    color: {
      dark: '#1a56db',
      light: '#ffffff'
    }
  })
}

const startCountdown = () => {
  timer = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      if (timer) clearInterval(timer)
    }
  }, 1000)
}
</script>
```

## Webhook Handling

```typescript
// server/api/webhooks/stripe.post.ts
import Stripe from 'stripe'
import { prisma } from '~/server/db'

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const stripe = new Stripe(config.stripeSecretKey, { apiVersion: '2023-10-16' })
  
  const signature = getHeader(event, 'stripe-signature')
  const rawBody = await readRawBody(event)
  
  if (!signature || !rawBody) {
    throw createError({ statusCode: 400, message: 'Missing signature or body' })
  }
  
  let stripeEvent: Stripe.Event
  
  try {
    stripeEvent = stripe.webhooks.constructEvent(
      rawBody,
      signature,
      config.stripeWebhookSecret
    )
  } catch (err) {
    throw createError({ statusCode: 400, message: 'Webhook signature verification failed' })
  }
  
  // Handle events
  switch (stripeEvent.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSucceeded(stripeEvent.data.object as Stripe.PaymentIntent)
      break
      
    case 'payment_intent.payment_failed':
      await handlePaymentFailed(stripeEvent.data.object as Stripe.PaymentIntent)
      break
      
    case 'charge.dispute.created':
      await handleDisputeCreated(stripeEvent.data.object as Stripe.Dispute)
      break
      
    case 'customer.subscription.updated':
    case 'customer.subscription.deleted':
      await handleSubscriptionChange(stripeEvent.data.object as Stripe.Subscription)
      break
  }
  
  return { received: true }
})

async function handlePaymentSucceeded(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId
  if (!orderId) return
  
  await prisma.order.update({
    where: { id: orderId },
    data: {
      paymentStatus: 'paid',
      status: 'confirmed',
      paymentIntentId: paymentIntent.id,
      paidAt: new Date()
    }
  })
  
  // Send confirmation email
  const order = await prisma.order.findUnique({
    where: { id: orderId },
    include: { user: true, items: true }
  })
  
  if (order?.user?.email) {
    await sendOrderConfirmationEmail(order)
  }
}

async function handlePaymentFailed(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId
  if (!orderId) return
  
  await prisma.order.update({
    where: { id: orderId },
    data: {
      paymentStatus: 'failed',
      status: 'cancelled'
    }
  })
}
```

## Transaction Records

```typescript
// server/api/payments/transactions.get.ts
export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const query = getQuery(event)
  const { page = 1, limit = 20, status } = query
  
  const where: any = { userId: user.id }
  if (status) where.status = status
  
  const [transactions, total] = await prisma.$transaction([
    prisma.paymentTransaction.findMany({
      where,
      include: {
        order: { select: { orderNumber: true } }
      },
      orderBy: { createdAt: 'desc' },
      skip: (Number(page) - 1) * Number(limit),
      take: Number(limit)
    }),
    prisma.paymentTransaction.count({ where })
  ])
  
  return {
    transactions,
    pagination: {
      total,
      page: Number(page),
      limit: Number(limit),
      totalPages: Math.ceil(total / Number(limit))
    }
  }
})
```

## Refund System

```typescript
// server/api/payments/refund.post.ts
import Stripe from 'stripe'

export default defineEventHandler(async (event) => {
  const user = await requireAuth(event)
  const config = useRuntimeConfig()
  const stripe = new Stripe(config.stripeSecretKey, { apiVersion: '2023-10-16' })
  
  const body = await readBody(event)
  const { orderId, amount, reason } = body
  
  // Verify order belongs to user
  const order = await prisma.order.findFirst({
    where: { id: orderId, userId: user.id },
    include: { refund: true }
  })
  
  if (!order) {
    throw createError({ statusCode: 404, message: 'Order not found' })
  }
  
  if (order.paymentStatus !== 'paid') {
    throw createError({ statusCode: 400, message: 'Order has not been paid' })
  }
  
  if (order.refund) {
    throw createError({ statusCode: 400, message: 'Order already refunded' })
  }
  
  // Process refund with Stripe
  const refundAmount = amount || order.total
  
  try {
    const refund = await stripe.refunds.create({
      payment_intent: order.paymentIntentId!,
      amount: Math.round(refundAmount * 100),
      reason: reason || 'requested_by_customer',
      metadata: { orderId }
    })
    
    // Update order
    await prisma.order.update({
      where: { id: orderId },
      data: {
        paymentStatus: 'refunded',
        status: 'refunded',
        refund: {
          create: {
            stripeRefundId: refund.id,
            amount: refundAmount,
            reason,
            status: refund.status
          }
        }
      }
    })
    
    // Restore inventory
    await restoreInventory(order)
    
    // Send refund email
    await sendRefundEmail(order, refundAmount)
    
    return { success: true, refundId: refund.id }
  } catch (error: any) {
    throw createError({ statusCode: 400, message: error.message })
  }
})
```

## สรุป

Payment Gateway Integration ที่ดีต้องมี:
1. Secure payment form (never handle raw card data)
2. Webhook verification ด้วย HMAC signature
3. Idempotent payment processing
4. Proper error handling
5. Transaction logging
6. Refund capability
7. PCI compliance
