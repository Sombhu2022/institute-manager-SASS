# Backend Architecture

This document outlines the backend architecture for the Educational Institution Management SaaS Platform, focusing on API design, authentication, multi-tenancy, scalability, and security considerations.

## Technology Stack

### Core Technologies

- **Framework**: Node.js with Express.js or NestJS
- **Language**: TypeScript
- **Database**: MongoDB (primary) and PostgreSQL (transactional data)
- **Authentication**: JWT with refresh token rotation
- **API Documentation**: OpenAPI/Swagger
- **Testing**: Jest, Supertest
- **Logging**: Winston, Morgan
- **Validation**: Joi or class-validator
- **ORM/ODM**: Mongoose (MongoDB) and TypeORM or Prisma (PostgreSQL)

### Infrastructure

- **Containerization**: Docker
- **Orchestration**: Kubernetes
- **CI/CD**: GitHub Actions or GitLab CI
- **Monitoring**: Prometheus, Grafana
- **APM**: New Relic or Datadog
- **Caching**: Redis
- **Message Queue**: RabbitMQ or Apache Kafka
- **File Storage**: AWS S3 or equivalent
- **CDN**: Cloudflare or AWS CloudFront

## Application Structure

```
src/
├── config/                 # Configuration files
│   ├── database.ts         # Database connection configuration
│   ├── auth.ts             # Authentication configuration
│   ├── cache.ts            # Cache configuration
│   ├── storage.ts          # File storage configuration
│   └── index.ts            # Main configuration
├── api/                    # API modules
│   ├── auth/               # Authentication module
│   ├── institutions/       # Institution management module
│   ├── students/           # Student management module
│   ├── courses/            # Course management module
│   ├── staff/              # Staff management module
│   ├── inventory/          # Inventory management module
│   ├── finance/            # Finance management module
│   ├── attendance/         # Attendance management module
│   └── notifications/      # Notification module
├── models/                 # Database models
│   ├── mongodb/            # MongoDB models
│   └── postgresql/         # PostgreSQL models
├── services/               # Business logic services
│   ├── auth.service.ts     # Authentication service
│   ├── tenant.service.ts   # Multi-tenant service
│   ├── student.service.ts  # Student service
│   ├── course.service.ts   # Course service
│   ├── staff.service.ts    # Staff service
│   ├── inventory.service.ts # Inventory service
│   ├── finance.service.ts  # Finance service
│   ├── attendance.service.ts # Attendance service
│   ├── payment.service.ts  # Payment service
│   ├── notification.service.ts # Notification service
│   └── export.service.ts   # Data export service
├── middleware/             # Express middleware
│   ├── auth.middleware.ts  # Authentication middleware
│   ├── tenant.middleware.ts # Multi-tenant middleware
│   ├── validation.middleware.ts # Request validation middleware
│   ├── error.middleware.ts # Error handling middleware
│   └── logging.middleware.ts # Logging middleware
├── utils/                  # Utility functions
│   ├── errors.ts           # Custom error classes
│   ├── validators.ts       # Custom validators
│   ├── helpers.ts          # Helper functions
│   └── constants.ts        # Constants
├── types/                  # TypeScript type definitions
├── jobs/                   # Background jobs
│   ├── reminder.job.ts     # Payment reminder job
│   ├── report.job.ts       # Report generation job
│   └── cleanup.job.ts      # Data cleanup job
├── events/                 # Event handlers
│   ├── payment.events.ts   # Payment events
│   ├── student.events.ts   # Student events
│   └── notification.events.ts # Notification events
└── app.ts                  # Main application entry point
```

## API Design

### RESTful API Design Principles

1. **Resource-Based URLs**
   - Use nouns to represent resources
   - Use plural forms for collections
   - Use hierarchical structure for related resources

2. **HTTP Methods**
   - GET: Retrieve resources
   - POST: Create resources
   - PUT: Update resources (full update)
   - PATCH: Update resources (partial update)
   - DELETE: Remove resources

3. **Status Codes**
   - 200: OK
   - 201: Created
   - 204: No Content
   - 400: Bad Request
   - 401: Unauthorized
   - 403: Forbidden
   - 404: Not Found
   - 409: Conflict
   - 422: Unprocessable Entity
   - 500: Internal Server Error

4. **Query Parameters**
   - Filtering: `?status=active`
   - Sorting: `?sort=name:asc,createdAt:desc`
   - Pagination: `?page=1&limit=10`
   - Field selection: `?fields=name,email,phone`
   - Search: `?search=john`

5. **Response Format**
   ```json
   {
     "data": {},
     "meta": {
       "pagination": {
         "page": 1,
         "limit": 10,
         "total": 100,
         "totalPages": 10
       }
     },
     "links": {
       "self": "/api/students?page=1&limit=10",
       "first": "/api/students?page=1&limit=10",
       "prev": null,
       "next": "/api/students?page=2&limit=10",
       "last": "/api/students?page=10&limit=10"
     }
   }
   ```

### API Versioning

API versioning is implemented using URL path versioning:

```
/api/v1/students
/api/v2/students
```

### API Documentation

OpenAPI/Swagger is used for API documentation:

```typescript
// src/api/students/student.controller.ts
import { Controller, Get, Post, Body, Param, Query } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiParam, ApiQuery } from '@nestjs/swagger';
import { StudentService } from '../../services/student.service';
import { CreateStudentDto, UpdateStudentDto, StudentResponseDto } from './dto';

@ApiTags('students')
@Controller('api/v1/students')
export class StudentController {
  constructor(private readonly studentService: StudentService) {}

  @Get()
  @ApiOperation({ summary: 'Get all students' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  @ApiQuery({ name: 'sort', required: false, type: String })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiQuery({ name: 'status', required: false, enum: ['active', 'inactive'] })
  @ApiResponse({ status: 200, description: 'List of students', type: [StudentResponseDto] })
  async findAll(@Query() query) {
    return this.studentService.findAll(query);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get a student by ID' })
  @ApiParam({ name: 'id', description: 'Student ID' })
  @ApiResponse({ status: 200, description: 'Student details', type: StudentResponseDto })
  @ApiResponse({ status: 404, description: 'Student not found' })
  async findOne(@Param('id') id: string) {
    return this.studentService.findOne(id);
  }

  @Post()
  @ApiOperation({ summary: 'Create a new student' })
  @ApiResponse({ status: 201, description: 'Student created', type: StudentResponseDto })
  @ApiResponse({ status: 400, description: 'Invalid input' })
  async create(@Body() createStudentDto: CreateStudentDto) {
    return this.studentService.create(createStudentDto);
  }

  // Other endpoints...
}
```

