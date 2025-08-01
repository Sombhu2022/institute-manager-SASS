# Security Implementation

This document outlines the comprehensive security implementation for the Educational Institution Management SaaS platform.

## Overview

Security is a critical aspect of the Educational Institution Management SaaS platform, as it handles sensitive student data, financial information, and institutional records. The security implementation follows industry best practices and regulatory requirements to ensure data protection, privacy, and compliance.

## Authentication and Authorization

### User Authentication

#### JWT-Based Authentication

```typescript
// src/services/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { UserService } from './user.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private userService: UserService,
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  async validateUser(email: string, password: string, tenantId: string): Promise<any> {
    const user = await this.userService.findByEmail(email, tenantId);
    
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }
    
    const isPasswordValid = await bcrypt.compare(password, user.password);
    
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }
    
    // Check if user is active
    if (user.status !== 'active') {
      throw new UnauthorizedException('User account is not active');
    }
    
    // Remove password from returned user object
    const { password: _, ...result } = user;
    return result;
  }

  async login(user: any) {
    // Create payload for JWT
    const payload = {
      sub: user._id,
      email: user.email,
      roles: user.roles,
      permissions: user.permissions,
      tenantId: user.tenantId,
    };
    
    // Generate access token (short-lived)
    const accessToken = this.jwtService.sign(payload, {
      secret: this.configService.get('JWT_SECRET'),
      expiresIn: '15m',
    });
    
    // Generate refresh token (long-lived)
    const refreshToken = this.jwtService.sign(
      { sub: user._id, tenantId: user.tenantId },
      {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: '7d',
      },
    );
    
    // Store refresh token hash in database
    const refreshTokenHash = await bcrypt.hash(refreshToken, 10);
    await this.userService.storeRefreshToken(user._id, refreshTokenHash);
    
    return {
      accessToken,
      refreshToken,
      user: {
        id: user._id,
        email: user.email,
        name: user.name,
        roles: user.roles,
      },
    };
  }

  async refreshToken(refreshToken: string) {
    try {
      // Verify refresh token
      const payload = this.jwtService.verify(refreshToken, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
      });
      
      // Get user from database
      const user = await this.userService.findById(payload.sub, payload.tenantId);
      
      if (!user || !user.refreshToken) {
        throw new UnauthorizedException('Invalid refresh token');
      }
      
      // Verify stored refresh token hash
      const isRefreshTokenValid = await bcrypt.compare(
        refreshToken,
        user.refreshToken,
      );
      
      if (!isRefreshTokenValid) {
        throw new UnauthorizedException('Invalid refresh token');
      }
      
      // Generate new tokens
      return this.login(user);
    } catch (error) {
      throw new UnauthorizedException('Invalid refresh token');
    }
  }

  async logout(userId: string) {
    // Remove refresh token from database
    await this.userService.removeRefreshToken(userId);
    return { success: true };
  }
}
```

#### Multi-Factor Authentication

```typescript
// src/services/mfa.service.ts
import { Injectable } from '@nestjs/common';
import { UserService } from './user.service';
import * as speakeasy from 'speakeasy';
import * as QRCode from 'qrcode';

@Injectable()
export class MfaService {
  constructor(private userService: UserService) {}

  async generateMfaSecret(userId: string, tenantId: string) {
    // Get user
    const user = await this.userService.findById(userId, tenantId);
    
    // Generate new secret
    const secret = speakeasy.generateSecret({
      name: `EduSaaS:${user.email}`,
      length: 20,
    });
    
    // Store secret in database (temporarily until verified)
    await this.userService.storeTempMfaSecret(userId, secret.base32);
    
    // Generate QR code
    const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url);
    
    return {
      secret: secret.base32,
      qrCodeUrl,
    };
  }

  async verifyAndEnableMfa(userId: string, token: string, tenantId: string) {
    // Get user with temp MFA secret
    const user = await this.userService.findById(userId, tenantId);
    
    if (!user.tempMfaSecret) {
      throw new Error('No MFA setup in progress');
    }
    
    // Verify token
    const isValid = speakeasy.totp.verify({
      secret: user.tempMfaSecret,
      encoding: 'base32',
      token,
    });
    
    if (!isValid) {
      throw new Error('Invalid MFA token');
    }
    
    // Enable MFA for user
    await this.userService.enableMfa(userId, user.tempMfaSecret);
    
    return { success: true };
  }

  async verifyMfaToken(userId: string, token: string, tenantId: string) {
    // Get user
    const user = await this.userService.findById(userId, tenantId);
    
    if (!user.mfaEnabled || !user.mfaSecret) {
      throw new Error('MFA not enabled for this user');
    }
    
    // Verify token
    const isValid = speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token,
      window: 1, // Allow 1 step before/after for time drift
    });
    
    return { isValid };
  }

  async disableMfa(userId: string, token: string, tenantId: string) {
    // Verify token first
    const verification = await this.verifyMfaToken(userId, token, tenantId);
    
    if (!verification.isValid) {
      throw new Error('Invalid MFA token');
    }
    
    // Disable MFA
    await this.userService.disableMfa(userId);
    
    return { success: true };
  }
}
```

