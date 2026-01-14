# File Storage & Media Processing

## Overview

File storage and media processing are essential for modern applications. This tutorial covers uploading files to cloud storage (AWS S3, Cloudinary), processing images, handling videos, and implementing secure file management.

## Practical Use Cases

- **Profile pictures** and avatars
- **Product images** for e-commerce
- **Document uploads** (PDFs, spreadsheets)
- **Video hosting** for courses or content platforms
- **File sharing** and collaboration tools

## Storage Solutions Comparison

| Solution                 | Best For                   | Pricing             | Features                           |
| ------------------------ | -------------------------- | ------------------- | ---------------------------------- |
| **AWS S3**               | Any file type, large scale | Pay per GB          | Highly scalable, CDN integration   |
| **Cloudinary**           | Images/videos              | Free tier available | Auto-optimization, transformations |
| **Google Cloud Storage** | Enterprise apps            | Pay per GB          | Similar to S3                      |
| **Local Storage**        | Development only           | Free                | Not scalable                       |

## Step-by-Step Implementation

### 1. AWS S3 File Upload with NestJS

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner multer
npm install -D @types/multer
```

```typescript
// upload/upload.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { UploadController } from "./upload.controller";
import { UploadService } from "./upload.service";
import { S3Service } from "./s3.service";

@Module({
  imports: [ConfigModule],
  controllers: [UploadController],
  providers: [UploadService, S3Service],
  exports: [UploadService, S3Service],
})
export class UploadModule {}

// upload/s3.service.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { v4 as uuid } from "uuid";

@Injectable()
export class S3Service {
  private s3Client: S3Client;
  private bucketName: string;

  constructor(private configService: ConfigService) {
    this.s3Client = new S3Client({
      region: this.configService.get("AWS_REGION"),
      credentials: {
        accessKeyId: this.configService.get("AWS_ACCESS_KEY_ID"),
        secretAccessKey: this.configService.get("AWS_SECRET_ACCESS_KEY"),
      },
    });
    this.bucketName = this.configService.get("AWS_S3_BUCKET");
  }

  async uploadFile(
    file: Express.Multer.File,
    folder: string = "uploads"
  ): Promise<{ key: string; url: string }> {
    const fileExtension = file.originalname.split(".").pop();
    const key = `${folder}/${uuid()}.${fileExtension}`;

    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: "public-read", // or 'private' for secure files
    });

    await this.s3Client.send(command);

    const url = `https://${this.bucketName}.s3.${this.configService.get(
      "AWS_REGION"
    )}.amazonaws.com/${key}`;

    return { key, url };
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
    folder: string = "private"
  ): Promise<{ key: string; signedUrl: string }> {
    const fileExtension = file.originalname.split(".").pop();
    const key = `${folder}/${uuid()}.${fileExtension}`;

    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: "private",
    });

    await this.s3Client.send(command);

    const signedUrl = await this.getSignedUrl(key);

    return { key, signedUrl };
  }
}

// upload/upload.service.ts
import { Injectable, BadRequestException } from "@nestjs/common";
import { S3Service } from "./s3.service";
import * as sharp from "sharp";

@Injectable()
export class UploadService {
  constructor(private s3Service: S3Service) {}

  private readonly allowedImageTypes = [
    "image/jpeg",
    "image/png",
    "image/webp",
  ];
  private readonly allowedDocTypes = ["application/pdf", "application/msword"];
  private readonly maxFileSize = 5 * 1024 * 1024; // 5MB

  validateFile(file: Express.Multer.File, allowedTypes: string[]): void {
    if (!allowedTypes.includes(file.mimetype)) {
      throw new BadRequestException(
        `Invalid file type. Allowed types: ${allowedTypes.join(", ")}`
      );
    }

    if (file.size > this.maxFileSize) {
      throw new BadRequestException(
        `File too large. Maximum size: ${this.maxFileSize / 1024 / 1024}MB`
      );
    }
  }