## Authentication and Authorization

### JWT Authentication

```typescript
// src/services/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UserModel } from '../models/mongodb/user.model';
import { compare } from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private readonly jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string) {
    const user = await UserModel.findOne({ email }).select('+password');
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await compare(password, user.password);
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    return user;
  }

  async login(user: any) {
    const payload = {
      sub: user._id,
      email: user.email,
      role: user.role,
      tenantId: user.tenantId,
    };

    const accessToken = this.jwtService.sign(payload, {
      expiresIn: '15m',
    });

    const refreshToken = this.jwtService.sign(payload, {
      expiresIn: '7d',
    });

    // Store refresh token in database
    await UserModel.findByIdAndUpdate(user._id, {
      refreshToken: refreshToken,
    });

    return {
      accessToken,
      refreshToken,
    };
  }

  async refreshToken(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken);
      const user = await UserModel.findById(payload.sub);

      if (!user || user.refreshToken !== refreshToken) {
        throw new UnauthorizedException('Invalid refresh token');
      }

      const newPayload = {
        sub: user._id,
        email: user.email,
        role: user.role,
        tenantId: user.tenantId,
      };

      const accessToken = this.jwtService.sign(newPayload, {
        expiresIn: '15m',
      });

      const newRefreshToken = this.jwtService.sign(newPayload, {
        expiresIn: '7d',
      });

      // Update refresh token in database
      await UserModel.findByIdAndUpdate(user._id, {
        refreshToken: newRefreshToken,
      });

      return {
        accessToken,
        refreshToken: newRefreshToken,
      };
    } catch (error) {
      throw new UnauthorizedException('Invalid refresh token');
    }
  }

  async logout(userId: string) {
    await UserModel.findByIdAndUpdate(userId, {
      refreshToken: null,
    });
    return { success: true };
  }
}
```

### Role-Based Access Control

```typescript
// src/middleware/auth.middleware.ts
import { Injectable, NestMiddleware, UnauthorizedException, ForbiddenException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class AuthMiddleware implements NestMiddleware {
  constructor(private readonly jwtService: JwtService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedException('No token provided');
    }

    const token = authHeader.split(' ')[1];
    try {
      const payload = this.jwtService.verify(token);
      req.user = payload;
      next();
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// src/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// src/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    if (!user) {
      throw new ForbiddenException('User not authenticated');
    }

    const hasRole = requiredRoles.some((role) => user.role === role);
    if (!hasRole) {
      throw new ForbiddenException('Insufficient permissions');
    }

    return true;
  }
}
```

### Permission-Based Access Control

```typescript
// src/models/mongodb/role.model.ts
import mongoose from 'mongoose';

const permissionSchema = new mongoose.Schema({
  resource: { type: String, required: true },
  actions: [{ type: String, enum: ['create', 'read', 'update', 'delete'] }],
});

const roleSchema = new mongoose.Schema({
  name: { type: String, required: true, unique: true },
  description: { type: String },
  permissions: [permissionSchema],
  tenantId: { type: mongoose.Schema.Types.ObjectId, ref: 'Institution', required: true },
  isDefault: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

export const RoleModel = mongoose.model('Role', roleSchema);

// src/guards/permission.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { RoleModel } from '../models/mongodb/role.model';

export const PERMISSION_KEY = 'permission';
export const RequirePermission = (resource: string, action: string) =>
  SetMetadata(PERMISSION_KEY, { resource, action });

@Injectable()
export class PermissionGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermission = this.reflector.getAllAndOverride<{ resource: string; action: string }>(
      PERMISSION_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermission) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    if (!user) {
      throw new ForbiddenException('User not authenticated');
    }

    const role = await RoleModel.findOne({
      name: user.role,
      tenantId: user.tenantId,
    });

    if (!role) {
      throw new ForbiddenException('Role not found');
    }

    const hasPermission = role.permissions.some(
      (permission) =>
        permission.resource === requiredPermission.resource &&
        permission.actions.includes(requiredPermission.action),
    );

    if (!hasPermission) {
      throw new ForbiddenException('Insufficient permissions');
    }

    return true;
  }
}
```

## Multi-Tenancy Implementation

### Tenant Identification

```typescript
// src/middleware/tenant.middleware.ts
import { Injectable, NestMiddleware, BadRequestException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { InstitutionModel } from '../models/mongodb/institution.model';

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  async use(req: Request, res: Response, next: NextFunction) {
    // Option 1: Subdomain-based tenant identification
    const hostname = req.hostname;
    const subdomain = hostname.split('.')[0];
    
    if (subdomain !== 'app' && subdomain !== 'www' && subdomain !== 'api') {
      const institution = await InstitutionModel.findOne({ subdomain });
      if (institution) {
        req.tenantId = institution._id;
        next();
        return;
      }
    }
    
    // Option 2: Header-based tenant identification
    const tenantId = req.headers['x-tenant-id'] as string;
    if (tenantId) {
      const institution = await InstitutionModel.findById(tenantId);
      if (institution) {
        req.tenantId = institution._id;
        next();
        return;
      }
    }
    
    // Option 3: JWT token-based tenant identification (from auth middleware)
    if (req.user && req.user.tenantId) {
      req.tenantId = req.user.tenantId;
      next();
      return;
    }
    
    // For public routes or tenant creation, no tenant ID is required
    if (req.path === '/api/v1/institutions' && req.method === 'POST') {
      next();
      return;
    }
    
    throw new BadRequestException('Tenant not identified');
  }
}
```

### Database Multi-Tenancy

#### MongoDB Multi-Tenancy

