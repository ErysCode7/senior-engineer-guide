# File Storage & Media Processing

## Overview

File storage and media processing are essential for modern applications. This guide covers uploading files with Express using Multer, storing them in AWS S3 and Cloudinary, processing images with Sharp, and implementing secure file management patterns.

## Storage Solutions Comparison

| Solution                 | Best For                   | Pricing             | Features                           |
| ------------------------ | -------------------------- | ------------------- | ---------------------------------- |
| **AWS S3**               | Any file type, large scale | Pay per GB          | Highly scalable, CDN integration   |
| **Cloudinary**           | Images/videos              | Free tier available | Auto-optimization, transformations |
| **Google Cloud Storage** | Enterprise apps            | Pay per GB          | Similar to S3                      |
| **Local Storage**        | Development only           | Free                | Not scalable                       |

## File Upload with Multer

### 1. Basic Setup

```bash
npm install multer @aws-sdk/client-s3 sharp
npm install -D @types/multer
```

```typescript
// src/config/multer.ts
import multer from 'multer';
import path from 'path';
import { AppError } from '../utils/AppError';

const storage = multer.memoryStorage();

const fileFilter = (req: any, file: Express.Multer.File, cb: any) => {
  const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
  
  if (allowedMimeTypes.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new AppError('Invalid file type. Only JPEG, PNG, WebP, and GIF allowed', 400), false);
  }
};

export const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
  },
});
```

```typescript
// src/services/upload.service.ts
import sharp from 'sharp';
import { v4 as uuid } from 'uuid';
import { AppError } from '../utils/AppError';

export class UploadService {
  private readonly MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

  validateFile(file: Express.Multer.File, allowedTypes: string[]): void {
    if (!allowedTypes.includes(file.mimetype)) {
      throw new AppError(`Invalid file type. Allowed: ${allowedTypes.join(', ')}`, 400);
    }

    if (file.size > this.MAX_FILE_SIZE) {
      throw new AppError(`File too large. Max size: ${this.MAX_FILE_SIZE / 1024 / 1024}MB`, 400);
    }
  }

  async optimizeImage(buffer: Buffer): Promise<Buffer> {
    return sharp(buffer)
      .resize(1920, 1920, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 85 })
      .toBuffer();
  }

  async createThumbnail(buffer: Buffer, width = 300, height = 300): Promise<Buffer> {
    return sharp(buffer)
      .resize(width, height, { fit: 'cover' })
      .jpeg({ quality: 80 })
      .toBuffer();
  }

  generateFileName(originalName: string): string {
    const ext = originalName.split('.').pop();
    return `${uuid()}.${ext}`;
  }
}

export default new UploadService();
```

## AWS S3 Integration

### 1. S3 Service

```typescript
// src/services/s3.service.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import uploadService from './upload.service';

export class S3Service {
  private s3Client: S3Client;
  private bucketName: string;
  private region: string;

  constructor() {
    this.region = process.env.AWS_REGION || 'us-east-1';
    this.bucketName = process.env.AWS_S3_BUCKET || '';
    
    this.s3Client = new S3Client({
      region: this.region,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || '',
      },
    });
  }

  async uploadFile(
    file: Express.Multer.File,
    folder: string = 'uploads'
  ): Promise<{ key: string; url: string }> {
    const key = `${folder}/${uploadService.generateFileName(file.originalname)}`;

    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: 'public-read',
    });

    await this.s3Client.send(command);

    const url = `https://${this.bucketName}.s3.${this.region}.amazonaws.com/${key}`;

    return { key, url };
  }

  async uploadOptimizedImage(
    file: Express.Multer.File,
    folder: string = 'images'
  ): Promise<{ key: string; url: string; thumbnailUrl: string }> {
    const [optimizedBuffer, thumbnailBuffer] = await Promise.all([
      uploadService.optimizeImage(file.buffer),
      uploadService.createThumbnail(file.buffer),
    ]);

    const fileName = uploadService.generateFileName(file.originalname);
    const key = `${folder}/${fileName}`;
    const thumbnailKey = `${folder}/thumbnails/${fileName}`;

    await Promise.all([
      this.s3Client.send(
        new PutObjectCommand({
          Bucket: this.bucketName,
          Key: key,
          Body: optimizedBuffer,
          ContentType: 'image/jpeg',
          ACL: 'public-read',
        })
      ),
      this.s3Client.send(
        new PutObjectCommand({
          Bucket: this.bucketName,
          Key: thumbnailKey,
          Body: thumbnailBuffer,
          ContentType: 'image/jpeg',
          ACL: 'public-read',
        })
      ),
    ]);

    const baseUrl = `https://${this.bucketName}.s3.${this.region}.amazonaws.com`;

    return {
      key,
      url: `${baseUrl}/${key}`,
      thumbnailUrl: `${baseUrl}/${thumbnailKey}`,
    };
  }

  async deleteFile(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    await this.s3Client.send(command);
  }

  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    return getSignedUrl(this.s3Client, command, { expiresIn });
  }

  async uploadPrivateFile(
    file: Express.Multer.File,
    folder: string = 'private'
  ): Promise<{ key: string; signedUrl: string }> {
    const key = `${folder}/${uploadService.generateFileName(file.originalname)}`;

    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.bucketName,
        Key: key,
        Body: file.buffer,
        ContentType: file.mimetype,
        ACL: 'private',
      })
    );

    const signedUrl = await this.getSignedUrl(key);

    return { key, signedUrl };
  }
}