  async uploadImage(
    file: Express.Multer.File
  ): Promise<{ url: string; key: string }> {
    this.validateFile(file, this.allowedImageTypes);

    // Optimize image
    const optimizedBuffer = await sharp(file.buffer)
      .resize(1920, 1920, { fit: "inside", withoutEnlargement: true })
      .jpeg({ quality: 85 })
      .toBuffer();

    const optimizedFile = {
      ...file,
      buffer: optimizedBuffer,
      mimetype: "image/jpeg",
    };

    return this.s3Service.uploadFile(optimizedFile, "images");
  }

  async uploadThumbnail(
    file: Express.Multer.File
  ): Promise<{ url: string; key: string }> {
    this.validateFile(file, this.allowedImageTypes);

    // Create thumbnail
    const thumbnailBuffer = await sharp(file.buffer)
      .resize(300, 300, { fit: "cover" })
      .jpeg({ quality: 80 })
      .toBuffer();

    const thumbnailFile = {
      ...file,
      buffer: thumbnailBuffer,
      mimetype: "image/jpeg",
    };

    return this.s3Service.uploadFile(thumbnailFile, "thumbnails");
  }

  async uploadDocument(
    file: Express.Multer.File
  ): Promise<{ url: string; key: string }> {
    this.validateFile(file, this.allowedDocTypes);
    return this.s3Service.uploadFile(file, "documents");
  }

  async uploadProfilePicture(
    file: Express.Multer.File,
    userId: string
  ): Promise<{ url: string; thumbnailUrl: string }> {
    // Upload full size
    const fullSize = await this.uploadImage(file);

    // Create and upload thumbnail
    const thumbnail = await this.uploadThumbnail(file);

    return {
      url: fullSize.url,
      thumbnailUrl: thumbnail.url,
    };
  }

  async deleteFile(key: string): Promise<void> {
    return this.s3Service.deleteFile(key);
  }
}

// upload/upload.controller.ts
import {
  Controller,
  Post,
  Delete,
  UseGuards,
  UseInterceptors,
  UploadedFile,
  UploadedFiles,
  Body,
  BadRequestException,
} from "@nestjs/common";
import {
  FileInterceptor,
  FilesInterceptor,
  FileFieldsInterceptor,
} from "@nestjs/platform-express";
import { ApiTags, ApiConsumes, ApiBody, ApiBearerAuth } from "@nestjs/swagger";
import { UploadService } from "./upload.service";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@ApiTags("upload")
@Controller("upload")
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class UploadController {
  constructor(private uploadService: UploadService) {}

  @Post("image")
  @UseInterceptors(FileInterceptor("file"))
  @ApiConsumes("multipart/form-data")
  @ApiBody({
    schema: {
      type: "object",
      properties: {
        file: {
          type: "string",
          format: "binary",
        },
      },
    },
  })
  async uploadImage(@UploadedFile() file: Express.Multer.File) {
    if (!file) {
      throw new BadRequestException("No file provided");
    }
    return this.uploadService.uploadImage(file);
  }

  @Post("profile-picture")
  @UseInterceptors(FileInterceptor("file"))
  @ApiConsumes("multipart/form-data")
  async uploadProfilePicture(
    @UploadedFile() file: Express.Multer.File,
    @CurrentUser("id") userId: string
  ) {
    if (!file) {
      throw new BadRequestException("No file provided");
    }
    return this.uploadService.uploadProfilePicture(file, userId);
  }

  @Post("document")
  @UseInterceptors(FileInterceptor("file"))
  @ApiConsumes("multipart/form-data")
  async uploadDocument(@UploadedFile() file: Express.Multer.File) {
    if (!file) {
      throw new BadRequestException("No file provided");
    }
    return this.uploadService.uploadDocument(file);
  }

  @Post("multiple")
  @UseInterceptors(FilesInterceptor("files", 10))
  @ApiConsumes("multipart/form-data")
  async uploadMultiple(@UploadedFiles() files: Express.Multer.File[]) {
    if (!files || files.length === 0) {
      throw new BadRequestException("No files provided");
    }

    const uploads = await Promise.all(
      files.map((file) => this.uploadService.uploadImage(file))
    );

    return { files: uploads };
  }

  @Post("mixed")
  @UseInterceptors(
    FileFieldsInterceptor([
      { name: "avatar", maxCount: 1 },
      { name: "documents", maxCount: 5 },
    ])
  )
  @ApiConsumes("multipart/form-data")
  async uploadMixed(
    @UploadedFiles()
    files: {
      avatar?: Express.Multer.File[];
      documents?: Express.Multer.File[];
    }
  ) {
    const result: any = {};

    if (files.avatar) {
      result.avatar = await this.uploadService.uploadImage(files.avatar[0]);
    }

    if (files.documents) {
      result.documents = await Promise.all(
        files.documents.map((file) => this.uploadService.uploadDocument(file))
      );
    }

    return result;
  }

  @Delete()
  async deleteFile(@Body("key") key: string) {
    await this.uploadService.deleteFile(key);
    return { message: "File deleted successfully" };
  }
}
```

### 2. Cloudinary Integration

```bash
npm install cloudinary
```

```typescript
// upload/cloudinary.service.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { v2 as cloudinary } from "cloudinary";
import * as streamifier from "streamifier";