```typescript
// src/models/mongodb/base.model.ts
import mongoose, { Schema, Document } from 'mongoose';

export interface TenantDocument extends Document {
  tenantId: mongoose.Types.ObjectId;
}

export const tenantSchema = new Schema({
  tenantId: { type: Schema.Types.ObjectId, ref: 'Institution', required: true, index: true },
});

// Apply tenant filter to all queries
export function applyTenantFilter(schema: Schema) {
  schema.pre('find', function() {
    const query = this;
    if (!query.getQuery().tenantId && !query.getQuery()._id) {
      const tenantId = mongoose.connection.get('tenantId');
      if (tenantId) {
        query.where({ tenantId });
      }
    }
  });

  schema.pre('findOne', function() {
    const query = this;
    if (!query.getQuery().tenantId && !query.getQuery()._id) {
      const tenantId = mongoose.connection.get('tenantId');
      if (tenantId) {
        query.where({ tenantId });
      }
    }
  });

  schema.pre('save', function(next) {
    const doc = this as TenantDocument;
    if (!doc.tenantId) {
      const tenantId = mongoose.connection.get('tenantId');
      if (tenantId) {
        doc.tenantId = tenantId;
      }
    }
    next();
  });
}

// src/middleware/database.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import mongoose from 'mongoose';

@Injectable()
export class DatabaseMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    if (req.tenantId) {
      mongoose.connection.set('tenantId', req.tenantId);
    } else {
      mongoose.connection.set('tenantId', null);
    }
    next();
  }
}
```

#### PostgreSQL Multi-Tenancy

```typescript
// src/middleware/database.middleware.ts (extended)
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import mongoose from 'mongoose';
import { getConnection } from 'typeorm';

@Injectable()
export class DatabaseMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Set MongoDB tenant context
    if (req.tenantId) {
      mongoose.connection.set('tenantId', req.tenantId);
    } else {
      mongoose.connection.set('tenantId', null);
    }
    
    // Set PostgreSQL tenant context
    if (req.tenantId) {
      const connection = getConnection();
      connection.createQueryRunner().manager.query(
        `SET app.current_tenant_id = '${req.tenantId}'`
      );
    }
    
    next();
  }
}

// src/config/database.ts (PostgreSQL setup)
import { ConnectionOptions } from 'typeorm';

export const postgresConfig: ConnectionOptions = {
  type: 'postgres',
  host: process.env.POSTGRES_HOST,
  port: parseInt(process.env.POSTGRES_PORT, 10),
  username: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB,
  entities: [__dirname + '/../models/postgresql/**/*.entity{.ts,.js}'],
  migrations: [__dirname + '/../migrations/**/*{.ts,.js}'],
  synchronize: process.env.NODE_ENV !== 'production',
  logging: process.env.NODE_ENV !== 'production',
  cli: {
    entitiesDir: 'src/models/postgresql',
    migrationsDir: 'src/migrations',
  },
};

// PostgreSQL Row-Level Security
// Migration file
export class SetupRowLevelSecurity1234567890123 {
  async up(queryRunner) {
    // Create tenant ID column in all tables
    await queryRunner.query(`
      ALTER TABLE payments ADD COLUMN tenant_id UUID NOT NULL;
      ALTER TABLE expenses ADD COLUMN tenant_id UUID NOT NULL;
      ALTER TABLE invoices ADD COLUMN tenant_id UUID NOT NULL;
      ALTER TABLE invoice_items ADD COLUMN tenant_id UUID NOT NULL;
      ALTER TABLE financial_transactions ADD COLUMN tenant_id UUID NOT NULL;
      ALTER TABLE payment_reminders ADD COLUMN tenant_id UUID NOT NULL;
    `);
    
    // Create tenant ID parameter
    await queryRunner.query(`
      CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS UUID AS $$
      BEGIN
        RETURN current_setting('app.current_tenant_id', true)::UUID;
      END;
      $$ LANGUAGE plpgsql;
    `);
    
    // Enable Row-Level Security
    await queryRunner.query(`
      ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
      ALTER TABLE expenses ENABLE ROW LEVEL SECURITY;
      ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
      ALTER TABLE invoice_items ENABLE ROW LEVEL SECURITY;
      ALTER TABLE financial_transactions ENABLE ROW LEVEL SECURITY;
      ALTER TABLE payment_reminders ENABLE ROW LEVEL SECURITY;
      
      CREATE POLICY tenant_isolation_payments ON payments
        USING (tenant_id = current_tenant_id());
      CREATE POLICY tenant_isolation_expenses ON expenses
        USING (tenant_id = current_tenant_id());
      CREATE POLICY tenant_isolation_invoices ON invoices
        USING (tenant_id = current_tenant_id());
      CREATE POLICY tenant_isolation_invoice_items ON invoice_items
        USING (tenant_id = current_tenant_id());
      CREATE POLICY tenant_isolation_financial_transactions ON financial_transactions
        USING (tenant_id = current_tenant_id());
      CREATE POLICY tenant_isolation_payment_reminders ON payment_reminders
        USING (tenant_id = current_tenant_id());
    `);
    
    // Create triggers to automatically set tenant_id
    await queryRunner.query(`
      CREATE OR REPLACE FUNCTION set_tenant_id() RETURNS TRIGGER AS $$
      BEGIN
        NEW.tenant_id = current_tenant_id();
        RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
      
      CREATE TRIGGER set_tenant_id_payments
        BEFORE INSERT ON payments
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
      CREATE TRIGGER set_tenant_id_expenses
        BEFORE INSERT ON expenses
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
      CREATE TRIGGER set_tenant_id_invoices
        BEFORE INSERT ON invoices
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
      CREATE TRIGGER set_tenant_id_invoice_items
        BEFORE INSERT ON invoice_items
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
      CREATE TRIGGER set_tenant_id_financial_transactions
        BEFORE INSERT ON financial_transactions
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
      CREATE TRIGGER set_tenant_id_payment_reminders
        BEFORE INSERT ON payment_reminders
        FOR EACH ROW EXECUTE FUNCTION set_tenant_id();
    `);
  }

  async down(queryRunner) {
    // Disable Row-Level Security
    await queryRunner.query(`
      DROP POLICY tenant_isolation_payments ON payments;
      DROP POLICY tenant_isolation_expenses ON expenses;
      DROP POLICY tenant_isolation_invoices ON invoices;
      DROP POLICY tenant_isolation_invoice_items ON invoice_items;
      DROP POLICY tenant_isolation_financial_transactions ON financial_transactions;
      DROP POLICY tenant_isolation_payment_reminders ON payment_reminders;
      
      ALTER TABLE payments DISABLE ROW LEVEL SECURITY;
      ALTER TABLE expenses DISABLE ROW LEVEL SECURITY;
      ALTER TABLE invoices DISABLE ROW LEVEL SECURITY;
      ALTER TABLE invoice_items DISABLE ROW LEVEL SECURITY;
      ALTER TABLE financial_transactions DISABLE ROW LEVEL SECURITY;
      ALTER TABLE payment_reminders DISABLE ROW LEVEL SECURITY;
    `);
    
    // Drop triggers
    await queryRunner.query(`
      DROP TRIGGER set_tenant_id_payments ON payments;
      DROP TRIGGER set_tenant_id_expenses ON expenses;
      DROP TRIGGER set_tenant_id_invoices ON invoices;
      DROP TRIGGER set_tenant_id_invoice_items ON invoice_items;
      DROP TRIGGER set_tenant_id_financial_transactions ON financial_transactions;
      DROP TRIGGER set_tenant_id_payment_reminders ON payment_reminders;
      
      DROP FUNCTION set_tenant_id();
      DROP FUNCTION current_tenant_id();
    `);
    
    // Remove tenant ID columns
    await queryRunner.query(`
      ALTER TABLE payments DROP COLUMN tenant_id;
      ALTER TABLE expenses DROP COLUMN tenant_id;
      ALTER TABLE invoices DROP COLUMN tenant_id;
      ALTER TABLE invoice_items DROP COLUMN tenant_id;
      ALTER TABLE financial_transactions DROP COLUMN tenant_id;
      ALTER TABLE payment_reminders DROP COLUMN tenant_id;
    `);
  }
}
```