### Role-Based Access Control (RBAC)

```typescript
// src/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (!requiredRoles) {
      return true; // No roles required
    }
    
    const { user } = context.switchToHttp().getRequest();
    
    if (!user || !user.roles) {
      return false;
    }
    
    return requiredRoles.some(role => user.roles.includes(role));
  }
}
```

```typescript
// src/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

### Permission-Based Access Control (PBAC)

```typescript
// src/guards/permissions.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>('permissions', [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (!requiredPermissions) {
      return true; // No permissions required
    }
    
    const { user } = context.switchToHttp().getRequest();
    
    if (!user || !user.permissions) {
      return false;
    }
    
    return requiredPermissions.every(permission =>
      user.permissions.includes(permission),
    );
  }
}
```

```typescript
// src/decorators/permissions.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Permissions = (...permissions: string[]) => SetMetadata('permissions', permissions);
```

## Data Protection

### Data Encryption

#### Encryption at Rest

```typescript
// src/services/encryption.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;

  constructor(private configService: ConfigService) {
    // Derive key from environment variable
    const encryptionKey = this.configService.get<string>('ENCRYPTION_KEY');
    this.key = crypto.scryptSync(encryptionKey, 'salt', 32);
  }

  encrypt(text: string): { encryptedData: string; iv: string; authTag: string } {
    // Generate initialization vector
    const iv = crypto.randomBytes(16);
    
    // Create cipher
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
    
    // Encrypt data
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    // Get auth tag for GCM mode
    const authTag = cipher.getAuthTag().toString('hex');
    
    return {
      encryptedData: encrypted,
      iv: iv.toString('hex'),
      authTag,
    };
  }

  decrypt(encryptedData: string, iv: string, authTag: string): string {
    // Create decipher
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.key,
      Buffer.from(iv, 'hex'),
    );
    
    // Set auth tag
    decipher.setAuthTag(Buffer.from(authTag, 'hex'));
    
    // Decrypt data
    let decrypted = decipher.update(encryptedData, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

#### Field-Level Encryption

```typescript
// src/models/mongodb/plugins/sensitive-data.plugin.ts
import { Schema } from 'mongoose';
import { EncryptionService } from '../../../services/encryption.service';

export function sensitiveDataPlugin(schema: Schema, options: { fields: string[] }) {
  const encryptionService = new EncryptionService(null); // Dependency injection would be better in real app
  
  // Add encrypted fields to schema
  options.fields.forEach(field => {
    // Add fields for storing encryption metadata
    schema.add({
      [`${field}_encrypted`]: { type: String },
      [`${field}_iv`]: { type: String },
      [`${field}_auth_tag`]: { type: String },
    });
    
    // Virtual getter/setter for the original field
    schema.virtual(field)
      .get(function() {
        if (!this[`${field}_encrypted`]) return undefined;
        
        try {
          return encryptionService.decrypt(
            this[`${field}_encrypted`],
            this[`${field}_iv`],
            this[`${field}_auth_tag`],
          );
        } catch (error) {
          console.error(`Error decrypting ${field}:`, error);
          return undefined;
        }
      })
      .set(function(value) {
        if (value === undefined || value === null) {
          this[`${field}_encrypted`] = undefined;
          this[`${field}_iv`] = undefined;
          this[`${field}_auth_tag`] = undefined;
          return;
        }
        
        const encrypted = encryptionService.encrypt(value.toString());
        this[`${field}_encrypted`] = encrypted.encryptedData;
        this[`${field}_iv`] = encrypted.iv;
        this[`${field}_auth_tag`] = encrypted.authTag;
      });
  });
  
  // Pre-save hook to ensure virtual fields are processed
  schema.pre('save', function() {
    options.fields.forEach(field => {
      if (this.isModified(field)) {
        // The virtual setter will handle encryption
        const value = this[field];
        this[field] = value;
      }
    });
  });
  
  // Exclude encrypted fields from JSON output
  schema.set('toJSON', {
    transform: (doc, ret) => {
      options.fields.forEach(field => {
        delete ret[`${field}_encrypted`];
        delete ret[`${field}_iv`];
        delete ret[`${field}_auth_tag`];
      });
      return ret;
    },
    ...schema.get('toJSON'),
  });
}
```

### Data Masking

```typescript
// src/interceptors/data-masking.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface MaskingConfig {
  fields: {
    [key: string]: {
      type: 'full' | 'partial' | 'custom';
      pattern?: RegExp;
      replacement?: string;
      customMask?: (value: string) => string;
    };
  };
}

@Injectable()
export class DataMaskingInterceptor implements NestInterceptor {
  constructor(private readonly config: MaskingConfig) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => {
        return this.maskData(data);
      }),
    );
  }

  private maskData(data: any): any {
    if (!data) return data;
    
    if (Array.isArray(data)) {
      return data.map(item => this.maskData(item));
    }
    
    if (typeof data === 'object' && data !== null) {
      const result = { ...data };
      
      for (const [key, value] of Object.entries(result)) {
        if (this.config.fields[key]) {
          const maskConfig = this.config.fields[key];
          
          if (typeof value === 'string') {
            result[key] = this.applyMask(value, maskConfig);
          }
        } else if (typeof value === 'object' && value !== null) {
          result[key] = this.maskData(value);
        }
      }
      
      return result;
    }
    
    return data;
  }

  private applyMask(value: string, config: MaskingConfig['fields'][string]): string {
    if (!value) return value;
    
    switch (config.type) {
      case 'full':
        return '********';
      
      case 'partial':
        if (config.pattern && config.replacement) {
          return value.replace(config.pattern, config.replacement);
        }
        // Default partial masking (show first and last character)
        return value.length > 2
          ? `${value[0]}${'*'.repeat(value.length - 2)}${value[value.length - 1]}`
          : '**';
      
      case 'custom':
        if (config.customMask && typeof config.customMask === 'function') {
          return config.customMask(value);
        }
        return value;
      
      default:
        return value;
    }
  }
}
```

## Network Security

### HTTPS Implementation

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as fs from 'fs';
import * as path from 'path';

async function bootstrap() {
  // HTTPS options for development
  // In production, this would typically be handled by a load balancer or proxy
  const httpsOptions = process.env.NODE_ENV === 'development'
    ? {
        key: fs.readFileSync(path.join(__dirname, '../ssl/private-key.pem')),
        cert: fs.readFileSync(path.join(__dirname, '../ssl/public-certificate.pem')),
      }
    : undefined;
  
  const app = await NestFactory.create(AppModule, { httpsOptions });
  
  // Set security headers
  app.use((req, res, next) => {
    // HSTS header
    res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    next();
  });
  
  await app.listen(3000);
}

bootstrap();
```

### API Security

```typescript
// src/main.ts (continued)
import { ValidationPipe } from '@nestjs/common';
import * as helmet from 'helmet';
import * as rateLimit from 'express-rate-limit';

async function bootstrap() {
  // ... previous code
  
  // Apply helmet middleware for security headers
  app.use(helmet());
  
  // Apply rate limiting
  app.use(
    rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
      message: 'Too many requests from this IP, please try again later',
    }),
  );
  
  // Enable CORS with specific options
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    methods: 'GET,HEAD,PUT,PATCH,POST,DELETE',
    credentials: true,
  });
  
  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Strip properties not in DTO
      forbidNonWhitelisted: true, // Throw error if non-whitelisted properties are present
      transform: true, // Transform payloads to DTO instances
    }),
  );
  
  await app.listen(3000);
}

