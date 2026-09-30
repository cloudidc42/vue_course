# Part 62: Email System ใน Nuxt.js

## Email ใน Nuxt

Nuxt.js รองรับการส่ง email ผ่าน server-side API routes ซึ่งปลอดภัยกว่าการส่งจาก client

## Resend Integration

Resend คือ modern email service ที่ง่ายต่อการใช้งาน

```bash
npm install resend
```

```typescript
// server/services/email.ts
import { Resend } from 'resend'

const resend = new Resend(process.env.RESEND_API_KEY)

interface SendEmailOptions {
  to: string | string[]
  subject: string
  html: string
  text?: string
  from?: string
  replyTo?: string
  attachments?: Array<{
    filename: string
    content: Buffer | string
  }>
}

export async function sendEmail(options: SendEmailOptions) {
  const { to, subject, html, text, from, replyTo, attachments } = options
  
  try {
    const { data, error } = await resend.emails.send({
      from: from || `${process.env.EMAIL_FROM_NAME} <${process.env.EMAIL_FROM}>`,
      to: Array.isArray(to) ? to : [to],
      subject,
      html,
      text,
      reply_to: replyTo,
      attachments
    })
    
    if (error) {
      throw new Error(error.message)
    }
    
    return { success: true, id: data?.id }
  } catch (error: any) {
    console.error('Email send error:', error)
    throw error
  }
}
```

## Nodemailer Integration

```typescript
// server/services/nodemailer.ts
import nodemailer from 'nodemailer'

const config = useRuntimeConfig()

const createTransporter = () => {
  // SMTP configuration
  return nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: Number(process.env.SMTP_PORT) || 587,
    secure: process.env.SMTP_SECURE === 'true',
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS
    },
    tls: {
      rejectUnauthorized: process.env.NODE_ENV === 'production'
    }
  })
}

// For development: use Mailtrap or Ethereal
const createDevTransporter = async () => {
  const testAccount = await nodemailer.createTestAccount()
  return nodemailer.createTransport({
    host: 'smtp.ethereal.email',
    port: 587,
    secure: false,
    auth: {
      user: testAccount.user,
      pass: testAccount.pass
    }
  })
}

export async function sendEmailWithNodemailer(options: {
  to: string
  subject: string
  html: string
  text?: string
}) {
  const transporter = process.env.NODE_ENV === 'production'
    ? createTransporter()
    : await createDevTransporter()
  
  const info = await transporter.sendMail({
    from: `"${process.env.EMAIL_FROM_NAME}" <${process.env.EMAIL_FROM}>`,
    ...options
  })
  
  if (process.env.NODE_ENV !== 'production') {
    console.log('Preview URL:', nodemailer.getTestMessageUrl(info))
  }
  
  return info
}
```

## SendGrid Integration

```typescript
// server/services/sendgrid.ts
import sgMail from '@sendgrid/mail'

sgMail.setApiKey(process.env.SENDGRID_API_KEY!)

export async function sendWithSendGrid(options: {
  to: string
  templateId: string
  dynamicTemplateData: Record<string, any>
  from?: string
}) {
  const { to, templateId, dynamicTemplateData, from } = options
  
  try {
    await sgMail.send({
      to,
      from: from || process.env.EMAIL_FROM!,
      templateId,
      dynamicTemplateData
    })
    return { success: true }
  } catch (error: any) {
    console.error('SendGrid error:', error.response?.body || error.message)
    throw error
  }
}

// ส่ง bulk email
export async function sendBulkEmails(emails: Array<{
  to: string
  templateId: string
  dynamicTemplateData: Record<string, any>
}>) {
  const messages = emails.map(({ to, templateId, dynamicTemplateData }) => ({
    to,
    from: process.env.EMAIL_FROM!,
    templateId,
    dynamicTemplateData
  }))
  
  await sgMail.send(messages)
}
```

## Email Templates