### Tenant Configuration

```typescript
// src/models/mongodb/institution.model.ts
import mongoose from 'mongoose';

const institutionSchema = new mongoose.Schema({
  name: { type: String, required: true },
  subdomain: { type: String, required: true, unique: true },
  logo: { type: String },
  address: {
    street: { type: String },
    city: { type: String },
    state: { type: String },
    country: { type: String },
    postalCode: { type: String },
  },
  contact: {
    email: { type: String, required: true },
    phone: { type: String },
    website: { type: String },
  },
  settings: {
    theme: {
      primaryColor: { type: String, default: '#4f46e5' },
      secondaryColor: { type: String, default: '#f59e0b' },
      fontFamily: { type: String, default: 'Inter' },
    },
    features: {
      inventory: { type: Boolean, default: true },
      finance: { type: Boolean, default: true },
      attendance: { type: Boolean, default: true },
      notifications: { type: Boolean, default: true },
      onlinePayments: { type: Boolean, default: true },
    },
    customFields: {
      student: [{
        name: { type: String, required: true },
        label: { type: String, required: true },
        type: { type: String, enum: ['text', 'number', 'date', 'select', 'checkbox'], required: true },
        options: [{
          value: { type: String },
          label: { type: String },
        }],
        required: { type: Boolean, default: false },
      }],
      course: [{
        name: { type: String, required: true },
        label: { type: String, required: true },
        type: { type: String, enum: ['text', 'number', 'date', 'select', 'checkbox'], required: true },
        options: [{
          value: { type: String },
          label: { type: String },
        }],
        required: { type: Boolean, default: false },
      }],
    },
    payment: {
      currency: { type: String, default: 'USD' },
      paymentGateways: [{
        provider: { type: String, enum: ['stripe', 'razorpay', 'paypal'], required: true },
        apiKey: { type: String },
        apiSecret: { type: String },
        isActive: { type: Boolean, default: false },
      }],
      reminderSettings: {
        enableReminders: { type: Boolean, default: true },
        reminderDays: [{ type: Number }], // Days before due date
        reminderChannels: [{ type: String, enum: ['email', 'sms', 'push'] }],
      },
    },
    notification: {
      email: {
        enabled: { type: Boolean, default: true },
        provider: { type: String, enum: ['smtp', 'sendgrid', 'mailgun'], default: 'smtp' },
        config: {
          host: { type: String },
          port: { type: Number },
          username: { type: String },
          password: { type: String },
          apiKey: { type: String },
        },
        templates: {
          welcome: { type: String },
          paymentReminder: { type: String },
          paymentReceipt: { type: String },
        },
      },
      sms: {
        enabled: { type: Boolean, default: false },
        provider: { type: String, enum: ['twilio', 'messagebird'], default: 'twilio' },
        config: {
          accountSid: { type: String },
          authToken: { type: String },
          phoneNumber: { type: String },
        },
        templates: {
          welcome: { type: String },
          paymentReminder: { type: String },
          paymentReceipt: { type: String },
        },
      },
    },
  },
  subscription: {
    plan: { type: String, enum: ['free', 'basic', 'premium', 'enterprise'], default: 'free' },
    startDate: { type: Date },
    endDate: { type: Date },
    status: { type: String, enum: ['active', 'inactive', 'trial'], default: 'trial' },
    paymentMethod: { type: String },
    billingCycle: { type: String, enum: ['monthly', 'yearly'] },
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

institutionSchema.index({ subdomain: 1 }, { unique: true });

export const InstitutionModel = mongoose.model('Institution', institutionSchema);
```

## Payment Integration

### Payment Gateway Integration

```typescript
// src/services/payment.service.ts
import { Injectable } from '@nestjs/common';
import Stripe from 'stripe';
import Razorpay from 'razorpay';
import { InstitutionModel } from '../models/mongodb/institution.model';
import { PaymentModel } from '../models/postgresql/payment.entity';

@Injectable()
export class PaymentService {
  private stripeClients = new Map<string, Stripe>();
  private razorpayClients = new Map<string, Razorpay>();

  async getPaymentGateway(tenantId: string, provider: 'stripe' | 'razorpay') {
    const institution = await InstitutionModel.findById(tenantId);
    if (!institution) {
      throw new Error('Institution not found');
    }

    const paymentGateway = institution.settings.payment.paymentGateways.find(
      (gateway) => gateway.provider === provider && gateway.isActive
    );

    if (!paymentGateway) {
      throw new Error(`${provider} payment gateway not configured or inactive`);
    }

    if (provider === 'stripe') {
      if (!this.stripeClients.has(tenantId)) {
        this.stripeClients.set(
          tenantId,
          new Stripe(paymentGateway.apiSecret, {
            apiVersion: '2023-10-16',
          })
        );
      }
      return this.stripeClients.get(tenantId);
    } else if (provider === 'razorpay') {
      if (!this.razorpayClients.has(tenantId)) {
        this.razorpayClients.set(
          tenantId,
          new Razorpay({
            key_id: paymentGateway.apiKey,
            key_secret: paymentGateway.apiSecret,
          })
        );
      }
      return this.razorpayClients.get(tenantId);
    }
  }

  async createPaymentIntent(tenantId: string, amount: number, currency: string, metadata: any) {
    const stripe = await this.getPaymentGateway(tenantId, 'stripe');
    
    const paymentIntent = await stripe.paymentIntents.create({
      amount,
      currency,
      metadata,
    });

    return paymentIntent;
  }

  async createRazorpayOrder(tenantId: string, amount: number, currency: string, receipt: string) {
    const razorpay = await this.getPaymentGateway(tenantId, 'razorpay');
    
    const order = await razorpay.orders.create({
      amount,
      currency,
      receipt,
    });

    return order;
  }

  async handleStripeWebhook(event: any) {
    const { type, data } = event;

    switch (type) {
      case 'payment_intent.succeeded':
        await this.handleSuccessfulPayment(data.object);
        break;
      case 'payment_intent.payment_failed':
        await this.handleFailedPayment(data.object);
        break;
      default:
        console.log(`Unhandled event type: ${type}`);
    }
  }

  async handleRazorpayWebhook(event: any) {
    const { event: eventType, payload } = event;

    switch (eventType) {
      case 'payment.authorized':
        await this.handleSuccessfulPayment(payload.payment.entity);
        break;
      case 'payment.failed':
        await this.handleFailedPayment(payload.payment.entity);
        break;
      default:
        console.log(`Unhandled event type: ${eventType}`);
    }
  }

  private async handleSuccessfulPayment(paymentData: any) {
    // Update payment status in database
    // Generate receipt
    // Send notification
  }

  private async handleFailedPayment(paymentData: any) {
    // Update payment status in database
    // Send notification
  }
}
```