bootstrap();
```

## Audit and Compliance

### Audit Logging

```typescript
// src/services/audit-log.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { AuditLog } from '../models/mongodb/audit-log.model';

export enum AuditAction {
  CREATE = 'create',
  READ = 'read',
  UPDATE = 'update',
  DELETE = 'delete',
  LOGIN = 'login',
  LOGOUT = 'logout',
  EXPORT = 'export',
  IMPORT = 'import',
  PAYMENT = 'payment',
  SETTINGS = 'settings',
}

export interface AuditLogDto {
  tenantId: string;
  userId: string;
  action: AuditAction;
  resource: string;
  resourceId?: string;
  description: string;
  metadata?: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
}

@Injectable()
export class AuditLogService {
  constructor(
    @InjectModel(AuditLog.name) private auditLogModel: Model<AuditLog>,
  ) {}

  async log(logDto: AuditLogDto): Promise<AuditLog> {
    const auditLog = new this.auditLogModel({
      ...logDto,
      timestamp: new Date(),
    });
    
    return auditLog.save();
  }

  async findByTenant(
    tenantId: string,
    filters: {
      startDate?: Date;
      endDate?: Date;
      userId?: string;
      action?: AuditAction;
      resource?: string;
      resourceId?: string;
    },
    pagination: { page: number; limit: number },
  ): Promise<{ logs: AuditLog[]; total: number }> {
    const query: any = { tenantId };
    
    if (filters.startDate && filters.endDate) {
      query.timestamp = {
        $gte: filters.startDate,
        $lte: filters.endDate,
      };
    }
    
    if (filters.userId) query.userId = filters.userId;
    if (filters.action) query.action = filters.action;
    if (filters.resource) query.resource = filters.resource;
    if (filters.resourceId) query.resourceId = filters.resourceId;
    
    const total = await this.auditLogModel.countDocuments(query);
    
    const logs = await this.auditLogModel
      .find(query)
      .sort({ timestamp: -1 })
      .skip((pagination.page - 1) * pagination.limit)
      .limit(pagination.limit)
      .exec();
    
    return { logs, total };
  }

