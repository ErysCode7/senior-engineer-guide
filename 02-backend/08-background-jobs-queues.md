# Background Jobs & Queues

## Overview

Background jobs allow you to offload time-consuming tasks from HTTP requests, improving response times and user experience. This guide covers Bull queues with Redis for job processing, scheduling, and monitoring in Express applications.

## Setup

```bash
npm install bull
npm install -D @types/bull
```

```typescript
// src/config/redis.ts
import Redis from 'ioredis';

export const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null,
});

export default redis;
```

## Basic Queue

```typescript
// src/queues/email.queue.ts
import Queue from 'bull';
import redis from '../config/redis';
import emailService from '../services/email.service';

export const emailQueue = new Queue('email', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

// Define job processor
emailQueue.process(async (job) => {
  const { to, subject, html } = job.data;
  
  await emailService.sendEmail({ to, subject, html });
  
  return { sent: true, messageId: 'xyz' };
});

// Error handling
emailQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err);
});

emailQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

// Add job to queue
export const addEmailJob = async (to: string, subject: string, html: string) => {
  return emailQueue.add(
    { to, subject, html },
    {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 2000,
      },
      removeOnComplete: true,
      removeOnFail: false,
    }
  );
};
```

## Queue Service

```typescript
// src/services/queue.service.ts
import Queue, { Job, JobOptions } from 'bull';

export class QueueService {
  private queues: Map<string, Queue.Queue> = new Map();

  createQueue(name: string): Queue.Queue {
    if (this.queues.has(name)) {
      return this.queues.get(name)!;
    }

    const queue = new Queue(name, {
      redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379'),
      },
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000,
        },
        removeOnComplete: 100,
        removeOnFail: false,
      },
    });

    this.queues.set(name, queue);
    return queue;
  }

  getQueue(name: string): Queue.Queue | undefined {
    return this.queues.get(name);
  }

  async addJob(
    queueName: string,
    data: any,
    options?: JobOptions
  ): Promise<Job> {
    const queue = this.getQueue(queueName);
    
    if (!queue) {
      throw new Error(`Queue ${queueName} not found`);
    }

    return queue.add(data, options);
  }

  async getJobStatus(queueName: string, jobId: string) {
    const queue = this.getQueue(queueName);
    
    if (!queue) {
      throw new Error(`Queue ${queueName} not found`);
    }

    const job = await queue.getJob(jobId);
    
    if (!job) {
      return null;
    }

    const state = await job.getState();
    
    return {
      id: job.id,
      data: job.data,
      progress: job.progress(),
      state,
      failedReason: job.failedReason,
      finishedOn: job.finishedOn,
    };
  }

  async closeAll() {
    await Promise.all(
      Array.from(this.queues.values()).map((queue) => queue.close())
    );
  }
}

export default new QueueService();
```

## Job Types

```typescript
// src/queues/image-processing.queue.ts
import Queue from 'bull';
import sharp from 'sharp';
import s3Service from '../services/s3.service';

export const imageQueue = new Queue('image-processing', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

imageQueue.process('resize', async (job) => {
  const { imageUrl, width, height } = job.data;
  
  // Download image
  const response = await fetch(imageUrl);
  const buffer = Buffer.from(await response.arrayBuffer());
  
  // Process image
  const resized = await sharp(buffer)
    .resize(width, height)
    .jpeg({ quality: 85 })
    .toBuffer();
  
  // Upload to S3
  const result = await s3Service.uploadFile({
    buffer: resized,
    mimetype: 'image/jpeg',
    originalname: `resized-${width}x${height}.jpg`,
  } as any, 'processed');
  
  job.progress(100);
  
  return result;
});

imageQueue.process('thumbnail', async (job) => {
  const { imageUrl } = job.data;
  
  const response = await fetch(imageUrl);
  const buffer = Buffer.from(await response.arrayBuffer());
  
  const thumbnail = await sharp(buffer)
    .resize(300, 300, { fit: 'cover' })
    .jpeg({ quality: 80 })
    .toBuffer();
  
  const result = await s3Service.uploadFile({
    buffer: thumbnail,
    mimetype: 'image/jpeg',
    originalname: 'thumbnail.jpg',
  } as any, 'thumbnails');
  
  return result;
});

export const processImage = async (imageUrl: string, width: number, height: number) => {
  return imageQueue.add('resize', { imageUrl, width, height });
};

export const createThumbnail = async (imageUrl: string) => {
  return imageQueue.add('thumbnail', { imageUrl });
};
```

