# Input Validation & Sanitization

## Overview

Input validation and sanitization are critical security practices that protect applications from malicious input and injection attacks. Validation ensures data meets expected criteria, while sanitization removes or encodes potentially dangerous characters. Together, they form the first line of defense against vulnerabilities like SQL injection, XSS, and command injection.

**Key Concepts:**

- **Validation**: Checking input against expected format and rules
- **Sanitization**: Cleaning input to remove harmful content
- **Whitelisting**: Accepting only known-good input
- **Blacklisting**: Rejecting known-bad input (less secure)
- **Escaping**: Converting special characters to safe representations

## Practical Use Cases

### 1. **User Registration & Authentication**

Validate user credentials

- Email format validation
- Password strength requirements
- Username character restrictions
- Phone number formats

### 2. **Form Submissions**

Validate user input

- Contact forms
- Survey responses
- Comment sections
- Search queries

### 3. **API Endpoints**

Validate request data

- Request body validation
- Query parameter validation
- Path parameter validation
- Header validation

### 4. **File Uploads**

Validate uploaded files

- File type verification
- File size limits
- Filename sanitization
- Content validation

### 5. **Database Operations**

Prevent injection attacks

- SQL injection prevention
- NoSQL injection prevention
- Command injection prevention
- LDAP injection prevention

## NestJS Validation with class-validator

### 1. Basic DTO Validation

```typescript
// src/dtos/create-user.dto.ts
import {
  IsEmail,
  IsString,
  IsStrongPassword,
  MinLength,
  MaxLength,
  Matches,
  IsOptional,
  IsUrl,
  IsInt,
  Min,
  Max,
  IsEnum,
  ValidateNested,
  IsArray,
  ArrayMinSize,
  ArrayMaxSize,
} from "class-validator";
import { Transform, Type } from "class-transformer";

export enum UserRole {
  USER = "user",
  ADMIN = "admin",
  MODERATOR = "moderator",
}

export class CreateUserDto {
  @IsEmail({}, { message: "Please provide a valid email address" })
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsStrongPassword(
    {
      minLength: 8,
      minLowercase: 1,
      minUppercase: 1,
      minNumbers: 1,
      minSymbols: 1,
    },
    {
      message:
        "Password must be at least 8 characters with uppercase, lowercase, number, and symbol",
    }
  )
  password: string;

  @IsString()
  @MinLength(2, { message: "Name must be at least 2 characters long" })
  @MaxLength(50, { message: "Name must not exceed 50 characters" })
  @Matches(/^[a-zA-Z\s'-]+$/, {
    message: "Name can only contain letters, spaces, hyphens, and apostrophes",
  })
  @Transform(({ value }) => value?.trim())
  name: string;

  @IsOptional()
  @IsString()
  @MaxLength(20, { message: "Username must not exceed 20 characters" })
  @Matches(/^[a-zA-Z0-9_-]+$/, {
    message:
      "Username can only contain letters, numbers, underscores, and hyphens",
  })
  @Transform(({ value }) => value?.toLowerCase().trim())
  username?: string;

  @IsOptional()
  @IsUrl({}, { message: "Please provide a valid URL" })
  website?: string;

  @IsInt()
  @Min(18, { message: "User must be at least 18 years old" })
  @Max(120, { message: "Age must be realistic" })
  age: number;

  @IsEnum(UserRole, { message: "Role must be user, admin, or moderator" })
  role: UserRole;
}

// src/dtos/address.dto.ts
export class AddressDto {
  @IsString()
  @MinLength(5)
  @MaxLength(100)
  street: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  city: string;

  @IsString()
  @Matches(/^[A-Z]{2}$/, { message: "State must be a 2-letter code" })
  @Transform(({ value }) => value?.toUpperCase())
  state: string;

  @IsString()
  @Matches(/^\d{5}(-\d{4})?$/, { message: "Invalid ZIP code format" })
  zipCode: string;
}

// src/dtos/create-user-with-address.dto.ts
export class CreateUserWithAddressDto extends CreateUserDto {
  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;

  @IsOptional()
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(5)
  @IsString({ each: true })
  @Transform(({ value }) =>
    value?.map((tag: string) => tag.trim().toLowerCase())
  )
  tags?: string[];
}
```

### 2. Custom Validators