  async getAuditTrailForResource(
    tenantId: string,
    resource: string,
    resourceId: string,
  ): Promise<AuditLog[]> {
    return this.auditLogModel
      .find({ tenantId, resource, resourceId })
      .sort({ timestamp: -1 })
      .exec();
  }
}
```

### Audit Log Interceptor

```typescript
// src/interceptors/audit-log.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { AuditLogService, AuditAction } from '../services/audit-log.service';

interface AuditLogConfig {
  action: AuditAction;
  resource: string;
  getResourceId?: (request: any, response: any) => string;
  getDescription?: (request: any, response: any) => string;
  getMetadata?: (request: any, response: any) => Record<string, any>;
}

@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  constructor(
    private readonly auditLogService: AuditLogService,
    private readonly config: AuditLogConfig,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    if (!user) {
      return next.handle();
    }
    
    return next.handle().pipe(
      tap(async (response) => {
        try {
          const resourceId = this.config.getResourceId
            ? this.config.getResourceId(request, response)
            : request.params.id;
          
          const description = this.config.getDescription
            ? this.config.getDescription(request, response)
            : `${this.config.action} ${this.config.resource}`;
          
          const metadata = this.config.getMetadata
            ? this.config.getMetadata(request, response)
            : undefined;
          
          await this.auditLogService.log({
            tenantId: user.tenantId,
            userId: user.sub,
            action: this.config.action,
            resource: this.config.resource,
            resourceId,
            description,
            metadata,
            ipAddress: request.ip,
            userAgent: request.headers['user-agent'],
          });
        } catch (error) {
          console.error('Error logging audit event:', error);
        }
      }),
    );
  }
}
```

### Data Retention and Compliance

```typescript
// src/services/data-retention.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { ConfigService } from '@nestjs/config';
import { TenantService } from './tenant.service';