## Scheduled Jobs

```typescript
// src/queues/scheduled.queue.ts
import Queue from 'bull';
import { prisma } from '../config/database';

export const scheduledQueue = new Queue('scheduled', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

// Daily report job
scheduledQueue.process('daily-report', async () => {
  const report = await generateDailyReport();
  await sendReportEmail(report);
});

// Clean up old data
scheduledQueue.process('cleanup', async () => {
  await prisma.log.deleteMany({
    where: {
      createdAt: {
        lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000), // 30 days
      },
    },
  });
});

// Schedule repeating jobs
export const scheduleRepeatingJobs = () => {
  // Daily report at 9 AM
  scheduledQueue.add(
    'daily-report',
    {},
    {
      repeat: {
        cron: '0 9 * * *',
      },
    }
  );

  // Cleanup every Sunday at midnight
  scheduledQueue.add(
    'cleanup',
    {},
    {
      repeat: {
        cron: '0 0 * * 0',
      },
    }
  );
};
```

## Priority Queues

```typescript
// src/queues/notification.queue.ts
import Queue from 'bull';
import notificationService from '../services/notification.service';

export const notificationQueue = new Queue('notifications', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

notificationQueue.process(async (job) => {
  const { userId, type, data } = job.data;
  
  await notificationService.sendNotification({
    userId,
    channel: 'email',
    type,
    data,
  });
});

export const sendUrgentNotification = async (userId: string, data: any) => {
  return notificationQueue.add(
    { userId, type: 'urgent', data },
    { priority: 1 } // High priority
  );
};

export const sendNormalNotification = async (userId: string, data: any) => {
  return notificationQueue.add(
    { userId, type: 'normal', data },
    { priority: 5 } // Normal priority
  );
};
```

## Job Dependencies

```typescript
// src/queues/video-processing.queue.ts
import Queue from 'bull';

export const videoQueue = new Queue('video-processing', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

export const processVideo = async (videoUrl: string) => {
  // Step 1: Download video
  const downloadJob = await videoQueue.add('download', { videoUrl });
  
  // Step 2: Transcode (waits for download)
  const transcodeJob = await videoQueue.add(
    'transcode',
    { videoPath: 'pending' },
    {
      jobId: `transcode-${downloadJob.id}`,
    }
  );
  
  // Step 3: Upload to CDN (waits for transcode)
  const uploadJob = await videoQueue.add(
    'upload',
    { videoPath: 'pending' },
    {
      jobId: `upload-${transcodeJob.id}`,
    }
  );
  
  return uploadJob;
};

videoQueue.process('download', async (job) => {
  // Download logic
  return { path: '/tmp/video.mp4' };
});

videoQueue.process('transcode', async (job) => {
  // Transcode logic
  return { path: '/tmp/video-transcoded.mp4' };
});

videoQueue.process('upload', async (job) => {
  // Upload to CDN
  return { url: 'https://cdn.example.com/video.mp4' };
});
```

## Bull Board (Dashboard)

```bash
npm install @bull-board/express @bull-board/api
```

```typescript
// src/config/bull-board.ts
import { createBullBoard } from '@bull-board/api';
import { BullAdapter } from '@bull-board/api/bullAdapter';
import { ExpressAdapter } from '@bull-board/express';
import { emailQueue } from '../queues/email.queue';
import { imageQueue } from '../queues/image-processing.queue';

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
  queues: [
    new BullAdapter(emailQueue),
    new BullAdapter(imageQueue),
  ],
  serverAdapter,
});

export default serverAdapter;
```

```typescript
// src/server.ts - Add to routes
import bullBoard from './config/bull-board';

app.use('/admin/queues', bullBoard.getRouter());
```

## Controller