### Receipt Generation

```typescript
// src/services/receipt.service.ts
import { Injectable } from '@nestjs/common';
import * as PDFDocument from 'pdfkit';
import * as fs from 'fs';
import * as path from 'path';
import { S3 } from 'aws-sdk';
import { InstitutionModel } from '../models/mongodb/institution.model';
import { PaymentModel } from '../models/postgresql/payment.entity';
import { StudentModel } from '../models/mongodb/student.model';

@Injectable()
export class ReceiptService {
  private s3: S3;

  constructor() {
    this.s3 = new S3({
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      region: process.env.AWS_REGION,
    });
  }

  async generateReceipt(paymentId: string) {
    const payment = await PaymentModel.findById(paymentId);
    if (!payment) {
      throw new Error('Payment not found');
    }

    const institution = await InstitutionModel.findById(payment.tenantId);
    const student = await StudentModel.findById(payment.studentId);

    const doc = new PDFDocument({ margin: 50 });
    const buffers = [];
    doc.on('data', buffers.push.bind(buffers));

    // Add institution logo
    if (institution.logo) {
      doc.image(institution.logo, 50, 45, { width: 100 });
    }

    // Add receipt header
    doc.fontSize(20).text('RECEIPT', { align: 'center' });
    doc.moveDown();

    // Add institution details
    doc.fontSize(12).text(`${institution.name}`, { align: 'left' });
    doc.fontSize(10).text(`${institution.address.street}`);
    doc.text(`${institution.address.city}, ${institution.address.state} ${institution.address.postalCode}`);
    doc.text(`${institution.address.country}`);
    doc.text(`Email: ${institution.contact.email}`);
    doc.text(`Phone: ${institution.contact.phone}`);
    doc.moveDown();

    // Add receipt details
    doc.fontSize(12).text('Receipt Details', { underline: true });
    doc.fontSize(10).text(`Receipt Number: ${payment.receiptNumber}`);
    doc.text(`Date: ${payment.createdAt.toLocaleDateString()}`);
    doc.moveDown();

    // Add student details
    doc.fontSize(12).text('Student Details', { underline: true });
    doc.fontSize(10).text(`Name: ${student.name.first} ${student.name.last}`);
    doc.text(`ID: ${student.studentId}`);
    doc.text(`Email: ${student.email}`);
    doc.moveDown();

    // Add payment details
    doc.fontSize(12).text('Payment Details', { underline: true });
    doc.fontSize(10).text(`Amount: ${payment.amount} ${payment.currency}`);
    doc.text(`Payment Method: ${payment.paymentMethod}`);
    doc.text(`Transaction ID: ${payment.transactionId}`);
    doc.text(`Status: ${payment.status}`);
    doc.moveDown();

    // Add payment items
    doc.fontSize(12).text('Payment Items', { underline: true });
    doc.moveDown();

    // Create a table for payment items
    const items = payment.items;
    const tableTop = doc.y;
    const itemX = 50;
    const descriptionX = 150;
    const amountX = 400;

    doc.fontSize(10).text('Item', itemX, tableTop);
    doc.text('Description', descriptionX, tableTop);
    doc.text('Amount', amountX, tableTop);

    const lineY = doc.y + 5;
    doc.moveTo(itemX, lineY).lineTo(500, lineY).stroke();

    let y = lineY + 10;
    items.forEach((item) => {
      doc.text(item.name, itemX, y);
      doc.text(item.description, descriptionX, y);
      doc.text(`${item.amount} ${payment.currency}`, amountX, y);
      y += 20;
    });

    doc.moveTo(itemX, y).lineTo(500, y).stroke();
    y += 10;
    doc.text('Total:', 350, y);
    doc.text(`${payment.amount} ${payment.currency}`, amountX, y);

    // Add footer
    doc.fontSize(10).text('Thank you for your payment!', { align: 'center' });

    // Finalize the PDF
    doc.end();

    return new Promise<Buffer>((resolve, reject) => {
      doc.on('end', () => {
        const pdfBuffer = Buffer.concat(buffers);
        resolve(pdfBuffer);
      });
      doc.on('error', reject);
    });
  }

  async saveReceiptToS3(pdfBuffer: Buffer, fileName: string) {
    const params = {
      Bucket: process.env.AWS_S3_BUCKET,
      Key: `receipts/${fileName}`,
      Body: pdfBuffer,
      ContentType: 'application/pdf',
    };

    const result = await this.s3.upload(params).promise();
    return result.Location;
  }
}
```

## Notification System

### Email Notifications