@Injectable()
export class DataRetentionService {
  private readonly logger = new Logger(DataRetentionService.name);

  constructor(
    @InjectModel('AuditLog') private auditLogModel: Model<any>,
    @InjectModel('Student') private studentModel: Model<any>,
    @InjectModel('Payment') private paymentModel: Model<any>,
    private configService: ConfigService,
    private tenantService: TenantService,
  ) {}

  // Run at 1:00 AM every day
  @Cron('0 1 * * *')
  async handleDataRetention() {
    this.logger.log('Running data retention job');
    
    // Get all tenants
    const tenants = await this.tenantService.findAll();
    
    for (const tenant of tenants) {
      await this.processTenantDataRetention(tenant);
    }
    
    this.logger.log('Data retention job completed');
  }

  private async processTenantDataRetention(tenant: any) {
    try {
      const tenantId = tenant.id;
      const retentionPolicies = tenant.configuration?.dataRetention || this.getDefaultRetentionPolicies();
      
      // Process audit logs
      if (retentionPolicies.auditLogs) {
        const auditLogRetentionDays = retentionPolicies.auditLogs;
        const auditLogCutoffDate = new Date();
        auditLogCutoffDate.setDate(auditLogCutoffDate.getDate() - auditLogRetentionDays);
        
        const result = await this.auditLogModel.deleteMany({
          tenantId,
          timestamp: { $lt: auditLogCutoffDate },
        });
        
        this.logger.log(`Deleted ${result.deletedCount} audit logs for tenant ${tenantId}`);
      }
      
      // Process inactive students
      if (retentionPolicies.inactiveStudents) {
        const inactiveStudentRetentionDays = retentionPolicies.inactiveStudents;
        const inactiveStudentCutoffDate = new Date();
        inactiveStudentCutoffDate.setDate(inactiveStudentCutoffDate.getDate() - inactiveStudentRetentionDays);
        
        const result = await this.studentModel.deleteMany({
          tenantId,
          status: 'inactive',
          updatedAt: { $lt: inactiveStudentCutoffDate },
        });
        
        this.logger.log(`Deleted ${result.deletedCount} inactive students for tenant ${tenantId}`);
      }
      
      // Process completed payments
      if (retentionPolicies.completedPayments) {
        const completedPaymentRetentionDays = retentionPolicies.completedPayments;
        const completedPaymentCutoffDate = new Date();
        completedPaymentCutoffDate.setDate(completedPaymentCutoffDate.getDate() - completedPaymentRetentionDays);
        
        // For payments, we might want to anonymize rather than delete
        const result = await this.paymentModel.updateMany(
          {
            tenantId,
            status: 'completed',
            updatedAt: { $lt: completedPaymentCutoffDate },
            anonymized: { $ne: true },
          },
          {
            $set: {
              anonymized: true,
              studentDetails: null,
              metadata: null,
              // Keep essential data for reporting
              // amount, date, course, etc.
            },
          },
        );
        
        this.logger.log(`Anonymized ${result.modifiedCount} completed payments for tenant ${tenantId}`);
      }
    } catch (error) {
      this.logger.error(`Error processing data retention for tenant ${tenant.id}:`, error);
    }
  }

