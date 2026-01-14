# Email & SMS Services

## Overview

Email and SMS are essential communication channels for user notifications, authentication, marketing, and transactional messages. This tutorial covers integration with SendGrid, Twilio, and best practices for templating and deliverability.

## Practical Use Cases

- **Transactional emails**: Order confirmations, receipts, password resets
- **Marketing emails**: Newsletters, promotional campaigns
- **SMS verification**: Two-factor authentication, OTP codes
- **Notifications**: Real-time alerts via SMS
- **Welcome sequences**: Onboarding email series

## Service Comparison

| Service      | Best For             | Pricing         | Features                     |
| ------------ | -------------------- | --------------- | ---------------------------- |
| **SendGrid** | Email at scale       | Pay as you go   | Templates, analytics, API    |
| **AWS SES**  | Cost-effective email | Very cheap      | Reliable, scalable           |
| **Twilio**   | SMS/Voice            | Pay per message | Global coverage, MMS         |
| **Postmark** | Transactional email  | Simple pricing  | Fast delivery, great support |

## Step-by-Step Implementation

### 1. SendGrid Email Integration

```bash
npm install @sendgrid/mail handlebars
```

```typescript
// email/email.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { EmailService } from "./email.service";
import { EmailController } from "./email.controller";

@Module({
  imports: [ConfigModule],
  controllers: [EmailController],
  providers: [EmailService],
  exports: [EmailService],
})
export class EmailModule {}

// email/email.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import * as sgMail from "@sendgrid/mail";
import * as handlebars from "handlebars";
import * as fs from "fs";
import * as path from "path";

export interface EmailOptions {
  to: string | string[];
  subject: string;
  template?: string;
  context?: Record<string, any>;
  html?: string;
  text?: string;
  attachments?: Array<{
    content: string;
    filename: string;
    type: string;
  }>;
}

@Injectable()
export class EmailService {
  private readonly logger = new Logger(EmailService.name);

  constructor(private configService: ConfigService) {
    sgMail.setApiKey(this.configService.get("SENDGRID_API_KEY"));
  }

  async sendEmail(options: EmailOptions): Promise<void> {
    try {
      const { to, subject, template, context, html, text, attachments } =
        options;

      let emailHtml = html;
      let emailText = text;

      // If template provided, render it
      if (template && context) {
        emailHtml = await this.renderTemplate(template, context);
        emailText = this.stripHtml(emailHtml);
      }

      const msg: sgMail.MailDataRequired = {
        to: Array.isArray(to) ? to : [to],
        from: {
          email: this.configService.get("EMAIL_FROM"),
          name: this.configService.get("EMAIL_FROM_NAME", "MyApp"),
        },
        subject,
        html: emailHtml,
        text: emailText,
        attachments,
      };

      await sgMail.send(msg);
      this.logger.log(`Email sent to ${to}`);
    } catch (error) {
      this.logger.error(`Failed to send email: ${error.message}`, error.stack);
      throw error;
    }
  }

  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    return this.sendEmail({
      to: email,
      subject: "Welcome to MyApp!",
      template: "welcome",
      context: {
        name,
        appName: "MyApp",
        loginUrl: `${this.configService.get("FRONTEND_URL")}/login`,
      },
    });
  }

  async sendPasswordResetEmail(
    email: string,
    resetToken: string
  ): Promise<void> {
    const resetUrl = `${this.configService.get(
      "FRONTEND_URL"
    )}/reset-password?token=${resetToken}`;

    return this.sendEmail({
      to: email,
      subject: "Reset Your Password",
      template: "password-reset",
      context: {
        resetUrl,
        expiresIn: "1 hour",
      },
    });
  }

  async sendVerificationEmail(
    email: string,
    verificationToken: string
  ): Promise<void> {
    const verificationUrl = `${this.configService.get(
      "FRONTEND_URL"
    )}/verify-email?token=${verificationToken}`;

    return this.sendEmail({
      to: email,
      subject: "Verify Your Email",
      template: "email-verification",
      context: {
        verificationUrl,
      },
    });
  }

  async sendOrderConfirmation(email: string, orderDetails: any): Promise<void> {
    return this.sendEmail({
      to: email,
      subject: `Order Confirmation #${orderDetails.orderNumber}`,
      template: "order-confirmation",
      context: {
        orderNumber: orderDetails.orderNumber,
        items: orderDetails.items,
        total: orderDetails.total,
        shippingAddress: orderDetails.shippingAddress,
        trackingUrl: orderDetails.trackingUrl,
      },
    });
  }

  async sendBulkEmails(
    recipients: Array<{ email: string; context: Record<string, any> }>,
    subject: string,
    template: string
  ): Promise<void> {
    const messages = await Promise.all(
      recipients.map(async ({ email, context }) => ({
        to: email,
        from: {
          email: this.configService.get("EMAIL_FROM"),
          name: this.configService.get("EMAIL_FROM_NAME"),
        },
        subject,
        html: await this.renderTemplate(template, context),
      }))
    );

    try {
      await sgMail.send(messages);
      this.logger.log(`Bulk email sent to ${recipients.length} recipients`);
    } catch (error) {
      this.logger.error("Failed to send bulk email", error.stack);
      throw error;
    }
  }

  private async renderTemplate(
    templateName: string,
    context: Record<string, any>
  ): Promise<string> {
    const templatePath = path.join(
      process.cwd(),
      "src/email/templates",
      `${templateName}.hbs`
    );

    const templateContent = fs.readFileSync(templatePath, "utf-8");
    const template = handlebars.compile(templateContent);

    return template(context);
  }

  private stripHtml(html: string): string {
    return html.replace(/<[^>]*>?/gm, "");
  }
}
```

### 2. Email Templates (Handlebars)

```html
<!-- src/email/templates/welcome.hbs -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <style>
      body {
        font-family: Arial, sans-serif;
        line-height: 1.6;
        color: #333;
        max-width: 600px;
        margin: 0 auto;
        padding: 20px;
      }
      .header {
        background-color: #4f46e5;
        color: white;
        padding: 20px;
        text-align: center;
        border-radius: 8px 8px 0 0;
      }
      .content {
        background-color: #f9fafb;
        padding: 30px;
        border-radius: 0 0 8px 8px;
      }
      .button {
        display: inline-block;
        padding: 12px 24px;
        background-color: #4f46e5;
        color: white;
        text-decoration: none;
        border-radius: 6px;
        margin: 20px 0;
      }
      .footer {
        text-align: center;
        margin-top: 30px;
        color: #6b7280;
        font-size: 14px;
      }
    </style>
  </head>
  <body>
    <div class="header">
      <h1>Welcome to {{appName}}!</h1>
    </div>
    <div class="content">
      <p>Hi {{name}},</p>
      <p>
        We're excited to have you on board! You can now access all the features
        of {{appName}}.
      </p>
      <p>
        <a href="{{loginUrl}}" class="button">Get Started</a>
      </p>
      <p>If you have any questions, feel free to reply to this email.</p>
      <p>Best regards,<br />The {{appName}} Team</p>
    </div>
    <div class="footer">
      <p>You received this email because you signed up for {{appName}}.</p>
    </div>
  </body>
