# Email & SMS Services

## Overview

Email and SMS are critical communication channels for modern applications. This guide covers integrating Nodemailer for transactional emails, Twilio for SMS, template engines, and building reliable notification systems with Express.

## Email with Nodemailer

### 1. Setup

```bash
npm install nodemailer
npm install -D @types/nodemailer
```

```typescript
// src/config/email.ts
import nodemailer from 'nodemailer';

const createTransporter = () => {
  if (process.env.NODE_ENV === 'production') {
    // Production: Use real SMTP service (e.g., SendGrid, AWS SES)
    return nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT || '587'),
      secure: process.env.SMTP_PORT === '465',
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASSWORD,
      },
    });
  } else {
    // Development: Use Ethereal (fake SMTP for testing)
    return nodemailer.createTransport({
      host: 'smtp.ethereal.email',
      port: 587,
      auth: {
        user: process.env.ETHEREAL_USER,
        pass: process.env.ETHEREAL_PASSWORD,
      },
    });
  }
};

export const transporter = createTransporter();

export default transporter;
```

```env
# .env
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your_sendgrid_api_key
FROM_EMAIL=noreply@yourdomain.com
FROM_NAME=Your App Name
```

### 2. Email Service

```typescript
// src/services/email.service.ts
import transporter from '../config/email';
import { AppError } from '../utils/AppError';

interface EmailOptions {
  to: string;
  subject: string;
  text?: string;
  html?: string;
}

export class EmailService {
  private fromEmail: string;
  private fromName: string;

  constructor() {
    this.fromEmail = process.env.FROM_EMAIL || 'noreply@example.com';
    this.fromName = process.env.FROM_NAME || 'App';
  }

  async sendEmail({ to, subject, text, html }: EmailOptions) {
    try {
      const info = await transporter.sendMail({
        from: `"${this.fromName}" <${this.fromEmail}>`,
        to,
        subject,
        text,
        html,
      });

      console.log('Email sent:', info.messageId);
      
      if (process.env.NODE_ENV === 'development') {
        console.log('Preview URL:', nodemailer.getTestMessageUrl(info));
      }

      return info;
    } catch (error) {
      console.error('Email sending failed:', error);
      throw new AppError('Failed to send email', 500);
    }
  }

  async sendWelcomeEmail(email: string, name: string) {
    const subject = `Welcome to ${this.fromName}!`;
    const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #333;">Welcome, ${name}!</h1>
        <p>Thanks for signing up. We're excited to have you on board.</p>
        <p>Get started by exploring our features:</p>
        <a href="${process.env.APP_URL}/dashboard" style="
          display: inline-block;
          padding: 12px 24px;
          background-color: #007bff;
          color: #fff;
          text-decoration: none;
          border-radius: 4px;
        ">Go to Dashboard</a>
        <p style="margin-top: 20px; color: #666;">
          If you have any questions, feel free to reply to this email.
        </p>
      </div>
    `;

    return this.sendEmail({ to: email, subject, html });
  }

  async sendPasswordResetEmail(email: string, token: string) {
    const resetUrl = `${process.env.APP_URL}/reset-password?token=${token}`;
    const subject = 'Password Reset Request';
    const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #333;">Password Reset</h1>
        <p>You requested to reset your password. Click the button below:</p>
        <a href="${resetUrl}" style="
          display: inline-block;
          padding: 12px 24px;
          background-color: #dc3545;
          color: #fff;
          text-decoration: none;
          border-radius: 4px;
        ">Reset Password</a>
        <p style="margin-top: 20px; color: #666;">
          This link expires in 1 hour.
        </p>
        <p style="color: #666;">
          If you didn't request this, please ignore this email.
        </p>
      </div>
    `;

    return this.sendEmail({ to: email, subject, html });
  }

  async sendVerificationEmail(email: string, token: string) {
    const verifyUrl = `${process.env.APP_URL}/verify-email?token=${token}`;
    const subject = 'Verify Your Email';
    const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #333;">Email Verification</h1>
        <p>Please verify your email address by clicking the button below:</p>
        <a href="${verifyUrl}" style="
          display: inline-block;
          padding: 12px 24px;
          background-color: #28a745;
          color: #fff;
          text-decoration: none;
          border-radius: 4px;
        ">Verify Email</a>
        <p style="margin-top: 20px; color: #666;">
          This link expires in 24 hours.
        </p>
      </div>
    `;

    return this.sendEmail({ to: email, subject, html });
  }

  async sendOrderConfirmation(email: string, orderDetails: any) {
    const subject = `Order Confirmation #${orderDetails.id}`;
    const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #333;">Order Confirmed!</h1>
        <p>Thank you for your order. Here are the details:</p>
        <div style="background-color: #f8f9fa; padding: 20px; border-radius: 4px;">
          <p><strong>Order #:</strong> ${orderDetails.id}</p>
          <p><strong>Total:</strong> $${orderDetails.total}</p>
          <p><strong>Status:</strong> ${orderDetails.status}</p>
        </div>
        <a href="${process.env.APP_URL}/orders/${orderDetails.id}" style="
          display: inline-block;
          margin-top: 20px;
          padding: 12px 24px;
          background-color: #007bff;
          color: #fff;
          text-decoration: none;
          border-radius: 4px;
        ">View Order</a>
      </div>
    `;

    return this.sendEmail({ to: email, subject, html });
  }
}

export default new EmailService();
```

## SMS with Twilio

### 1. Setup

```bash
npm install twilio
```

```typescript
// src/config/twilio.ts
import twilio from 'twilio';

if (!process.env.TWILIO_ACCOUNT_SID || !process.env.TWILIO_AUTH_TOKEN) {
  throw new Error('Twilio credentials are required');
}

export const twilioClient = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

export default twilioClient;
```

```env
# .env
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+1234567890
```

### 2. SMS Service

```typescript
// src/services/sms.service.ts
import twilioClient from '../config/twilio';
import { AppError } from '../utils/AppError';

export class SMSService {
  private fromNumber: string;

  constructor() {
    this.fromNumber = process.env.TWILIO_PHONE_NUMBER || '';
  }

  async sendSMS(to: string, message: string) {
    try {
      const result = await twilioClient.messages.create({
        body: message,
        from: this.fromNumber,
        to,
      });

      console.log('SMS sent:', result.sid);
      return result;
    } catch (error) {
      console.error('SMS sending failed:', error);
      throw new AppError('Failed to send SMS', 500);
    }
  }

  async sendVerificationCode(phoneNumber: string, code: string) {
    const message = `Your verification code is: ${code}. It expires in 10 minutes.`;
    return this.sendSMS(phoneNumber, message);
  }

  async sendOrderNotification(phoneNumber: string, orderId: string) {
    const message = `Your order #${orderId} has been confirmed. Track it at ${process.env.APP_URL}/orders/${orderId}`;
    return this.sendSMS(phoneNumber, message);
  }

  async sendSecurityAlert(phoneNumber: string, alertType: string) {
    const message = `Security Alert: ${alertType} detected on your account. If this wasn't you, secure your account immediately.`;
    return this.sendSMS(phoneNumber, message);
  }
}