```typescript
// server/templates/emails/base.ts
export function createBaseTemplate(content: string, options: {
  title: string
  previewText?: string
}): string {
  return `
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="x-apple-disable-message-reformatting">
  <title>${options.title}</title>
  ${options.previewText ? `<span style="display:none;font-size:1px;color:#ffffff;max-height:0;">${options.previewText}</span>` : ''}
  <style>
    body {
      margin: 0;
      padding: 0;
      background-color: #f9fafb;
      font-family: 'Sarabun', Arial, sans-serif;
      font-size: 16px;
      line-height: 1.6;
      color: #374151;
    }
    .container {
      max-width: 600px;
      margin: 40px auto;
      background: #ffffff;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    .header {
      background: linear-gradient(135deg, #7c3aed, #4f46e5);
      padding: 32px 40px;
      text-align: center;
    }
    .header img { max-width: 150px; }
    .body { padding: 40px; }
    .footer {
      background: #f3f4f6;
      padding: 24px 40px;
      text-align: center;
      font-size: 14px;
      color: #6b7280;
    }
    .button {
      display: inline-block;
      padding: 14px 32px;
      background: #7c3aed;
      color: #ffffff !important;
      text-decoration: none;
      border-radius: 8px;
      font-weight: 600;
      margin: 16px 0;
    }
    .divider {
      border: none;
      border-top: 1px solid #e5e7eb;
      margin: 24px 0;
    }
    @media only screen and (max-width: 600px) {
      .container { margin: 0; border-radius: 0; }
      .body { padding: 24px; }
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <img src="${process.env.APP_URL}/logo-white.png" alt="${process.env.APP_NAME}">
    </div>
    <div class="body">
      ${content}
    </div>
    <div class="footer">
      <p>${process.env.APP_NAME} • <a href="${process.env.APP_URL}">${process.env.APP_URL}</a></p>
      <p>© ${new Date().getFullYear()} ${process.env.APP_NAME}. All rights reserved.</p>
      <p>
        <a href="${process.env.APP_URL}/unsubscribe">ยกเลิกการรับอีเมล</a>
      </p>
    </div>
  </div>
</body>
</html>
  `.trim()
}
```

```typescript
// server/templates/emails/welcome.ts
import { createBaseTemplate } from './base'

interface WelcomeEmailData {
  name: string
  verificationUrl: string
  loginUrl: string
}

export function createWelcomeEmail(data: WelcomeEmailData): { html: string; text: string } {
  const content = `
    <h1 style="color: #1f2937; margin-bottom: 8px;">ยินดีต้อนรับ คุณ${data.name}! 🎉</h1>
    <p>ขอบคุณที่สมัครสมาชิกกับ ${process.env.APP_NAME} บัญชีของคุณถูกสร้างเรียบร้อยแล้ว</p>
    
    <p>กรุณายืนยันอีเมลของคุณเพื่อเริ่มใช้งาน:</p>
    
    <div style="text-align: center; margin: 32px 0;">
      <a href="${data.verificationUrl}" class="button">ยืนยันอีเมล</a>
    </div>
    
    <p style="color: #6b7280; font-size: 14px;">
      ลิงก์นี้จะหมดอายุใน 24 ชั่วโมง หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้
    </p>
    
    <hr class="divider">
    
    <h3>สิ่งที่คุณทำได้:</h3>
    <ul>
      <li>เรียนบทเรียน Vue.js และ Nuxt.js</li>
      <li>ดาวน์โหลด source code</li>
      <li>เข้าร่วม Discord community</li>
      <li>รับการอัปเดตบทเรียนใหม่</li>
    </ul>
  `
  
  const html = createBaseTemplate(content, {
    title: 'ยืนยันอีเมลของคุณ',
    previewText: 'ยืนยันอีเมลเพื่อเริ่มเรียน Vue.js!'
  })
  
  const text = `
ยินดีต้อนรับ คุณ${data.name}!

กรุณายืนยันอีเมลของคุณ: ${data.verificationUrl}

ลิงก์นี้จะหมดอายุใน 24 ชั่วโมง
  `.trim()
  
  return { html, text }
}
```

```typescript
// server/templates/emails/order-confirmation.ts
import { createBaseTemplate } from './base'
import type { Order } from '~/types/ecommerce'

export function createOrderConfirmationEmail(order: Order): { html: string; text: string } {
  const itemsHtml = order.items.map(item => `
    <tr>
      <td style="padding: 12px 0; border-bottom: 1px solid #e5e7eb;">
        <strong>${item.product.name}</strong>
        ${item.variant ? `<br><small>${item.variant.name}</small>` : ''}
        <br><small>x${item.quantity}</small>
      </td>
      <td style="padding: 12px 0; text-align: right; border-bottom: 1px solid #e5e7eb;">
        ฿${item.subtotal.toLocaleString('th-TH', { minimumFractionDigits: 2 })}
      </td>
    </tr>
  `).join('')

  const content = `
    <h1 style="color: #059669;">คำสั่งซื้อของคุณได้รับการยืนยันแล้ว ✓</h1>
    <p>ขอบคุณสำหรับการสั่งซื้อ! เราจะดำเนินการส่งสินค้าให้คุณโดยเร็วที่สุด</p>
    
    <div style="background: #f9fafb; border-radius: 8px; padding: 20px; margin: 24px 0;">
      <p><strong>หมายเลขคำสั่งซื้อ:</strong> ${order.orderNumber}</p>
      <p><strong>วันที่สั่งซื้อ:</strong> ${new Date(order.createdAt).toLocaleDateString('th-TH', { year: 'numeric', month: 'long', day: 'numeric' })}</p>
      <p><strong>สถานะ:</strong> ยืนยันแล้ว</p>
    </div>
    
    <h3>รายการสินค้า:</h3>
    <table style="width: 100%; border-collapse: collapse;">
      <tbody>${itemsHtml}</tbody>
      <tfoot>
        <tr>
          <td colspan="2"><hr class="divider"></td>
        </tr>
        <tr>
          <td>ยอดรวม</td>
          <td style="text-align: right;">฿${order.subtotal.toLocaleString('th-TH', { minimumFractionDigits: 2 })}</td>
        </tr>
        ${order.discount > 0 ? `
        <tr>
          <td>ส่วนลด</td>
          <td style="text-align: right; color: #059669;">-฿${order.discount.toLocaleString('th-TH', { minimumFractionDigits: 2 })}</td>
        </tr>` : ''}
        <tr>
          <td>ค่าจัดส่ง</td>
          <td style="text-align: right;">${order.shipping === 0 ? 'ฟรี' : `฿${order.shipping}`}</td>
        </tr>
        <tr>
          <td><strong>รวมทั้งหมด</strong></td>
          <td style="text-align: right;"><strong>฿${order.total.toLocaleString('th-TH', { minimumFractionDigits: 2 })}</strong></td>
        </tr>
      </tfoot>
    </table>
    
    <div style="text-align: center; margin: 32px 0;">
      <a href="${process.env.APP_URL}/orders/${order.id}" class="button">
        ติดตามคำสั่งซื้อ
      </a>
    </div>
  `
  
  return {
    html: createBaseTemplate(content, { title: `ยืนยันคำสั่งซื้อ ${order.orderNumber}` }),
    text: `ยืนยันคำสั่งซื้อ ${order.orderNumber} - ฿${order.total.toLocaleString('th-TH', { minimumFractionDigits: 2 })}`
  }
}
```

## Transactional Emails

```typescript
// server/services/transactional.ts
import { sendEmail } from './email'
import { createWelcomeEmail } from '../templates/emails/welcome'
import { createOrderConfirmationEmail } from '../templates/emails/order-confirmation'
import { createPasswordResetEmail } from '../templates/emails/password-reset'

export const EmailService = {
  async sendWelcome(user: { name: string; email: string; id: string }) {
    const token = await generateVerificationToken(user.id)
    const verificationUrl = `${process.env.APP_URL}/verify-email?token=${token}`
    
    const { html, text } = createWelcomeEmail({
      name: user.name,
      verificationUrl,
      loginUrl: `${process.env.APP_URL}/login`
    })
    
    return sendEmail({
      to: user.email,
      subject: `ยืนยันอีเมลของคุณ - ${process.env.APP_NAME}`,
      html,
      text
    })
  },
  
  async sendOrderConfirmation(order: any) {
    const { html, text } = createOrderConfirmationEmail(order)
    
    return sendEmail({
      to: order.user.email,
      subject: `ยืนยันคำสั่งซื้อ #${order.orderNumber}`,
      html,
      text
    })
  },
  
  async sendPasswordReset(user: { name: string; email: string; id: string }) {
    const token = await generatePasswordResetToken(user.id)
    const resetUrl = `${process.env.APP_URL}/reset-password?token=${token}`
    
    const { html, text } = createPasswordResetEmail({
      name: user.name,
      resetUrl
    })
    
    return sendEmail({
      to: user.email,
      subject: 'รีเซ็ตรหัสผ่านของคุณ',
      html,
      text
    })
  }
}
```

## Email Queue

สำหรับการส่ง email จำนวนมาก ควรใช้ Queue เพื่อไม่ให้ block request

```typescript
// server/services/emailQueue.ts
// Using BullMQ with Redis
import { Queue, Worker } from 'bullmq'
import { sendEmail } from './email'