</html>

<!-- src/email/templates/password-reset.hbs -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <style>
      /* Same styles as above */
    </style>
  </head>
  <body>
    <div class="header">
      <h1>Reset Your Password</h1>
    </div>
    <div class="content">
      <p>Hi there,</p>
      <p>
        We received a request to reset your password. Click the button below to
        create a new password:
      </p>
      <p>
        <a href="{{resetUrl}}" class="button">Reset Password</a>
      </p>
      <p>This link will expire in {{expiresIn}}.</p>
      <p>If you didn't request this, you can safely ignore this email.</p>
    </div>
    <div class="footer">
      <p>For security reasons, this link will only work once.</p>
    </div>
  </body>
</html>
```

### 3. Twilio SMS Integration

```bash
npm install twilio
```

```typescript
// sms/sms.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { SmsService } from "./sms.service";

@Module({
  imports: [ConfigModule],
  providers: [SmsService],
  exports: [SmsService],
})
export class SmsModule {}

// sms/sms.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import * as Twilio from "twilio";

@Injectable()
export class SmsService {
  private readonly logger = new Logger(SmsService.name);
  private twilioClient: Twilio.Twilio;
  private fromNumber: string;

  constructor(private configService: ConfigService) {
    this.twilioClient = Twilio(
      this.configService.get("TWILIO_ACCOUNT_SID"),
      this.configService.get("TWILIO_AUTH_TOKEN")
    );
    this.fromNumber = this.configService.get("TWILIO_PHONE_NUMBER");
  }