  private getDefaultRetentionPolicies() {
    return {
      auditLogs: 365, // 1 year
      inactiveStudents: 730, // 2 years
      completedPayments: 2555, // 7 years (for financial records)
    };
  }
}
```

## Vulnerability Management

### Input Validation

```typescript
// src/dto/student.dto.ts
import { IsString, IsEmail, IsOptional, IsDate, IsEnum, Length, Matches, IsArray, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';

export class AddressDto {
  @IsString()
  @Length(1, 100)
  line1: string;
  
  @IsString()
  @IsOptional()
  @Length(0, 100)
  line2?: string;
  
  @IsString()
  @Length(1, 50)
  city: string;
  
  @IsString()
  @Length(1, 50)
  state: string;
  
  @IsString()
  @Length(1, 20)
  @Matches(/^[0-9a-zA-Z\-\s]+$/) // Alphanumeric, hyphens, spaces
  postalCode: string;
  
  @IsString()
  @Length(2, 2)
  @Matches(/^[A-Z]{2}$/) // Two uppercase letters
  countryCode: string;
}

export class ContactDto {
  @IsString()
  @Length(1, 50)
  name: string;
  
  @IsString()
  @Length(10, 15)
  @Matches(/^\+?[0-9]+$/) // Numbers with optional leading +
  phone: string;
  
  @IsEmail()
  @Length(5, 100)
  email: string;
  
  @IsString()
  @IsOptional()
  @Length(1, 50)
  relationship?: string;
}

export class CreateStudentDto {
  @IsString()
  @Length(2, 50)
  firstName: string;
  
  @IsString()
  @Length(2, 50)
  lastName: string;
  
  @IsEmail()
  @Length(5, 100)
  email: string;
  
  @IsString()
  @Length(10, 15)
  @Matches(/^\+?[0-9]+$/) // Numbers with optional leading +
  phone: string;
  
  @IsDate()
  @Type(() => Date)
  dateOfBirth: Date;
  
  @IsEnum(['male', 'female', 'other', 'prefer_not_to_say'])
  gender: string;
  
  @IsString()
  @IsOptional()
  @Length(0, 500)
  notes?: string;
  
  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
  
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ContactDto)
  emergencyContacts: ContactDto[];
}
```

### XSS Prevention

```typescript
// src/pipes/sanitize.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';
import * as sanitizeHtml from 'sanitize-html';

@Injectable()
export class SanitizePipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (!value) return value;
    
    if (typeof value === 'string') {
      return this.sanitizeString(value);
    }
    
    if (typeof value === 'object' && value !== null) {
      return this.sanitizeObject(value);
    }
    
    return value;
  }

  private sanitizeString(text: string): string {
    return sanitizeHtml(text, {
      allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
      allowedAttributes: {
        'a': ['href', 'target'],
      },
      allowedIframeHostnames: [],
    });
  }

  private sanitizeObject(obj: Record<string, any>): Record<string, any> {
    const result = { ...obj };
    
    for (const [key, value] of Object.entries(result)) {
      if (typeof value === 'string') {
        result[key] = this.sanitizeString(value);
      } else if (typeof value === 'object' && value !== null) {
        if (Array.isArray(value)) {
          result[key] = value.map(item => 
            typeof item === 'object' && item !== null
              ? this.sanitizeObject(item)
              : typeof item === 'string'
              ? this.sanitizeString(item)
              : item
          );
        } else {
          result[key] = this.sanitizeObject(value);
        }
      }
    }
    
    return result;
  }
}
```

### CSRF Protection

```typescript
// src/main.ts (continued)
import * as csurf from 'csurf';
import * as cookieParser from 'cookie-parser';

async function bootstrap() {
  // ... previous code
  
  // Parse cookies
  app.use(cookieParser());
  
  // CSRF protection
  app.use(
    csurf({
      cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict',
      },
    }),
  );
  
  // Add CSRF token to response
  app.use((req, res, next) => {
    res.cookie('XSRF-TOKEN', req.csrfToken(), {
      httpOnly: false, // Client-side JavaScript needs to read this
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
    });
    next();
  });
  
  await app.listen(3000);
}

bootstrap();
```

## Security Monitoring and Incident Response

### Security Event Monitoring

```typescript
// src/services/security-monitoring.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { ConfigService } from '@nestjs/config';
import { AuditLogService } from './audit-log.service';

interface SecurityEvent {
  tenantId: string;
  eventType: string;
  severity: 'low' | 'medium' | 'high' | 'critical';
  source: string;
  description: string;
  metadata: Record<string, any>;
  timestamp: Date;
}

@Injectable()
export class SecurityMonitoringService {
  private readonly logger = new Logger(SecurityMonitoringService.name);