@Injectable()
export class CloudinaryService {
  constructor(private configService: ConfigService) {
    cloudinary.config({
      cloud_name: this.configService.get("CLOUDINARY_CLOUD_NAME"),
      api_key: this.configService.get("CLOUDINARY_API_KEY"),
      api_secret: this.configService.get("CLOUDINARY_API_SECRET"),
    });
  }

  async uploadImage(
    file: Express.Multer.File,
    folder: string = "uploads"
  ): Promise<any> {
    return new Promise((resolve, reject) => {
      const uploadStream = cloudinary.uploader.upload_stream(
        {
          folder,
          resource_type: "auto",
          transformation: [
            { width: 1920, height: 1920, crop: "limit" },
            { quality: "auto" },
            { fetch_format: "auto" },
          ],
        },
        (error, result) => {
          if (error) return reject(error);
          resolve(result);
        }
      );

      streamifier.createReadStream(file.buffer).pipe(uploadStream);
    });
  }

  async uploadVideo(
    file: Express.Multer.File,
    folder: string = "videos"
  ): Promise<any> {
    return new Promise((resolve, reject) => {
      const uploadStream = cloudinary.uploader.upload_stream(
        {
          folder,
          resource_type: "video",
          eager: [
            { width: 720, height: 480, crop: "pad", format: "mp4" },
            { width: 1280, height: 720, crop: "pad", format: "mp4" },
          ],
        },
        (error, result) => {
          if (error) return reject(error);
          resolve(result);
        }
      );

      streamifier.createReadStream(file.buffer).pipe(uploadStream);
    });
  }

  async deleteFile(publicId: string): Promise<any> {
    return cloudinary.uploader.destroy(publicId);
  }

  getOptimizedUrl(publicId: string, transformations: any = {}): string {
    return cloudinary.url(publicId, {
      transformation: [
        { quality: "auto" },
        { fetch_format: "auto" },
        ...Object.entries(transformations).map(([key, value]) => ({
          [key]: value,
        })),
      ],
    });
  }

  // Generate responsive image URLs
  getResponsiveUrls(publicId: string): {
    thumbnail: string;
    small: string;
    medium: string;
    large: string;
  } {
    return {
      thumbnail: this.getOptimizedUrl(publicId, {
        width: 150,
        height: 150,
        crop: "thumb",
      }),
      small: this.getOptimizedUrl(publicId, { width: 480 }),
      medium: this.getOptimizedUrl(publicId, { width: 1024 }),
      large: this.getOptimizedUrl(publicId, { width: 1920 }),
    };
  }
}
```

### 3. Image Processing with Sharp

```bash
npm install sharp
```

```typescript
// upload/image-processor.service.ts
import { Injectable } from "@nestjs/common";
import * as sharp from "sharp";

