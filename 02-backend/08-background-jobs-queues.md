# Background Jobs & Queue Systems

## Overview

Background jobs and queue systems allow you to offload time-consuming or resource-intensive tasks from your main application thread, improving response times and user experience. Tasks are processed asynchronously by worker processes, enabling better scalability and reliability.

**Key Technologies:**

- **Bull/BullMQ**: Redis-based queue system for Node.js with priority, scheduling, and retry support
- **Redis**: In-memory data store used as the queue backend
- **@nestjs/bull**: NestJS integration for Bull queues
- **Agenda**: MongoDB-based job scheduling library (alternative)

## Practical Use Cases

### 1. **Email Sending**

Deferred email delivery

- Welcome emails after registration
- Password reset emails
- Newsletter campaigns
- Order confirmations

### 2. **Image/Video Processing**

Media manipulation tasks

- Thumbnail generation
- Image optimization
- Video transcoding
- PDF generation

### 3. **Data Processing**

Heavy computation tasks

- Report generation
- Data exports (CSV, Excel)
- Analytics aggregation
- Batch updates

### 4. **External API Calls**

Third-party service integration

- Social media posting
- Payment processing callbacks
- Webhook deliveries
- Data synchronization

### 5. **Scheduled Tasks**

Recurring operations

- Database cleanup
- Cache invalidation
- Subscription renewals
- Reminder notifications

## Step-by-Step Implementation

### 1. Install Dependencies

```bash
npm install @nestjs/bull bull
npm install redis

# For BullMQ (Bull v4+)
npm install bullmq

# Optional: Bull Board for UI
npm install @bull-board/api @bull-board/nestjs
```

### 2. Configure Redis and Bull Module

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bull";
import { ConfigModule, ConfigService } from "@nestjs/config";