export default new S3Service();
```

### 2. Upload Controller

```typescript
// src/controllers/upload.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import s3Service from '../services/s3.service';
import uploadService from '../services/upload.service';

export class UploadController {
  uploadSingle = asyncHandler(async (req: Request, res: Response) => {
    if (!req.file) {
      throw new AppError('No file uploaded', 400);
    }

    const result = await s3Service.uploadFile(req.file, 'uploads');

    res.json({
      success: true,
      data: result,
    });
  });

  uploadImage = asyncHandler(async (req: Request, res: Response) => {
    if (!req.file) {
      throw new AppError('No file uploaded', 400);
    }

    uploadService.validateFile(req.file, ['image/jpeg', 'image/png', 'image/webp']);

    const result = await s3Service.uploadOptimizedImage(req.file, 'images');

    res.json({
      success: true,
      data: result,
    });
  });

  uploadMultiple = asyncHandler(async (req: Request, res: Response) => {
    if (!req.files || !Array.isArray(req.files)) {
      throw new AppError('No files uploaded', 400);
    }

    const uploadPromises = req.files.map((file) => 
      s3Service.uploadFile(file, 'uploads')
    );

    const results = await Promise.all(uploadPromises);

    res.json({
      success: true,
      data: results,
    });
  });

  deleteFile = asyncHandler(async (req: Request, res: Response) => {
    const { key } = req.params;

    await s3Service.deleteFile(key);

    res.json({
      success: true,
      message: 'File deleted successfully',
    });
  });
}

export default new UploadController();
```

### 3. Upload Routes

```typescript
// src/routes/upload.routes.ts
import { Router } from 'express';
import uploadController from '../controllers/upload.controller';
import { upload } from '../config/multer';
import { authenticate } from '../middleware/auth.middleware';

const router = Router();

router.use(authenticate);

router.post('/single', upload.single('file'), uploadController.uploadSingle);
router.post('/image', upload.single('image'), uploadController.uploadImage);
router.post('/multiple', upload.array('files', 10), uploadController.uploadMultiple);
router.delete('/:key', uploadController.deleteFile);

export default router;
```

## Cloudinary Integration

### 1. Setup

```bash
npm install cloudinary
```

```typescript
// src/config/cloudinary.ts
import { v2 as cloudinary } from 'cloudinary';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

export default cloudinary;
```

```typescript
// src/services/cloudinary.service.ts
import cloudinary from '../config/cloudinary';
import { AppError } from '../utils/AppError';

export class CloudinaryService {
  async uploadImage(
    file: Express.Multer.File,
    folder: string = 'uploads'
  ): Promise<{ url: string; publicId: string; thumbnailUrl: string }> {
    return new Promise((resolve, reject) => {
      const uploadStream = cloudinary.uploader.upload_stream(
        {
          folder,
          resource_type: 'auto',
          transformation: [
            { width: 1920, height: 1920, crop: 'limit' },
            { quality: 'auto:good' },
          ],
        },
        (error, result) => {
          if (error) {
            reject(new AppError('Failed to upload to Cloudinary', 500));
          } else {
            resolve({
              url: result!.secure_url,
              publicId: result!.public_id,
              thumbnailUrl: cloudinary.url(result!.public_id, {
                width: 300,
                height: 300,
                crop: 'fill',
              }),
            });
          }
        }
      );

      uploadStream.end(file.buffer);
    });
  }

  async deleteImage(publicId: string): Promise<void> {
    await cloudinary.uploader.destroy(publicId);
  }

  getTransformedUrl(publicId: string, options: any): string {
    return cloudinary.url(publicId, options);
  }
}

export default new CloudinaryService();
```

## Image Processing with Sharp

```typescript
// src/services/image.service.ts
import sharp from 'sharp';

export class ImageService {
  async resize(buffer: Buffer, width: number, height: number): Promise<Buffer> {
    return sharp(buffer)
      .resize(width, height, { fit: 'cover' })
      .toBuffer();
  }

  async crop(buffer: Buffer, x: number, y: number, width: number, height: number): Promise<Buffer> {
    return sharp(buffer)
      .extract({ left: x, top: y, width, height })
      .toBuffer();
  }

  async rotate(buffer: Buffer, angle: number): Promise<Buffer> {
    return sharp(buffer)
      .rotate(angle)
      .toBuffer();
  }