```typescript
// src/services/notification.service.ts
import { Injectable } from '@nestjs/common';
import * as nodemailer from 'nodemailer';
import * as SendGrid from '@sendgrid/mail';
import * as Mailgun from 'mailgun-js';
import * as Twilio from 'twilio';
import * as MessageBird from 'messagebird';
import { InstitutionModel } from '../models/mongodb/institution.model';
import { NotificationModel } from '../models/mongodb/notification.model';

@Injectable()
export class NotificationService {
  private emailTransporters = new Map<string, any>();
  private smsClients = new Map<string, any>();

  async getEmailTransporter(tenantId: string) {
    if (this.emailTransporters.has(tenantId)) {
      return this.emailTransporters.get(tenantId);
    }

    const institution = await InstitutionModel.findById(tenantId);
    if (!institution) {
      throw new Error('Institution not found');
    }

    const { email } = institution.settings.notification;
    if (!email.enabled) {
      throw new Error('Email notifications are not enabled');
    }

    let transporter;

    switch (email.provider) {
      case 'smtp':
        transporter = nodemailer.createTransport({
          host: email.config.host,
          port: email.config.port,
          secure: email.config.port === 465,
          auth: {
            user: email.config.username,
            pass: email.config.password,
          },
        });
        break;
      case 'sendgrid':
        SendGrid.setApiKey(email.config.apiKey);
        transporter = SendGrid;
        break;
      case 'mailgun':
        transporter = Mailgun({
          apiKey: email.config.apiKey,
          domain: email.config.domain,
        });
        break;
      default:
        throw new Error(`Unsupported email provider: ${email.provider}`);
    }

    this.emailTransporters.set(tenantId, {
      provider: email.provider,
      transporter,
    });

    return this.emailTransporters.get(tenantId);
  }

  async getSmsClient(tenantId: string) {
    if (this.smsClients.has(tenantId)) {
      return this.smsClients.get(tenantId);
    }

    const institution = await InstitutionModel.findById(tenantId);
    if (!institution) {
      throw new Error('Institution not found');
    }

    const { sms } = institution.settings.notification;
    if (!sms.enabled) {
      throw new Error('SMS notifications are not enabled');
    }

    let client;

    switch (sms.provider) {
      case 'twilio':
        client = Twilio(sms.config.accountSid, sms.config.authToken);
        break;
      case 'messagebird':
        client = MessageBird(sms.config.apiKey);
        break;
      default:
        throw new Error(`Unsupported SMS provider: ${sms.provider}`);
    }

    this.smsClients.set(tenantId, {
      provider: sms.provider,
      client,
      phoneNumber: sms.config.phoneNumber,
    });

    return this.smsClients.get(tenantId);
  }

  async sendEmail(tenantId: string, to: string, subject: string, html: string, text?: string) {
    const { provider, transporter } = await this.getEmailTransporter(tenantId);
    const institution = await InstitutionModel.findById(tenantId);

    const from = `${institution.name} <${institution.contact.email}>`;

    let result;

    switch (provider) {
      case 'smtp':
        result = await transporter.sendMail({
          from,
          to,
          subject,
          html,
          text: text || '',
        });
        break;
      case 'sendgrid':
        result = await transporter.send({
          from,
          to,
          subject,
          html,
          text: text || '',
        });
        break;
      case 'mailgun':
        result = await transporter.messages().send({
          from,
          to,
          subject,
          html,
          text: text || '',
        });
        break;
    }

    // Save notification record
    await NotificationModel.create({
      tenantId,
      recipient: to,
      channel: 'email',
      subject,
      content: html,
      status: 'sent',
      metadata: { provider, result },
    });

    return result;
  }

  async sendSms(tenantId: string, to: string, message: string) {
    const { provider, client, phoneNumber } = await this.getSmsClient(tenantId);

    let result;

    switch (provider) {
      case 'twilio':
        result = await client.messages.create({
          body: message,
          from: phoneNumber,
          to,
        });
        break;
      case 'messagebird':
        result = await new Promise((resolve, reject) => {
          client.messages.create(
            {
              originator: phoneNumber,
              recipients: [to],
              body: message,
            },
            (err, response) => {
              if (err) {
                reject(err);
              } else {
                resolve(response);
              }
            }
          );
        });
        break;
    }

    // Save notification record
    await NotificationModel.create({
      tenantId,
      recipient: to,
      channel: 'sms',
      content: message,
      status: 'sent',
      metadata: { provider, result },
    });

    return result;
  }

  async sendPaymentReminder(tenantId: string, studentId: string, paymentId: string) {
    // Implementation for payment reminder
  }

  async sendPaymentReceipt(tenantId: string, studentId: string, paymentId: string, receiptUrl: string) {
    // Implementation for payment receipt
  }
}
```

### SMS Notifications

Implemented in the NotificationService above.

### Scheduled Reminders

```typescript
// src/jobs/reminder.job.ts
import { Injectable } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { InstitutionModel } from '../models/mongodb/institution.model';
import { PaymentModel } from '../models/postgresql/payment.entity';
import { StudentModel } from '../models/mongodb/student.model';
import { NotificationService } from '../services/notification.service';

@Injectable()
export class ReminderJob {
  constructor(private readonly notificationService: NotificationService) {}

  @Cron('0 9 * * *') // Run every day at 9 AM
  async sendPaymentReminders() {
    const institutions = await InstitutionModel.find({
      'settings.payment.reminderSettings.enableReminders': true,
    });

    for (const institution of institutions) {
      const { reminderDays, reminderChannels } = institution.settings.payment.reminderSettings;

      // Process each reminder day
      for (const days of reminderDays) {
        const dueDate = new Date();
        dueDate.setDate(dueDate.getDate() + days);

        // Find payments due on the calculated date
        const payments = await PaymentModel.find({
          tenantId: institution._id,
          dueDate: {
            $gte: new Date(dueDate.setHours(0, 0, 0, 0)),
            $lt: new Date(dueDate.setHours(23, 59, 59, 999)),
          },
          status: 'pending',
        });

        // Send reminders for each payment
        for (const payment of payments) {
          const student = await StudentModel.findById(payment.studentId);

          if (reminderChannels.includes('email') && student.email) {
            await this.notificationService.sendPaymentReminder(
              institution._id,
              student._id,
              payment._id
            );
          }

          if (reminderChannels.includes('sms') && student.phone) {
            const message = `Dear ${student.name.first}, this is a reminder that your payment of ${payment.amount} ${payment.currency} is due in ${days} days. Please log in to make your payment.`;
            await this.notificationService.sendSms(institution._id, student.phone, message);
          }
        }
      }
    }
  }

  @Cron('0 10 * * *') // Run every day at 10 AM
  async sendOverduePaymentReminders() {
    const institutions = await InstitutionModel.find({
      'settings.payment.reminderSettings.enableReminders': true,
    });

    for (const institution of institutions) {
      const { reminderChannels } = institution.settings.payment.reminderSettings;

      // Find overdue payments
      const payments = await PaymentModel.find({
        tenantId: institution._id,
        dueDate: { $lt: new Date() },
        status: 'pending',
      });

      // Send reminders for each overdue payment
      for (const payment of payments) {
        const student = await StudentModel.findById(payment.studentId);

        if (reminderChannels.includes('email') && student.email) {
          const subject = 'Overdue Payment Reminder';
          const html = `
            <h2>Overdue Payment Reminder</h2>
            <p>Dear ${student.name.first},</p>
            <p>This is a reminder that your payment of ${payment.amount} ${payment.currency} is overdue.</p>
            <p>Please log in to make your payment as soon as possible.</p>
            <p>Thank you,</p>
            <p>${institution.name}</p>
          `;
          await this.notificationService.sendEmail(
            institution._id,
            student.email,
            subject,
            html
          );
        }

        if (reminderChannels.includes('sms') && student.phone) {
          const message = `Dear ${student.name.first}, your payment of ${payment.amount} ${payment.currency} is overdue. Please log in to make your payment as soon as possible.`;
          await this.notificationService.sendSms(institution._id, student.phone, message);
        }
      }
    }
  }
}
```