```typescript
// src/validators/is-username-unique.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from "class-validator";
import { Injectable } from "@nestjs/common";
import { UserRepository } from "../repositories/user.repository";

@ValidatorConstraint({ async: true })
@Injectable()
export class IsUsernameUniqueConstraint
  implements ValidatorConstraintInterface
{
  constructor(private userRepository: UserRepository) {}

  async validate(username: string, args: ValidationArguments) {
    const user = await this.userRepository.findByUsername(username);
    return !user;
  }

  defaultMessage(args: ValidationArguments) {
    return "Username $value is already taken";
  }
}

export function IsUsernameUnique(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsUsernameUniqueConstraint,
    });
  };
}

// Usage in DTO
export class CreateUserDto {
  @IsString()
  @IsUsernameUnique()
  username: string;
}

// src/validators/is-password-match.validator.ts
@ValidatorConstraint({ async: false })
export class IsPasswordMatchConstraint implements ValidatorConstraintInterface {
  validate(confirmPassword: string, args: ValidationArguments) {
    const object = args.object as any;
    return confirmPassword === object.password;
  }

  defaultMessage(args: ValidationArguments) {
    return "Passwords do not match";
  }
}

export function IsPasswordMatch(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsPasswordMatchConstraint,
    });
  };
}

// Usage
export class RegisterDto {
  @IsStrongPassword()
  password: string;

  @IsString()
  @IsPasswordMatch()
  confirmPassword: string;
}
```

### 3. Controller Validation

```typescript
// src/controllers/user.controller.ts
import {
  Controller,
  Post,
  Body,
  ValidationPipe,
  UsePipes,
} from "@nestjs/common";
import { CreateUserDto } from "../dtos/create-user.dto";

@Controller("users")
export class UserController {
  @Post()
  @UsePipes(
    new ValidationPipe({
      whitelist: true, // Strip properties not in DTO
      forbidNonWhitelisted: true, // Throw error for extra properties
      transform: true, // Transform payload to DTO instance
      transformOptions: {
        enableImplicitConversion: true,
      },
    })
  )
  async createUser(@Body() createUserDto: CreateUserDto) {
    return this.userService.create(createUserDto);
  }
}

// main.ts - Global validation pipe
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: {
      enableImplicitConversion: true,
    },
    exceptionFactory: (errors) => {
      const messages = errors.map((error) => ({
        field: error.property,
        errors: Object.values(error.constraints || {}),
      }));
      return new BadRequestException({
        message: "Validation failed",
        errors: messages,
      });
    },
  })
);
```

## SQL Injection Prevention

### 1. Parameterized Queries with TypeORM

```typescript
// ❌ VULNERABLE - Never do this
async findUserByEmail(email: string) {
  return this.userRepository.query(
    `SELECT * FROM users WHERE email = '${email}'`
  );
}

// ✅ SAFE - Use parameterized queries
async findUserByEmail(email: string) {
  return this.userRepository.query(
    'SELECT * FROM users WHERE email = $1',
    [email]
  );
}

// ✅ BETTER - Use query builder
async findUserByEmail(email: string) {
  return this.userRepository
    .createQueryBuilder('user')
    .where('user.email = :email', { email })
    .getOne();
}

// ✅ BEST - Use repository methods
async findUserByEmail(email: string) {
  return this.userRepository.findOne({
    where: { email },
  });
}
```

### 2. Prisma (SQL Injection Safe by Default)

```typescript
// Prisma automatically parameterizes queries
async findUserByEmail(email: string) {
  return this.prisma.user.findUnique({
    where: { email },
  });
}

// Raw queries still need parameterization
async rawQuery(email: string) {
  return this.prisma.$queryRaw`
    SELECT * FROM users WHERE email = ${email}
  `;
}
```

## XSS (Cross-Site Scripting) Prevention

### 1. HTML Sanitization

```typescript
// src/utils/sanitizer.util.ts
import * as sanitizeHtml from "sanitize-html";
import * as DOMPurify from "isomorphic-dompurify";

export class Sanitizer {
  // Basic HTML sanitization
  static sanitizeHtml(dirty: string): string {
    return sanitizeHtml(dirty, {
      allowedTags: sanitizeHtml.defaults.allowedTags.concat(["img"]),
      allowedAttributes: {
        ...sanitizeHtml.defaults.allowedAttributes,
        img: ["src", "alt"],
        a: ["href", "name", "target"],
      },
      allowedSchemes: ["http", "https", "mailto"],
    });
  }

  // Strict sanitization (text only)
  static sanitizeText(dirty: string): string {
    return sanitizeHtml(dirty, {
      allowedTags: [],
      allowedAttributes: {},
    });
  }

  // Rich text editor content
  static sanitizeRichText(dirty: string): string {
    return sanitizeHtml(dirty, {
      allowedTags: [
        "h1",
        "h2",
        "h3",
        "h4",
        "h5",
        "h6",
        "p",
        "br",
        "strong",
        "em",
        "u",
        "s",
        "ul",
        "ol",
        "li",
        "a",
        "img",
        "blockquote",
        "code",
        "pre",
      ],
      allowedAttributes: {
        a: ["href", "target"],
        img: ["src", "alt", "width", "height"],
      },
      allowedSchemes: ["http", "https"],
    });
  }

  // Using DOMPurify
  static purify(dirty: string): string {
    return DOMPurify.sanitize(dirty, {
      ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p"],
      ALLOWED_ATTR: ["href"],
    });
  }
}

// src/dtos/create-post.dto.ts
import { IsString, MaxLength } from "class-validator";
import { Transform } from "class-transformer";
import { Sanitizer } from "../utils/sanitizer.util";

export class CreatePostDto {
  @IsString()
  @MaxLength(200)
  @Transform(({ value }) => Sanitizer.sanitizeText(value))
  title: string;

  @IsString()
  @MaxLength(10000)
  @Transform(({ value }) => Sanitizer.sanitizeRichText(value))
  content: string;
}
```