export default new SMSService();
```

## Notification Service (Unified)

```typescript
// src/services/notification.service.ts
import emailService from './email.service';
import smsService from './sms.service';
import { prisma } from '../config/database';

type NotificationChannel = 'email' | 'sms' | 'both';

interface NotificationOptions {
  userId: string;
  channel: NotificationChannel;
  type: string;
  data: any;
}

export class NotificationService {
  async sendNotification({ userId, channel, type, data }: NotificationOptions) {
    const user = await prisma.user.findUnique({ where: { id: userId } });

    if (!user) {
      throw new Error('User not found');
    }

    const promises: Promise<any>[] = [];

    if (channel === 'email' || channel === 'both') {
      promises.push(this.sendEmailNotification(user.email, type, data));
    }

    if (channel === 'sms' || channel === 'both') {
      if (user.phoneNumber) {
        promises.push(this.sendSMSNotification(user.phoneNumber, type, data));
      }
    }

    await Promise.allSettled(promises);

    // Log notification
    await prisma.notification.create({
      data: {
        userId,
        type,
        channel,
        status: 'sent',
      },
    });
  }

  private async sendEmailNotification(email: string, type: string, data: any) {
    switch (type) {
      case 'welcome':
        return emailService.sendWelcomeEmail(email, data.name);
      case 'password_reset':
        return emailService.sendPasswordResetEmail(email, data.token);
      case 'email_verification':
        return emailService.sendVerificationEmail(email, data.token);
      case 'order_confirmation':
        return emailService.sendOrderConfirmation(email, data.order);
      default:
        throw new Error(`Unknown email notification type: ${type}`);
    }
  }

  private async sendSMSNotification(phoneNumber: string, type: string, data: any) {
    switch (type) {
      case 'verification_code':
        return smsService.sendVerificationCode(phoneNumber, data.code);
      case 'order_notification':
        return smsService.sendOrderNotification(phoneNumber, data.orderId);
      case 'security_alert':
        return smsService.sendSecurityAlert(phoneNumber, data.alertType);
      default:
        throw new Error(`Unknown SMS notification type: ${type}`);
    }
  }

  async getUserNotificationPreferences(userId: string) {
    return prisma.notificationPreference.findUnique({
      where: { userId },
    });
  }

  async updateNotificationPreferences(userId: string, preferences: any) {
    return prisma.notificationPreference.upsert({
      where: { userId },
      update: preferences,
      create: { userId, ...preferences },
    });
  }
}

export default new NotificationService();
```

## Email Templates with Handlebars

```bash
npm install handlebars
npm install -D @types/handlebars
```

```typescript
// src/services/template.service.ts
import Handlebars from 'handlebars';
import fs from 'fs/promises';
import path from 'path';

export class TemplateService {
  private templatesDir = path.join(__dirname, '../templates/emails');
  private cache = new Map<string, HandlebarsTemplateDelegate>();