## Attendance System

### QR Code Generation

```typescript
// src/services/attendance.service.ts
import { Injectable } from '@nestjs/common';
import * as QRCode from 'qrcode';
import * as crypto from 'crypto';
import { CourseModel } from '../models/mongodb/course.model';
import { AttendanceModel } from '../models/mongodb/attendance.model';

@Injectable()
export class AttendanceService {
  async generateSessionQRCode(tenantId: string, courseId: string, sessionId: string) {
    // Generate a unique token for this session
    const token = crypto.randomBytes(32).toString('hex');
    
    // Create or update the attendance session
    await AttendanceModel.findOneAndUpdate(
      {
        tenantId,
        courseId,
        sessionId,
      },
      {
        $set: {
          token,
          expiresAt: new Date(Date.now() + 2 * 60 * 60 * 1000), // 2 hours
          status: 'active',
        },
      },
      { upsert: true }
    );
    
    // Generate QR code with the attendance URL
    const attendanceUrl = `${process.env.FRONTEND_URL}/attendance/${token}`;
    const qrCodeDataUrl = await QRCode.toDataURL(attendanceUrl);
    
    return {
      token,
      qrCodeDataUrl,
      attendanceUrl,
    };
  }

  async markAttendance(token: string, studentId: string) {
    // Find the active attendance session with this token
    const session = await AttendanceModel.findOne({
      token,
      expiresAt: { $gt: new Date() },
      status: 'active',
    });
    
    if (!session) {
      throw new Error('Invalid or expired attendance session');
    }
    
    // Check if student is enrolled in the course
    const course = await CourseModel.findOne({
      _id: session.courseId,
      'students.studentId': studentId,
    });
    
    if (!course) {
      throw new Error('Student not enrolled in this course');
    }
    
    // Mark attendance
    const alreadyMarked = session.attendees.some(
      (attendee) => attendee.studentId.toString() === studentId
    );
    
    if (!alreadyMarked) {
      await AttendanceModel.findByIdAndUpdate(session._id, {
        $push: {
          attendees: {
            studentId,
            markedAt: new Date(),
            method: 'qrCode',
          },
        },
      });
    }
    
    return {
      success: true,
      message: alreadyMarked ? 'Attendance already marked' : 'Attendance marked successfully',
      courseName: course.name,
      sessionDate: session.date,
    };
  }

  async getAttendanceReport(tenantId: string, courseId: string, startDate: Date, endDate: Date) {
    // Implementation for attendance report
  }
}
```

### Attendance Tracking

```typescript
// src/models/mongodb/attendance.model.ts
import mongoose from 'mongoose';
import { tenantSchema, applyTenantFilter } from './base.model';

const attendeeSchema = new mongoose.Schema({
  studentId: { type: mongoose.Schema.Types.ObjectId, ref: 'Student', required: true },
  markedAt: { type: Date, default: Date.now },
  method: { type: String, enum: ['qrCode', 'link', 'manual'], required: true },
  location: {
    latitude: { type: Number },
    longitude: { type: Number },
  },
  device: {
    type: { type: String, enum: ['mobile', 'tablet', 'desktop'] },
    browser: { type: String },
    os: { type: String },
  },
});

const attendanceSchema = new mongoose.Schema({
  ...tenantSchema.obj,
  courseId: { type: mongoose.Schema.Types.ObjectId, ref: 'Course', required: true },
  sessionId: { type: String, required: true },
  date: { type: Date, default: Date.now },
  token: { type: String, required: true },
  expiresAt: { type: Date, required: true },
  status: { type: String, enum: ['active', 'completed', 'cancelled'], default: 'active' },
  attendees: [attendeeSchema],
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

attendanceSchema.index({ tenantId: 1, courseId: 1, sessionId: 1 });
attendanceSchema.index({ token: 1 });
attendanceSchema.index({ expiresAt: 1, status: 1 });

applyTenantFilter(attendanceSchema);

export const AttendanceModel = mongoose.model('Attendance', attendanceSchema);
```

## Inventory Management

### Inventory Tracking

```typescript
// src/models/mongodb/inventory.model.ts
import mongoose from 'mongoose';
import { tenantSchema, applyTenantFilter } from './base.model';

const inventoryItemSchema = new mongoose.Schema({
  ...tenantSchema.obj,
  name: { type: String, required: true },
  description: { type: String },
  category: { type: String, required: true },
  type: { type: String, enum: ['physical', 'digital'], default: 'physical' },
  quantity: { type: Number, default: 0 },
  unit: { type: String, default: 'piece' },
  cost: { type: Number },
  supplier: {
    name: { type: String },
    contactPerson: { type: String },
    email: { type: String },
    phone: { type: String },
    website: { type: String },
  },
  location: { type: String },
  purchaseDate: { type: Date },
  expiryDate: { type: Date },
  status: { type: String, enum: ['available', 'low_stock', 'out_of_stock', 'discontinued'], default: 'available' },
  minStockLevel: { type: Number, default: 5 },
  images: [{ type: String }],
  attachments: [{
    name: { type: String },
    url: { type: String },
    type: { type: String },
  }],
  customFields: { type: mongoose.Schema.Types.Mixed },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

inventoryItemSchema.index({ tenantId: 1, name: 1 });
inventoryItemSchema.index({ tenantId: 1, category: 1 });
inventoryItemSchema.index({ tenantId: 1, status: 1 });

applyTenantFilter(inventoryItemSchema);

export const InventoryItemModel = mongoose.model('InventoryItem', inventoryItemSchema);
```