interface EmailJob {
  to: string | string[]
  subject: string
  html: string
  text?: string
  priority?: number
}

const emailQueue = new Queue('email', {
  connection: {
    host: process.env.REDIS_HOST || 'localhost',
    port: Number(process.env.REDIS_PORT) || 6379
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000
    },
    removeOnComplete: { count: 100 },
    removeOnFail: { count: 50 }
  }
})

// Email worker
const worker = new Worker<EmailJob>(
  'email',
  async (job) => {
    await sendEmail({
      to: job.data.to,
      subject: job.data.subject,
      html: job.data.html,
      text: job.data.text
    })
  },
  {
    connection: emailQueue.opts.connection,
    concurrency: 5 // Process 5 emails at a time
  }
)

worker.on('completed', (job) => {
  console.log(`Email job ${job.id} completed`)
})

worker.on('failed', (job, err) => {
  console.error(`Email job ${job?.id} failed:`, err.message)
})

export async function queueEmail(data: EmailJob) {
  return emailQueue.add('send', data, {
    priority: data.priority || 0
  })
}

export async function queueBulkEmails(emails: EmailJob[]) {
  return emailQueue.addBulk(
    emails.map(data => ({ name: 'send', data }))
  )
}
```

## ตัวอย่าง: Registration Confirmation

```typescript
// server/api/auth/register.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const { name, email, password } = body
  
  // Validate
  if (!name || !email || !password) {
    throw createError({ statusCode: 400, message: 'Missing required fields' })
  }
  
  // Check if email exists
  const existingUser = await prisma.user.findUnique({ where: { email } })
  if (existingUser) {
    throw createError({ statusCode: 409, message: 'Email already registered' })
  }
  
  // Hash password
  const hashedPassword = await bcrypt.hash(password, 12)
  
  // Create user
  const user = await prisma.user.create({
    data: { name, email, password: hashedPassword }
  })
  
  // Send welcome email (using queue)
  await queueEmail({
    to: user.email,
    subject: `ยินดีต้อนรับสู่ ${process.env.APP_NAME}!`,
    html: createWelcomeEmail({ name: user.name, verificationUrl: '...', loginUrl: '...' }).html
  })
  
  return { success: true, message: 'กรุณาตรวจสอบอีเมลเพื่อยืนยันบัญชี' }
})
```

## Password Reset Email

```typescript
// server/templates/emails/password-reset.ts
import { createBaseTemplate } from './base'