  async renderTemplate(templateName: string, data: any): Promise<string> {
    let template = this.cache.get(templateName);

    if (!template) {
      const templatePath = path.join(this.templatesDir, `${templateName}.hbs`);
      const templateContent = await fs.readFile(templatePath, 'utf-8');
      template = Handlebars.compile(templateContent);
      this.cache.set(templateName, template);
    }

    return template(data);
  }
}

export default new TemplateService();
```

```handlebars
{{!-- src/templates/emails/welcome.hbs --}}
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; }
    .container { max-width: 600px; margin: 0 auto; }
    .button { 
      display: inline-block; 
      padding: 12px 24px; 
      background-color: #007bff; 
      color: #fff; 
      text-decoration: none; 
      border-radius: 4px; 
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Welcome, {{name}}!</h1>
    <p>Thanks for signing up for {{appName}}.</p>
    <a href="{{dashboardUrl}}" class="button">Get Started</a>
  </div>
</body>
</html>
```

```typescript
// Updated EmailService to use templates
async sendWelcomeEmail(email: string, name: string) {
  const html = await templateService.renderTemplate('welcome', {
    name,
    appName: this.fromName,
    dashboardUrl: `${process.env.APP_URL}/dashboard`,
  });

  return this.sendEmail({
    to: email,
    subject: `Welcome to ${this.fromName}!`,
    html,
  });
}
```

## Background Email Queue (Bull)

```typescript
// src/queues/email.queue.ts
import Queue from 'bull';
import emailService from '../services/email.service';

export const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

emailQueue.process(async (job) => {
  const { to, subject, html } = job.data;
  await emailService.sendEmail({ to, subject, html });
});

export const queueEmail = async (to: string, subject: string, html: string) => {
  await emailQueue.add({ to, subject, html }, {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  });
};
```

## Controller

```typescript
// src/controllers/notification.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import notificationService from '../services/notification.service';

export class NotificationController {
  sendNotification = asyncHandler(async (req: Request, res: Response) => {
    const { userId, channel, type, data } = req.body;

    await notificationService.sendNotification({ userId, channel, type, data });

    res.json({
      success: true,
      message: 'Notification sent',
    });
  });

  getPreferences = asyncHandler(async (req: Request, res: Response) => {
    const userId = req.user!.id;

    const preferences = await notificationService.getUserNotificationPreferences(userId);

    res.json({
      success: true,
      data: preferences,
    });
  });

  updatePreferences = asyncHandler(async (req: Request, res: Response) => {
    const userId = req.user!.id;
    const preferences = req.body;

    const updated = await notificationService.updateNotificationPreferences(
      userId,
      preferences
    );

    res.json({
      success: true,
      data: updated,
    });
  });
}

export default new NotificationController();
```

## Routes

```typescript
// src/routes/notification.routes.ts
import { Router } from 'express';
import notificationController from '../controllers/notification.controller';
import { authenticate } from '../middleware/auth.middleware';

const router = Router();

router.use(authenticate);

router.post('/send', notificationController.sendNotification);
router.get('/preferences', notificationController.getPreferences);
router.patch('/preferences', notificationController.updatePreferences);

export default router;
```

## Testing

```typescript
// src/__tests__/notification.test.ts
import request from 'supertest';
import app from '../server';

describe('Notification System', () => {
  let authToken: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = res.body.data.accessToken;
  });

  describe('Email Service', () => {
    it('should send a welcome email', async () => {
      const result = await emailService.sendWelcomeEmail(
        'test@example.com',
        'Test User'
      );

      expect(result.messageId).toBeDefined();
    });
  });

  describe('SMS Service', () => {
    it('should send a verification code', async () => {
      const result = await smsService.sendVerificationCode(
        '+1234567890',
        '123456'
      );

      expect(result.sid).toBeDefined();
    });
  });

  describe('POST /api/v1/notifications/preferences', () => {
    it('should update notification preferences', async () => {
      const res = await request(app)
        .patch('/api/v1/notifications/preferences')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          emailNotifications: true,
          smsNotifications: false,
        });

      expect(res.status).toBe(200);
      expect(res.body.data.emailNotifications).toBe(true);
    });
  });
});
```

## Best Practices

1. **Use transactional email services** (SendGrid, AWS SES, Mailgun)
2. **Queue emails** for better performance
3. **Implement retry logic** for failed deliveries
4. **Track delivery status** via webhooks
5. **Use templates** for consistent branding
6. **Test with Ethereal** in development
7. **Validate phone numbers** before sending SMS
8. **Respect user preferences** for notifications
9. **Monitor delivery rates** and bounce rates
10. **Comply with CAN-SPAM** and GDPR regulations

## Key Takeaways

1. **Nodemailer** handles email delivery
2. **Twilio** powers SMS functionality
3. **Templates** ensure consistent messaging
4. **Queues** improve performance
5. **Retry logic** handles failures gracefully
6. **User preferences** enable opt-out
7. **Unified service** simplifies multi-channel notifications
8. **Testing tools** streamline development

Reliable notifications build user trust and improve engagement.