export interface ImageProcessOptions {
  width?: number;
  height?: number;
  quality?: number;
  format?: "jpeg" | "png" | "webp";
  fit?: "cover" | "contain" | "fill" | "inside" | "outside";
}

@Injectable()
export class ImageProcessorService {
  async processImage(
    buffer: Buffer,
    options: ImageProcessOptions = {}
  ): Promise<Buffer> {
    const {
      width,
      height,
      quality = 85,
      format = "jpeg",
      fit = "inside",
    } = options;

    let processor = sharp(buffer);

    // Resize if dimensions provided
    if (width || height) {
      processor = processor.resize(width, height, {
        fit,
        withoutEnlargement: true,
      });
    }

    // Convert format and compress
    switch (format) {
      case "jpeg":
        processor = processor.jpeg({ quality, mozjpeg: true });
        break;
      case "png":
        processor = processor.png({ quality, compressionLevel: 9 });
        break;
      case "webp":
        processor = processor.webp({ quality });
        break;
    }

    return processor.toBuffer();
  }

  async createThumbnail(buffer: Buffer, size: number = 300): Promise<Buffer> {
    return sharp(buffer)
      .resize(size, size, { fit: "cover" })
      .jpeg({ quality: 80 })
      .toBuffer();
  }

  async createMultipleSizes(buffer: Buffer): Promise<{
    thumbnail: Buffer;
    small: Buffer;
    medium: Buffer;
    large: Buffer;
  }> {
    const [thumbnail, small, medium, large] = await Promise.all([
      this.processImage(buffer, { width: 150, height: 150, fit: "cover" }),
      this.processImage(buffer, { width: 480 }),
      this.processImage(buffer, { width: 1024 }),
      this.processImage(buffer, { width: 1920 }),
    ]);

    return { thumbnail, small, medium, large };
  }

  async getImageMetadata(buffer: Buffer) {
    const metadata = await sharp(buffer).metadata();
    return {
      width: metadata.width,
      height: metadata.height,
      format: metadata.format,
      size: metadata.size,
      hasAlpha: metadata.hasAlpha,
    };
  }

  async addWatermark(
    imageBuffer: Buffer,
    watermarkPath: string
  ): Promise<Buffer> {
    return sharp(imageBuffer)
      .composite([
        {
          input: watermarkPath,
          gravity: "southeast",
        },
      ])
      .toBuffer();
  }
}
```

### 4. Direct Browser Upload (Presigned URLs)

```typescript
// upload/presigned-upload.service.ts
import { Injectable } from "@nestjs/common";
import { S3Service } from "./s3.service";
import { v4 as uuid } from "uuid";

export interface PresignedUploadData {
  uploadUrl: string;
  key: string;
  fields: Record<string, string>;
}

@Injectable()
export class PresignedUploadService {
  constructor(private s3Service: S3Service) {}

  async generatePresignedPost(
    folder: string = "uploads",
    allowedTypes: string[] = ["image/jpeg", "image/png"],
    maxSize: number = 5 * 1024 * 1024 // 5MB
  ): Promise<PresignedUploadData> {
    const key = `${folder}/${uuid()}`;

    // Note: You'll need to implement createPresignedPost with AWS SDK
    // This is a simplified example
    return {
      uploadUrl: `https://your-bucket.s3.amazonaws.com`,
      key,
      fields: {
        key,
        "Content-Type": allowedTypes[0],
        "x-amz-algorithm": "AWS4-HMAC-SHA256",
        // ... other required fields
      },
    };
  }
}