@Module({
  imports: [
    ConfigModule.forRoot(),

    // Global Bull configuration
    BullModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        redis: {
          host: configService.get("REDIS_HOST", "localhost"),
          port: configService.get("REDIS_PORT", 6379),
          password: configService.get("REDIS_PASSWORD"),
          maxRetriesPerRequest: null, // Required for Bull
        },
        defaultJobOptions: {
          removeOnComplete: 100, // Keep last 100 completed jobs
          removeOnFail: 1000, // Keep last 1000 failed jobs
          attempts: 3, // Retry failed jobs 3 times
          backoff: {
            type: "exponential",
            delay: 2000, // Start with 2 seconds
          },
        },
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

### 3. Email Queue Implementation

```typescript
// email/email.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bull";
import { EmailService } from "./email.service";
import { EmailProcessor } from "./email.processor";
import { EmailController } from "./email.controller";

@Module({
  imports: [
    BullModule.registerQueue({
      name: "email",
      defaultJobOptions: {
        attempts: 5,
        backoff: {
          type: "exponential",
          delay: 1000,
        },
      },
    }),
  ],
  controllers: [EmailController],
  providers: [EmailService, EmailProcessor],
  exports: [EmailService],
})
export class EmailModule {}
```

```typescript
// email/email.service.ts
import { Injectable } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bull";
import { Queue, Job } from "bull";

export interface EmailJobData {
  to: string;
  subject: string;
  template: string;
  context: Record<string, any>;
  priority?: number;
}

@Injectable()
export class EmailService {
  constructor(@InjectQueue("email") private emailQueue: Queue) {}

  // Add email to queue
  async sendEmail(data: EmailJobData): Promise<Job<EmailJobData>> {
    return this.emailQueue.add("send-email", data, {
      priority: data.priority || 5,
      removeOnComplete: true,
    });
  }

  // Schedule email for future delivery
  async scheduleEmail(
    data: EmailJobData,
    delay: number
  ): Promise<Job<EmailJobData>> {
    return this.emailQueue.add("send-email", data, {
      delay, // milliseconds
      priority: data.priority || 5,
    });
  }

  // Batch email sending
  async sendBulkEmails(emails: EmailJobData[]): Promise<Job<EmailJobData>[]> {
    const jobs = emails.map((email) => ({
      name: "send-email",
      data: email,
      opts: {
        priority: email.priority || 5,
      },
    }));

    return this.emailQueue.addBulk(jobs);
  }

  // Get job status
  async getJobStatus(jobId: string): Promise<any> {
    const job = await this.emailQueue.getJob(jobId);

    if (!job) {
      return null;
    }

    const state = await job.getState();
    const progress = job.progress();
    const failedReason = job.failedReason;

    return {
      id: job.id,
      state,
      progress,
      failedReason,
      data: job.data,
      timestamp: job.timestamp,
    };
  }

  // Retry failed job
  async retryJob(jobId: string): Promise<void> {
    const job = await this.emailQueue.getJob(jobId);
    if (job) {
      await job.retry();
    }
  }

  // Get queue statistics
  async getQueueStats() {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      this.emailQueue.getWaitingCount(),
      this.emailQueue.getActiveCount(),
      this.emailQueue.getCompletedCount(),
      this.emailQueue.getFailedCount(),
      this.emailQueue.getDelayedCount(),
    ]);

    return {
      waiting,
      active,
      completed,
      failed,
      delayed,
      total: waiting + active + completed + failed + delayed,
    };
  }
}
```

```typescript
// email/email.processor.ts
import {
  Process,
  Processor,
  OnQueueActive,
  OnQueueCompleted,
  OnQueueFailed,
} from "@nestjs/bull";
import { Job } from "bull";
import { Logger } from "@nestjs/common";
import { EmailJobData } from "./email.service";
import * as nodemailer from "nodemailer";

@Processor("email")
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);
  private transporter: nodemailer.Transporter;

  constructor() {
    // Initialize email transporter
    this.transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT || "587"),
      secure: false,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });
  }

  @Process("send-email")
  async handleSendEmail(job: Job<EmailJobData>): Promise<void> {
    this.logger.log(`Processing email job ${job.id} to ${job.data.to}`);

    try {
      // Update job progress
      await job.progress(10);

      // Render email template
      const html = await this.renderTemplate(
        job.data.template,
        job.data.context
      );

      await job.progress(50);

      // Send email
      const info = await this.transporter.sendMail({
        from: process.env.EMAIL_FROM || "noreply@example.com",
        to: job.data.to,
        subject: job.data.subject,
        html,
      });

      await job.progress(100);

      this.logger.log(`Email sent successfully: ${info.messageId}`);
    } catch (error) {
      this.logger.error(`Failed to send email: ${error.message}`, error.stack);
      throw error; // Will trigger retry
    }
  }

  @OnQueueActive()
  onActive(job: Job) {
    this.logger.debug(`Processing job ${job.id} of type ${job.name}`);
  }

  @OnQueueCompleted()
  onCompleted(job: Job) {
    this.logger.debug(`Completed job ${job.id} of type ${job.name}`);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(
      `Failed job ${job.id} of type ${job.name}: ${error.message}`,
      error.stack
    );
  }

  private async renderTemplate(
    template: string,
    context: Record<string, any>
  ): Promise<string> {
    // Simple template rendering (use Handlebars/EJS in production)
    let html = template;

    Object.entries(context).forEach(([key, value]) => {
      html = html.replace(new RegExp(`{{${key}}}`, "g"), String(value));
    });

    return html;
  }
}
```

### 4. Image Processing Queue

```typescript
// image/image.service.ts
import { Injectable } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bull";
import { Queue } from "bull";

export interface ImageJobData {
  filePath: string;
  operations: {
    resize?: { width: number; height: number };
    format?: "jpeg" | "png" | "webp";
    quality?: number;
    thumbnail?: boolean;
  };
  userId: string;
}

@Injectable()
export class ImageService {
  constructor(@InjectQueue("image") private imageQueue: Queue) {}

  async processImage(data: ImageJobData) {
    return this.imageQueue.add("process-image", data, {
      attempts: 2,
      timeout: 30000, // 30 seconds timeout
    });
  }

  async generateThumbnails(filePath: string, userId: string) {
    const sizes = [
      { width: 150, height: 150, name: "thumbnail" },
      { width: 300, height: 300, name: "small" },
      { width: 600, height: 600, name: "medium" },
    ];

    const jobs = sizes.map((size) => ({
      name: "process-image",
      data: {
        filePath,
        userId,
        operations: {
          resize: { width: size.width, height: size.height },
          format: "webp" as const,
          quality: 80,
        },
      },
    }));

    return this.imageQueue.addBulk(jobs);
  }
}
```

```typescript
// image/image.processor.ts
import { Process, Processor } from "@nestjs/bull";
import { Job } from "bull";
import { Logger } from "@nestjs/common";
import * as sharp from "sharp";
import * as path from "path";
import * as fs from "fs/promises";
import { ImageJobData } from "./image.service";

@Processor("image")
export class ImageProcessor {
  private readonly logger = new Logger(ImageProcessor.name);

  @Process("process-image")
  async handleImageProcessing(job: Job<ImageJobData>): Promise<string> {
    const { filePath, operations, userId } = job.data;

    this.logger.log(`Processing image: ${filePath}`);

    try {
      await job.progress(10);

      // Load image
      let image = sharp(filePath);

      // Apply resize
      if (operations.resize) {
        image = image.resize(
          operations.resize.width,
          operations.resize.height,
          {
            fit: "cover",
            position: "center",
          }
        );
      }

      await job.progress(40);

      // Apply format conversion
      if (operations.format) {
        image = image.toFormat(operations.format, {
          quality: operations.quality || 80,
        });
      }

      await job.progress(60);

      // Generate output path
      const ext = operations.format || path.extname(filePath).slice(1);
      const outputPath = filePath.replace(
        path.extname(filePath),
        `-processed.${ext}`
      );

      // Save processed image
      await image.toFile(outputPath);

      await job.progress(90);

      // Generate thumbnail if requested
      if (operations.thumbnail) {
        const thumbnailPath = outputPath.replace(
          `-processed.${ext}`,
          `-thumbnail.${ext}`
        );

        await sharp(outputPath)
          .resize(200, 200, { fit: "cover" })
          .toFile(thumbnailPath);
      }

      await job.progress(100);

      this.logger.log(`Image processed successfully: ${outputPath}`);

      return outputPath;
    } catch (error) {
      this.logger.error(`Image processing failed: ${error.message}`);
      throw error;
    }
  }
}
```

### 5. Scheduled/Recurring Jobs

```typescript
// scheduler/scheduler.service.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bull";
import { Queue } from "bull";

@Injectable()
export class SchedulerService implements OnModuleInit {
  constructor(@InjectQueue("scheduled") private scheduledQueue: Queue) {}

  async onModuleInit() {
    // Set up recurring jobs
    await this.setupRecurringJobs();
  }

  private async setupRecurringJobs() {
    // Daily database cleanup at 2 AM
    await this.scheduledQueue.add(
      "database-cleanup",
      {},
      {
        repeat: {
          cron: "0 2 * * *", // Every day at 2:00 AM
        },
        jobId: "database-cleanup", // Prevent duplicates
      }
    );

    // Hourly cache invalidation
    await this.scheduledQueue.add(
      "cache-invalidation",
      {},
      {
        repeat: {
          every: 3600000, // Every hour (milliseconds)
        },
        jobId: "cache-invalidation",
      }
    );

    // Weekly report generation (Monday at 9 AM)
    await this.scheduledQueue.add(
      "weekly-report",
      {},
      {
        repeat: {
          cron: "0 9 * * 1", // Every Monday at 9:00 AM
        },
        jobId: "weekly-report",
      }
    );
  }

  // Remove a recurring job
  async removeRecurringJob(jobId: string) {
    const job = await this.scheduledQueue.getJob(jobId);
    if (job) {
      await job.remove();
    }
  }

  // List all repeatable jobs
  async getRepeatableJobs() {
    return this.scheduledQueue.getRepeatableJobs();
  }
}
```

```typescript
// scheduler/scheduler.processor.ts
import { Process, Processor } from "@nestjs/bull";
import { Job } from "bull";
import { Logger } from "@nestjs/common";

@Processor("scheduled")
export class SchedulerProcessor {
  private readonly logger = new Logger(SchedulerProcessor.name);

  @Process("database-cleanup")
  async handleDatabaseCleanup(job: Job) {
    this.logger.log("Running database cleanup...");

    // Implement cleanup logic
    // - Delete old records
    // - Archive data
    // - Vacuum database

    return { cleaned: true, timestamp: new Date() };
  }

  @Process("cache-invalidation")
  async handleCacheInvalidation(job: Job) {
    this.logger.log("Running cache invalidation...");

    // Implement cache cleanup
    // - Clear expired cache entries
    // - Refresh hot cache

    return { invalidated: true };
  }

  @Process("weekly-report")
  async handleWeeklyReport(job: Job) {
    this.logger.log("Generating weekly report...");

    // Generate and send reports
    // - Aggregate data
    // - Generate PDF
    // - Send via email

    return { reportGenerated: true };
  }
}
```

### 6. Bull Board (Queue UI)

```typescript
// bull-board.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bull";
import { BullBoardModule } from "@bull-board/nestjs";
import { ExpressAdapter } from "@bull-board/express";
import { BullAdapter } from "@bull-board/api/bullAdapter";

@Module({
  imports: [
    BullBoardModule.forRoot({
      route: "/admin/queues",
      adapter: ExpressAdapter,
    }),

    BullBoardModule.forFeature({
      name: "email",
      adapter: BullAdapter,
    }),

    BullBoardModule.forFeature({
      name: "image",
      adapter: BullAdapter,
    }),

    BullBoardModule.forFeature({
      name: "scheduled",
      adapter: BullAdapter,
    }),
  ],
})
export class BullBoardConfigModule {}
```

### 7. Queue Controller (REST API)

```typescript
// queue.controller.ts
import { Controller, Get, Post, Param, Body, Query } from "@nestjs/common";
import { EmailService } from "./email/email.service";
import { ImageService } from "./image/image.service";

@Controller("queue")
export class QueueController {
  constructor(
    private emailService: EmailService,
    private imageService: ImageService
  ) {}

  @Post("email")
  async queueEmail(@Body() data: any) {
    const job = await this.emailService.sendEmail(data);
    return { jobId: job.id, message: "Email queued successfully" };
  }

  @Get("email/:jobId")
  async getEmailJobStatus(@Param("jobId") jobId: string) {
    return this.emailService.getJobStatus(jobId);
  }

  @Post("email/:jobId/retry")
  async retryEmailJob(@Param("jobId") jobId: string) {
    await this.emailService.retryJob(jobId);
    return { message: "Job retry initiated" };
  }

  @Get("stats")
  async getQueueStats() {
    return {
      email: await this.emailService.getQueueStats(),
      // Add other queue stats
    };
  }
}
```

## Best Practices

### 1. **Job Idempotency**

Ensure jobs can be safely retried without side effects:

```typescript
@Process('create-order')
async handleCreateOrder(job: Job<OrderData>) {
  const { orderId } = job.data;

  // Check if order already exists
  const existing = await this.orderRepository.findOne({ id: orderId });
  if (existing) {
    this.logger.warn(`Order ${orderId} already processed`);
    return existing;
  }

  // Create order
  return this.orderRepository.create(job.data);
}
```

### 2. **Proper Error Handling**

```typescript
@Process('risky-task')
async handleRiskyTask(job: Job) {
  try {
    // Attempt task
    await this.performTask(job.data);
  } catch (error) {
    if (error instanceof TemporaryError) {
      // Retry for temporary errors
      throw error;
    } else {
      // Don't retry for permanent errors
      this.logger.error(`Permanent error: ${error.message}`);
      // Move to dead letter queue or send alert
      return { failed: true, reason: error.message };
    }
  }
}
```

### 3. **Job Progress Tracking**

```typescript
@Process('long-task')
async handleLongTask(job: Job) {
  const totalSteps = 100;

  for (let i = 0; i < totalSteps; i++) {
    await this.processStep(i);

    // Update progress
    const progress = Math.floor((i / totalSteps) * 100);
    await job.progress(progress);
  }

  return { completed: true };
}
```

### 4. **Queue Priority**

```typescript
// High priority (1) - urgent
await queue.add("urgent-email", data, { priority: 1 });

// Medium priority (5) - default
await queue.add("normal-email", data, { priority: 5 });

// Low priority (10) - can wait
await queue.add("newsletter", data, { priority: 10 });
```

### 5. **Rate Limiting**

```typescript
// Limit API calls to external service
@Process({ name: 'api-call', concurrency: 1 })
async handleApiCall(job: Job) {
  // Only 1 job processed at a time
  await this.externalApi.call(job.data);

  // Add delay between jobs
  await new Promise(resolve => setTimeout(resolve, 1000));
}
```

### 6. **Job Cleanup**

```typescript
// Configure automatic cleanup
BullModule.registerQueue({
  name: "email",
  defaultJobOptions: {
    removeOnComplete: {
      age: 86400, // Keep for 24 hours
      count: 1000, // Keep last 1000
    },
    removeOnFail: {
      age: 604800, // Keep failed for 7 days
    },
  },
});
```

### 7. **Monitoring and Alerts**

```typescript
@OnQueueFailed()
async onFailed(job: Job, error: Error) {
  // Log to monitoring service
  this.logger.error(`Job ${job.id} failed: ${error.message}`);

  // Send alert if too many failures
  const failedCount = await this.queue.getFailedCount();
  if (failedCount > 100) {
    await this.alertService.sendAlert({
      type: 'QUEUE_FAILURES',
      message: `Too many failed jobs: ${failedCount}`,
    });
  }
}
```

### 8. **Graceful Shutdown**

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable graceful shutdown
  app.enableShutdownHooks();

  await app.listen(3000);
}
```

## Key Takeaways

✅ **Use queues for time-consuming tasks to improve response times**  
✅ **Implement retry logic with exponential backoff**  
✅ **Make jobs idempotent for safe retries**  
✅ **Use job priorities for important tasks**  
✅ **Monitor queue health and failed jobs**  
✅ **Configure automatic job cleanup to manage memory**  
✅ **Use Bull Board for queue visualization and debugging**  
✅ **Implement proper error handling and logging**  
✅ **Scale workers independently from web servers**  
✅ **Use scheduled jobs for recurring tasks**

Background jobs and queues are essential for building scalable, responsive applications that handle heavy workloads efficiently.