  async addWatermark(imageBuffer: Buffer, watermarkBuffer: Buffer): Promise<Buffer> {
    return sharp(imageBuffer)
      .composite([{ input: watermarkBuffer, gravity: 'southeast' }])
      .toBuffer();
  }

  async convertFormat(buffer: Buffer, format: 'jpeg' | 'png' | 'webp'): Promise<Buffer> {
    return sharp(buffer)
      .toFormat(format)
      .toBuffer();
  }

  async getMetadata(buffer: Buffer) {
    return sharp(buffer).metadata();
  }
}

export default new ImageService();
```

## Secure File Handling

```typescript
// src/middleware/file-validation.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/AppError';

export const validateImageUpload = (req: Request, res: Response, next: NextFunction) => {
  if (!req.file) {
    throw new AppError('No file provided', 400);
  }

  const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
  
  if (!allowedMimeTypes.includes(req.file.mimetype)) {
    throw new AppError('Invalid file type', 400);
  }

  const maxSize = 5 * 1024 * 1024; // 5MB
  if (req.file.size > maxSize) {
    throw new AppError('File too large', 400);
  }

  next();
};

export const sanitizeFileName = (fileName: string): string => {
  return fileName
    .toLowerCase()
    .replace(/[^a-z0-9.-]/g, '-')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '');
};
```

## Direct Upload (Frontend → S3)

```typescript
// src/controllers/presigned-url.controller.ts
import { Request, Response } from 'express';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { asyncHandler } from '../utils/asyncHandler';
import { v4 as uuid } from 'uuid';

export class PresignedUrlController {
  getUploadUrl = asyncHandler(async (req: Request, res: Response) => {
    const { fileName, fileType } = req.body;

    const s3Client = new S3Client({
      region: process.env.AWS_REGION,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
      },
    });

    const key = `uploads/${uuid()}-${fileName}`;

    const command = new PutObjectCommand({
      Bucket: process.env.AWS_S3_BUCKET,
      Key: key,
      ContentType: fileType,
    });

    const uploadUrl = await getSignedUrl(s3Client, command, { expiresIn: 3600 });

    res.json({
      success: true,
      data: {
        uploadUrl,
        key,
        fileUrl: `https://${process.env.AWS_S3_BUCKET}.s3.amazonaws.com/${key}`,
      },
    });
  });
}

export default new PresignedUrlController();
```

## Testing

```typescript
// src/__tests__/upload.test.ts
import request from 'supertest';
import path from 'path';
import app from '../server';

describe('File Upload', () => {
  let authToken: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = res.body.data.accessToken;
  });

  describe('POST /api/v1/upload/image', () => {
    it('should upload an image', async () => {
      const res = await request(app)
        .post('/api/v1/upload/image')
        .set('Authorization', `Bearer ${authToken}`)
        .attach('image', path.join(__dirname, 'fixtures/test-image.jpg'));

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('url');
      expect(res.body.data).toHaveProperty('thumbnailUrl');
    });

    it('should reject invalid file types', async () => {
      const res = await request(app)
        .post('/api/v1/upload/image')
        .set('Authorization', `Bearer ${authToken}`)
        .attach('image', path.join(__dirname, 'fixtures/test.txt'));

      expect(res.status).toBe(400);
    });

    it('should reject files that are too large', async () => {
      const res = await request(app)
        .post('/api/v1/upload/image')
        .set('Authorization', `Bearer ${authToken}`)
        .attach('image', path.join(__dirname, 'fixtures/large-file.jpg'));

      expect(res.status).toBe(400);
    });
  });

  describe('POST /api/v1/upload/multiple', () => {
    it('should upload multiple files', async () => {
      const res = await request(app)
        .post('/api/v1/upload/multiple')
        .set('Authorization', `Bearer ${authToken}`)
        .attach('files', path.join(__dirname, 'fixtures/test1.jpg'))
        .attach('files', path.join(__dirname, 'fixtures/test2.jpg'));

      expect(res.status).toBe(200);
      expect(Array.isArray(res.body.data)).toBe(true);
      expect(res.body.data).toHaveLength(2);
    });
  });
});
```

## Best Practices

1. **Validate file types** on both client and server
2. **Limit file sizes** to prevent abuse
3. **Use unique file names** to avoid conflicts
4. **Store metadata** in database
5. **Optimize images** before storage
6. **Use CDN** for faster delivery
7. **Implement virus scanning** for production
8. **Clean up failed uploads**
9. **Use signed URLs** for private files
10. **Monitor storage costs**

## Key Takeaways

1. **Multer** handles file uploads in Express
2. **S3** is scalable for any file type
3. **Cloudinary** excels at media optimization
4. **Sharp** for server-side image processing
5. **Presigned URLs** enable direct uploads
6. **Validation** is critical for security
7. **Thumbnails** improve performance
8. **CDN integration** reduces latency

Choose storage based on your needs: S3 for general files, Cloudinary for media-heavy applications.