  constructor(
    @InjectModel('SecurityEvent') private securityEventModel: Model<SecurityEvent>,
    private configService: ConfigService,
    private auditLogService: AuditLogService,
  ) {}

  async logSecurityEvent(event: Omit<SecurityEvent, 'timestamp'>): Promise<void> {
    try {
      // Log to database
      await this.securityEventModel.create({
        ...event,
        timestamp: new Date(),
      });
      
      // Log critical events to application logs
      if (event.severity === 'critical' || event.severity === 'high') {
        this.logger.warn(
          `Security event: ${event.eventType} - ${event.description} (Tenant: ${event.tenantId})`,
          event.metadata,
        );
      }
      
      // Send notifications for critical events
      if (event.severity === 'critical') {
        await this.sendSecurityNotification(event);
      }
    } catch (error) {
      this.logger.error('Error logging security event:', error);
    }
  }

  async detectAnomalousLogin(loginData: {
    tenantId: string;
    userId: string;
    ipAddress: string;
    userAgent: string;
    timestamp: Date;
    success: boolean;
  }): Promise<boolean> {
    try {
      // Get user's recent login history
      const recentLogins = await this.auditLogService.findByTenant(
        loginData.tenantId,
        {
          userId: loginData.userId,
          action: 'login',
          startDate: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000), // 30 days
        },
        { page: 1, limit: 10 },
      );
      
      // Check for anomalies
      const isAnomalous = this.analyzeLoginForAnomalies(
        loginData,
        recentLogins.logs,
      );
      
      if (isAnomalous) {
        await this.logSecurityEvent({
          tenantId: loginData.tenantId,
          eventType: 'anomalous_login',
          severity: 'medium',
          source: 'auth_service',
          description: `Anomalous login detected for user ${loginData.userId}`,
          metadata: {
            ipAddress: loginData.ipAddress,
            userAgent: loginData.userAgent,
            timestamp: loginData.timestamp,
            success: loginData.success,
          },
        });
      }
      
      return isAnomalous;
    } catch (error) {
      this.logger.error('Error detecting anomalous login:', error);
      return false;
    }
  }

  private analyzeLoginForAnomalies(
    currentLogin: {
      ipAddress: string;
      userAgent: string;
      timestamp: Date;
      success: boolean;
    },
    recentLogins: any[],
  ): boolean {
    // No previous logins, can't determine anomaly
    if (recentLogins.length === 0) {
      return false;
    }
    
    // Check for new IP address
    const knownIPs = new Set(recentLogins.map(log => log.ipAddress));
    const isNewIP = !knownIPs.has(currentLogin.ipAddress);
    
    // Check for unusual time
    const loginHour = currentLogin.timestamp.getHours();
    const userTypicalLoginHours = recentLogins
      .map(log => new Date(log.timestamp).getHours())
      .reduce((acc, hour) => {
        acc[hour] = (acc[hour] || 0) + 1;
        return acc;
      }, {});
    
    const isUnusualTime = !userTypicalLoginHours[loginHour];
    
    // Check for rapid location change
    // This would require IP geolocation, simplified here
    const isRapidLocationChange = false;
    
    // Check for multiple failed attempts
    const recentFailedAttempts = recentLogins.filter(
      log => log.metadata?.success === false && 
            new Date(log.timestamp).getTime() > Date.now() - 24 * 60 * 60 * 1000
    ).length;
    
    const hasMultipleFailedAttempts = recentFailedAttempts >= 3;
    
    // Determine if anomalous based on factors
    return (
      (isNewIP && isUnusualTime) ||
      isRapidLocationChange ||
      hasMultipleFailedAttempts
    );
  }

  private async sendSecurityNotification(event: Omit<SecurityEvent, 'timestamp'>): Promise<void> {
    // Implementation would depend on notification service
    // Could send email, SMS, or integrate with incident management system
    this.logger.log(`Security notification would be sent for: ${event.eventType}`);
  }
}
```

### Brute Force Protection

```typescript
// src/guards/brute-force.guard.ts
import { Injectable, CanActivate, ExecutionContext, HttpException, HttpStatus } from '@nestjs/common';
import { Observable } from 'rxjs';
import * as Redis from 'ioredis';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class BruteForceGuard implements CanActivate {
  private readonly redis: Redis.Redis;
  private readonly maxAttempts: number;
  private readonly blockDuration: number; // in seconds

  constructor(private configService: ConfigService) {
    this.redis = new Redis.Redis({
      host: this.configService.get('REDIS_HOST'),
      port: this.configService.get('REDIS_PORT'),
      password: this.configService.get('REDIS_PASSWORD'),
    });
    
    this.maxAttempts = this.configService.get('MAX_LOGIN_ATTEMPTS') || 5;
    this.blockDuration = this.configService.get('LOGIN_BLOCK_DURATION') || 15 * 60; // 15 minutes
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const ip = request.ip;
    const email = request.body.email?.toLowerCase();
    
    if (!email) {
      return true; // No email provided, can't track attempts
    }
    
    // Keys for tracking
    const ipKey = `login_attempts:ip:${ip}`;
    const emailKey = `login_attempts:email:${email}`;
    const blockedIpKey = `login_blocked:ip:${ip}`;
    const blockedEmailKey = `login_blocked:email:${email}`;
    
    // Check if IP or email is blocked
    const [isIpBlocked, isEmailBlocked] = await Promise.all([
      this.redis.exists(blockedIpKey),
      this.redis.exists(blockedEmailKey),
    ]);
    
    if (isIpBlocked || isEmailBlocked) {
      // Get remaining block time
      const [ipTtl, emailTtl] = await Promise.all([
        this.redis.ttl(blockedIpKey),
        this.redis.ttl(blockedEmailKey),
      ]);
      
      const blockTimeRemaining = Math.max(ipTtl, emailTtl);
      
      throw new HttpException(
        {
          message: 'Too many failed login attempts. Please try again later.',
          blockTimeRemaining,
        },
        HttpStatus.TOO_MANY_REQUESTS,
      );
    }
    
    // Track this attempt
    const multi = this.redis.multi();
    multi.incr(ipKey);
    multi.incr(emailKey);
    multi.expire(ipKey, this.blockDuration * 2); // IP tracking lasts longer
    multi.expire(emailKey, this.blockDuration * 2);
    const results = await multi.exec();
    
    // Check if max attempts reached
    const ipAttempts = results[0][1] as number;
    const emailAttempts = results[1][1] as number;
    
    if (ipAttempts > this.maxAttempts * 2 || emailAttempts > this.maxAttempts) {
      // Block further attempts
      const multi = this.redis.multi();
      
      if (ipAttempts > this.maxAttempts * 2) {
        multi.set(blockedIpKey, '1');
        multi.expire(blockedIpKey, this.blockDuration);
      }
      
      if (emailAttempts > this.maxAttempts) {
        multi.set(blockedEmailKey, '1');
        multi.expire(blockedEmailKey, this.blockDuration);
      }
      
      await multi.exec();
      
      throw new HttpException(
        {
          message: 'Too many failed login attempts. Please try again later.',
          blockTimeRemaining: this.blockDuration,
        },
        HttpStatus.TOO_MANY_REQUESTS,
      );
    }
    
    // Store request for post-processing
    request.loginAttemptTracking = {
      ipKey,
      emailKey,
      ipAttempts,
      emailAttempts,
    };
    
    return true;
  }
}
```

## Conclusion

This security implementation provides a comprehensive approach to protecting the Educational Institution Management SaaS platform. By implementing robust authentication and authorization mechanisms, data protection measures, network security controls, audit and compliance features, and vulnerability management practices, the platform can securely handle sensitive educational data while meeting regulatory requirements.

The multi-layered security approach ensures that even if one security control fails, others are in place to prevent unauthorized access or data breaches. Regular security monitoring and incident response procedures further enhance the platform's security posture by enabling quick detection and mitigation of potential security threats.

By following these security best practices, the Educational Institution Management SaaS platform can provide educational institutions with a secure environment for managing their operations, protecting student data, and maintaining compliance with relevant regulations.