```typescript
// src/controllers/job.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import { addEmailJob } from '../queues/email.queue';
import { processImage } from '../queues/image-processing.queue';
import queueService from '../services/queue.service';

export class JobController {
  queueEmail = asyncHandler(async (req: Request, res: Response) => {
    const { to, subject, html } = req.body;
    
    const job = await addEmailJob(to, subject, html);
    
    res.json({
      success: true,
      data: {
        jobId: job.id,
      },
    });
  });

  processImage = asyncHandler(async (req: Request, res: Response) => {
    const { imageUrl, width, height } = req.body;
    
    const job = await processImage(imageUrl, width, height);
    
    res.json({
      success: true,
      data: {
        jobId: job.id,
      },
    });
  });

  getJobStatus = asyncHandler(async (req: Request, res: Response) => {
    const { queueName, jobId } = req.params;
    
    const status = await queueService.getJobStatus(queueName, jobId);
    
    if (!status) {
      return res.status(404).json({
        success: false,
        message: 'Job not found',
      });
    }
    
    res.json({
      success: true,
      data: status,
    });
  });
}

export default new JobController();
```

## Routes

```typescript
// src/routes/job.routes.ts
import { Router } from 'express';
import jobController from '../controllers/job.controller';
import { authenticate } from '../middleware/auth.middleware';

const router = Router();

router.use(authenticate);

router.post('/email', jobController.queueEmail);
router.post('/process-image', jobController.processImage);
router.get('/:queueName/:jobId', jobController.getJobStatus);

export default router;
```

## Testing

```typescript
// src/__tests__/queue.test.ts
import { emailQueue, addEmailJob } from '../queues/email.queue';

describe('Email Queue', () => {
  beforeEach(async () => {
    await emailQueue.empty();
  });

  afterAll(async () => {
    await emailQueue.close();
  });

  it('should add a job to the queue', async () => {
    const job = await addEmailJob(
      'test@example.com',
      'Test Subject',
      '<p>Test</p>'
    );

    expect(job.id).toBeDefined();
    expect(job.data.to).toBe('test@example.com');
  });

  it('should process a job successfully', async () => {
    const job = await addEmailJob(
      'test@example.com',
      'Test',
      '<p>Test</p>'
    );

    await job.finished();

    const state = await job.getState();
    expect(state).toBe('completed');
  });

  it('should retry failed jobs', async () => {
    // Mock failure
    const originalProcess = emailQueue.process;
    
    let attempts = 0;
    emailQueue.process(async () => {
      attempts++;
      if (attempts < 3) {
        throw new Error('Simulated failure');
      }
      return { sent: true };
    });

    const job = await addEmailJob('test@example.com', 'Test', '<p>Test</p>');
    await job.finished();

    expect(attempts).toBe(3);
  });
});
```

## Best Practices

1. **Set appropriate retry policies** based on job type
2. **Monitor queue metrics** (length, processing time)
3. **Use job priorities** for critical tasks
4. **Implement idempotency** to handle retries safely
5. **Clean up completed jobs** to save memory
6. **Use rate limiting** for external API calls
7. **Log job failures** for debugging
8. **Scale workers** based on queue depth
9. **Handle graceful shutdown** of workers
10. **Use separate queues** for different job types

## Production Considerations

```typescript
// src/workers/email-worker.ts
import { emailQueue } from '../queues/email.queue';

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing queue...');
  await emailQueue.close();
  process.exit(0);
});

// Error handling
emailQueue.on('error', (error) => {
  console.error('Queue error:', error);
  // Send to monitoring service
});

// Stalled job handling
emailQueue.on('stalled', (job) => {
  console.warn(`Job ${job.id} stalled`);
});
```

## Monitoring

```typescript
// src/services/queue-monitoring.service.ts
import { emailQueue } from '../queues/email.queue';

export class QueueMonitoringService {
  async getQueueStats() {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      emailQueue.getWaitingCount(),
      emailQueue.getActiveCount(),
      emailQueue.getCompletedCount(),
      emailQueue.getFailedCount(),
      emailQueue.getDelayedCount(),
    ]);

    return {
      waiting,
      active,
      completed,
      failed,
      delayed,
    };
  }

  async getFailedJobs() {
    return emailQueue.getFailed();
  }

  async retryFailedJob(jobId: string) {
    const job = await emailQueue.getJob(jobId);
    if (job) {
      await job.retry();
    }
  }
}

export default new QueueMonitoringService();
```

## Key Takeaways

1. **Bull** provides reliable job queuing with Redis
2. **Background jobs** improve API response times
3. **Retry logic** handles transient failures
4. **Scheduling** enables recurring tasks
5. **Priorities** ensure critical jobs run first
6. **Bull Board** offers visual queue management
7. **Monitoring** prevents queue overflow
8. **Graceful shutdown** prevents job loss

Background jobs are essential for scalable, responsive applications.