### 2. Output Encoding in Templates

```typescript
// Use template engines that auto-escape by default (Handlebars, EJS)

// Handlebars (auto-escapes by default)
<div>{{userInput}}</div> <!-- Safe -->
<div>{{{userInput}}}</div> <!-- Unsafe - triple braces disable escaping -->

// React (auto-escapes by default)
<div>{userInput}</div> {/* Safe */}
<div dangerouslySetInnerHTML={{__html: userInput}} /> {/* Unsafe */}
```

## NoSQL Injection Prevention

```typescript
// MongoDB/Mongoose
// ❌ VULNERABLE
async findUser(username: string) {
  return this.userModel.findOne({
    username: username, // If username is an object like {$gt: ""}, this is vulnerable
  });
}

// ✅ SAFE - Validate input type
async findUser(username: string) {
  if (typeof username !== 'string') {
    throw new BadRequestException('Invalid username format');
  }

  return this.userModel.findOne({
    username: username,
  });
}

// ✅ BETTER - Use schema validation
const UserSchema = new Schema({
  username: {
    type: String,
    required: true,
    validate: {
      validator: (v: any) => typeof v === 'string',
      message: 'Username must be a string',
    },
  },
});
```

## File Upload Validation

```typescript
// src/validators/file-upload.validator.ts
import { FileValidator } from "@nestjs/common";

export class FileTypeValidator extends FileValidator {
  constructor(private allowedTypes: string[]) {
    super({});
  }

  isValid(file: Express.Multer.File): boolean {
    return this.allowedTypes.includes(file.mimetype);
  }

  buildErrorMessage(): string {
    return `File type not allowed. Allowed types: ${this.allowedTypes.join(
      ", "
    )}`;
  }
}

export class FileSizeValidator extends FileValidator {
  constructor(private maxSize: number) {
    super({});
  }

  isValid(file: Express.Multer.File): boolean {
    return file.size <= this.maxSize;
  }

  buildErrorMessage(): string {
    return `File size exceeds maximum of ${this.maxSize} bytes`;
  }
}

// src/controllers/upload.controller.ts
import {
  Controller,
  Post,
  UseInterceptors,
  UploadedFile,
  ParseFilePipe,
  MaxFileSizeValidator,
  FileTypeValidator as NestFileTypeValidator,
} from "@nestjs/common";
import { FileInterceptor } from "@nestjs/platform-express";
import { diskStorage } from "multer";
import * as path from "path";
import { v4 as uuidv4 } from "uuid";

@Controller("upload")
export class UploadController {
  @Post("image")
  @UseInterceptors(
    FileInterceptor("file", {
      storage: diskStorage({
        destination: "./uploads",
        filename: (req, file, cb) => {
          const uniqueName = `${uuidv4()}${path.extname(file.originalname)}`;
          cb(null, uniqueName);
        },
      }),
      limits: {
        fileSize: 5 * 1024 * 1024, // 5MB
      },
    })
  )
  async uploadImage(
    @UploadedFile(
      new ParseFilePipe({
        validators: [
          new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }),
          new NestFileTypeValidator({
            fileType: /^image\/(jpeg|jpg|png|gif|webp)$/,
          }),
        ],
      })
    )
    file: Express.Multer.File
  ) {
    // Additional validation
    const allowedExtensions = [".jpg", ".jpeg", ".png", ".gif", ".webp"];
    const ext = path.extname(file.originalname).toLowerCase();

    if (!allowedExtensions.includes(ext)) {
      throw new BadRequestException("Invalid file extension");
    }

    // Sanitize filename
    const sanitizedFilename = file.filename.replace(/[^a-zA-Z0-9.-]/g, "_");

    return {
      filename: sanitizedFilename,
      path: `/uploads/${sanitizedFilename}`,
      size: file.size,
      mimetype: file.mimetype,
    };
  }
}
```