### Inventory Transactions

```typescript
// src/models/mongodb/inventory-transaction.model.ts
import mongoose from 'mongoose';
import { tenantSchema, applyTenantFilter } from './base.model';

const inventoryTransactionSchema = new mongoose.Schema({
  ...tenantSchema.obj,
  itemId: { type: mongoose.Schema.Types.ObjectId, ref: 'InventoryItem', required: true },
  type: { type: String, enum: ['purchase', 'use', 'adjustment', 'transfer'], required: true },
  quantity: { type: Number, required: true },
  date: { type: Date, default: Date.now },
  notes: { type: String },
  referenceId: { type: String },
  referenceType: { type: String, enum: ['purchase_order', 'course', 'staff', 'other'] },
  location: { type: String },
  cost: { type: Number },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

inventoryTransactionSchema.index({ tenantId: 1, itemId: 1 });
inventoryTransactionSchema.index({ tenantId: 1, type: 1 });
inventoryTransactionSchema.index({ tenantId: 1, date: 1 });

applyTenantFilter(inventoryTransactionSchema);

export const InventoryTransactionModel = mongoose.model('InventoryTransaction', inventoryTransactionSchema);
```

## Scalability and Performance

### Caching Strategy

```typescript
// src/services/cache.service.ts
import { Injectable } from '@nestjs/common';
import * as Redis from 'ioredis';

@Injectable()
export class CacheService {
  private readonly redis: Redis.Redis;
  private readonly defaultTTL = 3600; // 1 hour in seconds

  constructor() {
    this.redis = new Redis.Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT, 10),
      password: process.env.REDIS_PASSWORD,
    });
  }

  async get<T>(key: string): Promise<T | null> {
    const value = await this.redis.get(key);
    if (!value) return null;
    return JSON.parse(value) as T;
  }

  async set<T>(key: string, value: T, ttl: number = this.defaultTTL): Promise<void> {
    await this.redis.set(key, JSON.stringify(value), 'EX', ttl);
  }

  async delete(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async deletePattern(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  async getTenantCacheKey(tenantId: string, key: string): string {
    return `tenant:${tenantId}:${key}`;
  }

  async invalidateTenantCache(tenantId: string): Promise<void> {
    await this.deletePattern(`tenant:${tenantId}:*`);
  }
}
```

### Rate Limiting

```typescript
// src/middleware/rate-limit.middleware.ts
import { Injectable, NestMiddleware, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import * as Redis from 'ioredis';

@Injectable()
export class RateLimitMiddleware implements NestMiddleware {
  private readonly redis: Redis.Redis;
  private readonly windowMs = 60 * 1000; // 1 minute
  private readonly maxRequests = 100; // 100 requests per minute

  constructor() {
    this.redis = new Redis.Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT, 10),
      password: process.env.REDIS_PASSWORD,
    });
  }

  async use(req: Request, res: Response, next: NextFunction) {
    const ip = req.ip;
    const tenantId = req.tenantId || 'public';
    const key = `ratelimit:${tenantId}:${ip}`;

    const current = await this.redis.get(key);
    const currentRequests = current ? parseInt(current, 10) : 0;

    if (currentRequests >= this.maxRequests) {
      throw new HttpException('Too many requests', HttpStatus.TOO_MANY_REQUESTS);
    }

    if (currentRequests === 0) {
      await this.redis.set(key, '1', 'PX', this.windowMs);
    } else {
      await this.redis.incr(key);
    }

    // Set rate limit headers
    res.setHeader('X-RateLimit-Limit', this.maxRequests.toString());
    res.setHeader('X-RateLimit-Remaining', (this.maxRequests - currentRequests - 1).toString());

    next();
  }
}
```

### Horizontal Scaling

```typescript
// src/config/app.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import * as compression from 'compression';
import { ValidationPipe } from '@nestjs/common';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);

  // Security middleware
  app.use(helmet());
  app.use(compression());

  // CORS configuration
  app.enableCors({
    origin: configService.get('CORS_ORIGINS').split(','),
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    credentials: true,
  });

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      transform: true,
      forbidNonWhitelisted: true,
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  // Swagger documentation
  const options = new DocumentBuilder()
    .setTitle('Educational Institution Management API')
    .setDescription('API documentation for the Educational Institution Management SaaS Platform')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, options);
  SwaggerModule.setup('api/docs', app, document);

  // Start the server
  const port = configService.get('PORT', 3000);
  await app.listen(port);
  console.log(`Application is running on: ${await app.getUrl()}`);
}

bootstrap();
```

## Deployment and DevOps

### Docker Configuration

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production image, copy all the files and run node
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodejs

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

USER nodejs

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

### Kubernetes Configuration

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: ${DOCKER_REGISTRY}/institute-manage-api:${IMAGE_TAG}
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: mongodb-uri
        - name: POSTGRES_HOST
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: postgres-host
        - name: POSTGRES_PORT
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: postgres-port
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: postgres-user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: postgres-password
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: postgres-db
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: redis-host
        - name: REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: redis-port
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: redis-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-secret
        - name: JWT_REFRESH_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-refresh-secret
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: aws-access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: aws-secret-access-key
        - name: AWS_REGION
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: aws-region
        - name: AWS_S3_BUCKET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: aws-s3-bucket
        - name: CORS_ORIGINS
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: cors-origins
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

## Conclusion

This backend architecture provides a solid foundation for building a scalable, maintainable, and secure educational institution management SaaS platform. The combination of Node.js, Express.js/NestJS, TypeScript, MongoDB, and PostgreSQL offers a flexible and powerful development experience, while the multi-tenant design ensures efficient resource utilization and data isolation.

The architecture is designed with scalability in mind, using caching, rate limiting, and horizontal scaling to handle increasing loads. Security is a core consideration, with robust authentication, authorization, and data protection mechanisms in place.

The modular structure of the codebase promotes maintainability and extensibility, allowing for easy addition of new features and customization for different educational institutions. The comprehensive API design, with proper documentation and versioning, ensures a smooth integration with the frontend and third-party systems.

By following this architecture, developers can build a robust and feature-rich backend for the educational institution management SaaS platform that meets the needs of diverse educational institutions while providing a secure, scalable, and performant experience for all users.