interface PasswordResetData {
  name: string
  resetUrl: string
}

export function createPasswordResetEmail(data: PasswordResetData): { html: string; text: string } {
  const content = `
    <h1 style="color: #1f2937;">รีเซ็ตรหัสผ่าน</h1>
    <p>สวัสดี คุณ${data.name}</p>
    <p>เราได้รับคำขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณ คลิกปุ่มด้านล่างเพื่อตั้งรหัสผ่านใหม่:</p>
    
    <div style="text-align: center; margin: 32px 0;">
      <a href="${data.resetUrl}" class="button">รีเซ็ตรหัสผ่าน</a>
    </div>
    
    <p style="color: #6b7280; font-size: 14px;">
      ลิงก์นี้จะหมดอายุใน 1 ชั่วโมง
    </p>
    
    <hr class="divider">
    
    <div style="background: #fef3c7; border-radius: 8px; padding: 16px; margin: 24px 0;">
      <p style="margin: 0; color: #92400e;">
        <strong>⚠️ หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน</strong> กรุณาเพิกเฉยต่ออีเมลนี้ รหัสผ่านของคุณจะไม่เปลี่ยนแปลง
      </p>
    </div>
  `
  
  const html = createBaseTemplate(content, {
    title: 'รีเซ็ตรหัสผ่าน',
    previewText: 'รีเซ็ตรหัสผ่านบัญชีของคุณ'
  })
  
  const text = `
รีเซ็ตรหัสผ่าน

สวัสดี คุณ${data.name}

คลิกลิงก์เพื่อรีเซ็ตรหัสผ่าน: ${data.resetUrl}

ลิงก์นี้จะหมดอายุใน 1 ชั่วโมง
  `.trim()
  
  return { html, text }
}
```

## สรุป

Email System ที่ดีต้องมี:
1. Email service ที่เชื่อถือได้ (Resend, SendGrid)
2. HTML templates ที่ responsive
3. Queue สำหรับ bulk emails
4. Error handling และ retry logic
5. Email verification
6. Unsubscribe mechanism
7. Email preview ใน development