## URL Validation & Sanitization

```typescript
// src/validators/url.validator.ts
import { IsUrl } from "class-validator";
import { Transform } from "class-transformer";

export class UrlDto {
  @IsUrl({
    protocols: ["http", "https"],
    require_protocol: true,
    require_valid_protocol: true,
    require_tld: true,
  })
  @Transform(({ value }) => {
    try {
      const url = new URL(value);

      // Prevent SSRF attacks - block private IPs
      const hostname = url.hostname;
      if (
        hostname === "localhost" ||
        hostname.startsWith("127.") ||
        hostname.startsWith("192.168.") ||
        hostname.startsWith("10.") ||
        /^172\.(1[6-9]|2\d|3[01])\./.test(hostname)
      ) {
        throw new Error("Private IP addresses not allowed");
      }

      return url.href;
    } catch (error) {
      return value;
    }
  })
  url: string;
}
```

## Command Injection Prevention

```typescript
// ❌ VULNERABLE - Never execute user input directly
import { exec } from 'child_process';

async processFile(filename: string) {
  exec(`convert ${filename} output.png`, (error, stdout) => {
    // Vulnerable to injection if filename contains "; rm -rf /"
  });
}

// ✅ SAFE - Use parameterized execution
import { execFile } from 'child_process';
import { promisify } from 'util';

const execFileAsync = promisify(execFile);

async processFile(filename: string) {
  // Validate filename
  if (!/^[a-zA-Z0-9_.-]+$/.test(filename)) {
    throw new BadRequestException('Invalid filename');
  }

  // Use execFile with arguments array
  try {
    const { stdout } = await execFileAsync('convert', [filename, 'output.png']);
    return stdout;
  } catch (error) {
    throw new InternalServerErrorException('File processing failed');
  }
}

// ✅ BETTER - Use libraries instead of shell commands
import sharp from 'sharp';

async processFile(filename: string) {
  await sharp(filename)
    .png()
    .toFile('output.png');
}
```

## Request Rate Limiting by Input

```typescript
// src/guards/rate-limit.guard.ts
import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { ThrottlerGuard, ThrottlerException } from "@nestjs/throttler";

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();

    // Rate limit by IP + username combination for login attempts
    const key = `${request.ip}:${request.body?.username || "anonymous"}`;

    // Check rate limit
    // Implementation depends on your throttler configuration

    return super.canActivate(context);
  }
}
```

## Best Practices

### 1. **Never Trust User Input**

- Validate all input server-side
- Client-side validation is UX, not security
- Validate type, format, length, and content
- Use whitelisting over blacklisting

### 2. **Use Strong Validation Libraries**

- class-validator for DTOs
- Joi for complex validation
- validator.js for specific formats
- Custom validators when needed

### 3. **Sanitize Output**

- HTML-encode output in templates
- Use auto-escaping template engines
- Sanitize rich text content
- Be careful with dangerouslySetInnerHTML

### 4. **Parameterized Queries**

- Always use parameterized queries
- Never concatenate SQL strings
- Use ORM query builders
- Validate numeric IDs

### 5. **File Upload Security**

- Validate file type by content, not extension
- Limit file size
- Sanitize filenames
- Store outside webroot
- Scan for malware

### 6. **Error Messages**

- Don't leak sensitive information
- Generic error messages for users
- Detailed logs for developers
- Avoid SQL errors in responses

### 7. **Defense in Depth**

- Multiple layers of validation
- Validate at boundaries
- Sanitize before storage
- Escape on output

### 8. **Regular Updates**

- Keep dependencies updated
- Monitor security advisories
- Automated dependency scanning
- Test validation logic

## Key Takeaways

✅ **Validate all user input on the server side**  
✅ **Use parameterized queries to prevent SQL injection**  
✅ **Sanitize HTML content to prevent XSS attacks**  
✅ **Whitelist allowed values instead of blacklisting bad ones**  
✅ **Validate file uploads by content, not just extension**  
✅ **Never execute user input as commands**  
✅ **Use strong validation libraries like class-validator**  
✅ **Implement defense in depth with multiple validation layers**  
✅ **Keep error messages generic to avoid information leakage**  
✅ **Regular security audits and dependency updates**

Input validation and sanitization are your first and most important defense against injection attacks and data corruption.