  async sendSms(to: string, message: string): Promise<void> {
    try {
      const result = await this.twilioClient.messages.create({
        body: message,
        from: this.fromNumber,
        to,
      });

      this.logger.log(`SMS sent to ${to}: ${result.sid}`);
    } catch (error) {
      this.logger.error(`Failed to send SMS: ${error.message}`, error.stack);
      throw error;
    }
  }

  async sendVerificationCode(phoneNumber: string, code: string): Promise<void> {
    const message = `Your verification code is: ${code}. Valid for 10 minutes.`;
    return this.sendSms(phoneNumber, message);
  }

  async sendOTP(phoneNumber: string): Promise<string> {
    // Generate 6-digit OTP
    const otp = Math.floor(100000 + Math.random() * 900000).toString();

    await this.sendVerificationCode(phoneNumber, otp);

    return otp; // Store this in cache/database with expiration
  }

  async sendNotification(
    phoneNumber: string,
    notification: string
  ): Promise<void> {
    return this.sendSms(phoneNumber, notification);
  }

  async sendBulkSms(
    recipients: Array<{ phoneNumber: string; message: string }>
  ): Promise<void> {
    const promises = recipients.map(({ phoneNumber, message }) =>
      this.sendSms(phoneNumber, message).catch((error) => {
        this.logger.error(`Failed to send SMS to ${phoneNumber}`, error);
        return null;
      })
    );

    await Promise.allSettled(promises);
  }

  // Using Twilio Verify API (better for OTP)
  async sendVerifyOTP(phoneNumber: string): Promise<void> {
    try {
      await this.twilioClient.verify.v2
        .services(this.configService.get("TWILIO_VERIFY_SERVICE_SID"))
        .verifications.create({
          to: phoneNumber,
          channel: "sms",
        });

      this.logger.log(`Verification OTP sent to ${phoneNumber}`);
    } catch (error) {
      this.logger.error("Failed to send verification OTP", error.stack);
      throw error;
    }
  }

  async verifyOTP(phoneNumber: string, code: string): Promise<boolean> {
    try {
      const verification = await this.twilioClient.verify.v2
        .services(this.configService.get("TWILIO_VERIFY_SERVICE_SID"))
        .verificationChecks.create({
          to: phoneNumber,
          code,
        });

      return verification.status === "approved";
    } catch (error) {
      this.logger.error("Failed to verify OTP", error.stack);
      return false;
    }
  }
}
```

### 4. Notification Service (Unified)

```typescript
// notifications/notifications.service.ts
import { Injectable } from "@nestjs/common";
import { EmailService } from "../email/email.service";
import { SmsService } from "../sms/sms.service";

export interface NotificationOptions {
  userId: string;
  type: "email" | "sms" | "both";
  channel: "transactional" | "marketing";
  template?: string;
  context?: Record<string, any>;
}

@Injectable()
export class NotificationsService {
  constructor(
    private emailService: EmailService,
    private smsService: SmsService
  ) {}

  async sendNotification(
    recipient: { email?: string; phone?: string },
    message: { subject?: string; body: string },
    options: NotificationOptions
  ): Promise<void> {
    const promises: Promise<void>[] = [];

    if (
      (options.type === "email" || options.type === "both") &&
      recipient.email
    ) {
      promises.push(
        this.emailService.sendEmail({
          to: recipient.email,
          subject: message.subject || "Notification",
          html: message.body,
        })
      );
    }

    if (
      (options.type === "sms" || options.type === "both") &&
      recipient.phone
    ) {
      promises.push(this.smsService.sendSms(recipient.phone, message.body));
    }

    await Promise.allSettled(promises);
  }

  async sendWelcome(email: string, name: string): Promise<void> {
    return this.emailService.sendWelcomeEmail(email, name);
  }

  async sendPasswordReset(email: string, token: string): Promise<void> {
    return this.emailService.sendPasswordResetEmail(email, token);
  }

  async send2FACode(phoneNumber: string): Promise<string> {
    return this.smsService.sendOTP(phoneNumber);
  }

  async verify2FACode(phoneNumber: string, code: string): Promise<boolean> {
    return this.smsService.verifyOTP(phoneNumber, code);
  }
}
```

### 5. Queue for Reliable Delivery

```bash
npm install @nestjs/bull bull
npm install -D @types/bull
```

```typescript
// email/email-queue.processor.ts
import { Process, Processor } from "@nestjs/bull";
import { Logger } from "@nestjs/common";
import { Job } from "bull";
import { EmailService, EmailOptions } from "./email.service";