// Client-side usage (React):
/*
async function uploadWithPresignedUrl(file: File) {
  // 1. Get presigned URL from backend
  const { uploadUrl, key, fields } = await fetch('/api/upload/presigned-url')
    .then(res => res.json());

  // 2. Upload directly to S3
  const formData = new FormData();
  Object.entries(fields).forEach(([key, value]) => {
    formData.append(key, value);
  });
  formData.append('file', file);

  await fetch(uploadUrl, {
    method: 'POST',
    body: formData,
  });

  // 3. File is now uploaded, use the key to reference it
  return key;
}
*/
```

### 5. Video Processing (Advanced)

```bash
npm install fluent-ffmpeg
npm install -D @types/fluent-ffmpeg
```

```typescript
// upload/video-processor.service.ts
import { Injectable } from "@nestjs/common";
import * as ffmpeg from "fluent-ffmpeg";
import { promisify } from "util";
import * as fs from "fs";

@Injectable()
export class VideoProcessorService {
  async generateThumbnail(
    videoPath: string,
    outputPath: string,
    timeInSeconds: number = 1
  ): Promise<string> {
    return new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .screenshots({
          timestamps: [timeInSeconds],
          filename: "thumbnail.jpg",
          folder: outputPath,
          size: "320x240",
        })
        .on("end", () => resolve(`${outputPath}/thumbnail.jpg`))
        .on("error", reject);
    });
  }

  async compressVideo(inputPath: string, outputPath: string): Promise<string> {
    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .outputOptions(["-c:v libx264", "-crf 28", "-c:a aac", "-b:a 128k"])
        .output(outputPath)
        .on("end", () => resolve(outputPath))
        .on("error", reject)
        .run();
    });
  }

  async getVideoMetadata(videoPath: string): Promise<any> {
    return new Promise((resolve, reject) => {
      ffmpeg.ffprobe(videoPath, (err, metadata) => {
        if (err) return reject(err);
        resolve({
          duration: metadata.format.duration,
          size: metadata.format.size,
          bitRate: metadata.format.bit_rate,
          format: metadata.format.format_name,
          streams: metadata.streams,
        });
      });
    });
  }
}
```

## Best Practices

### 1. Security

```typescript
// Validate file types properly
const ALLOWED_MIME_TYPES = {
  images: ["image/jpeg", "image/png", "image/webp", "image/gif"],
  documents: ["application/pdf", "application/msword"],
  videos: ["video/mp4", "video/quicktime"],
};

// Check file extension AND mime type
function isValidFile(file: Express.Multer.File, category: string): boolean {
  return ALLOWED_MIME_TYPES[category].includes(file.mimetype);
}

// Sanitize filenames
function sanitizeFilename(filename: string): string {
  return filename.replace(/[^a-zA-Z0-9.-]/g, "_");
}

// Scan for malware (use ClamAV or similar)
async function scanFile(buffer: Buffer): Promise<boolean> {
  // Implement virus scanning
  return true;
}
```

### 2. Optimization

```typescript
// Always optimize images before upload
async function optimizeBeforeUpload(file: Express.Multer.File) {
  const optimized = await sharp(file.buffer)
    .resize(1920, 1920, { fit: "inside", withoutEnlargement: true })
    .jpeg({ quality: 85, mozjpeg: true })
    .toBuffer();

  return optimized;
}

// Use CDN for serving files
const cdnUrl = "https://cdn.example.com";
const fileUrl = `${cdnUrl}/${key}`;

// Implement lazy loading on frontend
<img loading="lazy" src={fileUrl} alt="..." />;
```

### 3. Error Handling

```typescript
try {
  const result = await s3Service.uploadFile(file);
  return result;
} catch (error) {
  if (error.code === "NoSuchBucket") {
    throw new InternalServerErrorException("Storage bucket not configured");
  }
  throw new BadRequestException("Upload failed");
}
```

## Key Takeaways

1. **Use cloud storage** (S3, Cloudinary) for scalability
2. **Validate files** - check type, size, and scan for malware
3. **Optimize images** before uploading with Sharp
4. **Use CDN** for serving files efficiently
5. **Generate thumbnails** for faster loading
6. **Presigned URLs** for direct browser uploads
7. **Process async** - use queues for heavy processing
8. **Clean up** - delete old/unused files regularly

Proper file handling is crucial for user experience and security. Always validate, optimize, and use appropriate storage solutions for your needs.