@Processor("email")
export class EmailQueueProcessor {
  private readonly logger = new Logger(EmailQueueProcessor.name);

  constructor(private emailService: EmailService) {}

  @Process("send-email")
  async handleSendEmail(job: Job<EmailOptions>) {
    this.logger.log(`Processing email job ${job.id}`);

    try {
      await this.emailService.sendEmail(job.data);
      this.logger.log(`Email job ${job.id} completed`);
    } catch (error) {
      this.logger.error(`Email job ${job.id} failed`, error.stack);
      throw error; // Will trigger retry
    }
  }

  @Process("send-bulk-email")
  async handleBulkEmail(job: Job) {
    this.logger.log(`Processing bulk email job ${job.id}`);

    const { recipients, subject, template } = job.data;

    try {
      await this.emailService.sendBulkEmails(recipients, subject, template);
      this.logger.log(`Bulk email job ${job.id} completed`);
    } catch (error) {
      this.logger.error(`Bulk email job ${job.id} failed`, error.stack);
      throw error;
    }
  }
}

// email/email-queue.service.ts
import { Injectable } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bull";
import { Queue } from "bull";
import { EmailOptions } from "./email.service";

@Injectable()
export class EmailQueueService {
  constructor(@InjectQueue("email") private emailQueue: Queue) {}

  async queueEmail(options: EmailOptions): Promise<void> {
    await this.emailQueue.add("send-email", options, {
      attempts: 3,
      backoff: {
        type: "exponential",
        delay: 2000,
      },
    });
  }

  async queueBulkEmail(
    recipients: any[],
    subject: string,
    template: string
  ): Promise<void> {
    await this.emailQueue.add(
      "send-bulk-email",
      { recipients, subject, template },
      {
        attempts: 3,
        backoff: {
          type: "exponential",
          delay: 5000,
        },
      }
    );
  }
}
```

### 6. Rate Limiting for SMS

```typescript
// sms/sms-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import { Redis } from "ioredis";

@Injectable()
export class SmsRateLimiterService {
  constructor(@InjectRedis() private redis: Redis) {}

  async canSendSms(phoneNumber: string): Promise<boolean> {
    const key = `sms:ratelimit:${phoneNumber}`;
    const count = await this.redis.get(key);

    if (count && parseInt(count) >= 3) {
      return false; // Max 3 SMS per hour
    }

    return true;
  }

  async incrementSmsCount(phoneNumber: string): Promise<void> {
    const key = `sms:ratelimit:${phoneNumber}`;
    const current = await this.redis.incr(key);

    if (current === 1) {
      await this.redis.expire(key, 3600); // 1 hour
    }
  }
}
```

## Best Practices

### 1. Deliverability

```typescript
// Use proper SPF, DKIM, DMARC records
// Warm up IP addresses gradually
// Monitor bounce rates and complaints
// Use double opt-in for marketing emails
// Implement unsubscribe links
```

### 2. Template Management

```typescript
// Store templates in database or file system
// Version templates
// Preview before sending
// Test with different data
// Use consistent branding
```

### 3. Error Handling

```typescript
try {
  await emailService.sendEmail(options);
} catch (error) {
  if (error.code === 403) {
    // Rate limit exceeded
    await queueForLater(options);
  } else if (error.code === 550) {
    // Invalid recipient
    await markAsInvalid(recipient);
  } else {
    // Retry
    throw error;
  }
}
```

### 4. Testing

```typescript
// Use test mode APIs
const testRecipient = process.env.TEST_EMAIL;

// Mock in tests
const mockEmailService = {
  sendEmail: jest.fn().mockResolvedValue(undefined),
};

// Capture emails in development
// Use services like Mailtrap or MailHog
```

## Key Takeaways

1. **Queue emails** - Don't block requests
2. **Handle failures** - Implement retries
3. **Rate limit SMS** - Prevent abuse
4. **Use templates** - Maintain consistency
5. **Monitor deliverability** - Track bounces
6. **Test thoroughly** - Use test modes
7. **Comply with laws** - CAN-SPAM, GDPR
8. **Unsubscribe** - Always provide option

Reliable email and SMS delivery requires proper infrastructure, monitoring, and compliance with regulations!
