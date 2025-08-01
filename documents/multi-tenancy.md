# Multi-Tenancy Implementation

This document outlines the multi-tenancy implementation strategy for the Educational Institution Management SaaS platform.

## Multi-Tenancy Overview

Multi-tenancy is a core architectural principle of our SaaS platform, allowing multiple educational institutions (tenants) to use the same application instance while maintaining complete data isolation and customization capabilities.

### Multi-Tenancy Models

There are several approaches to implementing multi-tenancy in a SaaS application:

1. **Database-per-tenant**: Each tenant has its own dedicated database
2. **Schema-per-tenant**: Each tenant has its own schema within a shared database
3. **Shared schema with tenant identifier**: All tenants share the same database and schema, with a tenant identifier column for data isolation

For our Educational Institution Management platform, we'll implement a **hybrid approach**:

- **MongoDB**: Shared database with tenant identifier for flexible document collections
- **PostgreSQL**: Schema-per-tenant for transactional data with Row-Level Security (RLS) as an additional security layer

## Tenant Identification

### Tenant Resolution

Tenants will be identified through one of the following methods:

1. **Subdomain-based**: Each tenant gets a unique subdomain (e.g., `school1.institute-manage.com`)
2. **Custom domain**: Premium tenants can use their own domain (e.g., `portal.school1.edu`)
3. **Path-based**: For specific scenarios, tenants can be identified via URL path (e.g., `/t/school1/dashboard`)

```typescript
// src/middleware/tenant-resolver.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { TenantService } from '../services/tenant.service';

@Injectable()
export class TenantResolverMiddleware implements NestMiddleware {
  constructor(private readonly tenantService: TenantService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    let tenantIdentifier: string | null = null;
    
    // 1. Check for tenant in subdomain
    const hostname = req.hostname;
    const subdomainMatch = hostname.match(/^([^.]+)\.institute-manage\.com$/);
    
    if (subdomainMatch) {
      tenantIdentifier = subdomainMatch[1];
    }
    
    // 2. Check for custom domain mapping
    if (!tenantIdentifier) {
      const tenant = await this.tenantService.findByCustomDomain(hostname);
      if (tenant) {
        tenantIdentifier = tenant.identifier;
      }
    }
    
    // 3. Check for tenant in path
    if (!tenantIdentifier) {
      const pathMatch = req.path.match(/^\/t\/([^\/]+)/);
      if (pathMatch) {
        tenantIdentifier = pathMatch[1];
      }
    }
    
    // 4. Check for tenant in header (for API calls)
    if (!tenantIdentifier) {
      tenantIdentifier = req.header('X-Tenant-ID');
    }
    
    if (tenantIdentifier) {
      // Validate tenant exists and is active
      const tenant = await this.tenantService.findByIdentifier(tenantIdentifier);
      
      if (tenant && tenant.status === 'active') {
        // Attach tenant info to request object for use in controllers/services
        req.tenantId = tenant.id;
        req.tenantIdentifier = tenant.identifier;
        req.tenantConfig = tenant.configuration;
      } else {
        // Handle invalid or inactive tenant
        return res.status(404).json({ message: 'Tenant not found or inactive' });
      }
    }
    
    next();
  }
}
```

### Tenant Context

The tenant context will be maintained throughout the request lifecycle:

```typescript
// src/utils/tenant-context.ts
import { AsyncLocalStorage } from 'async_hooks';

interface TenantContext {
  tenantId: string;
  tenantIdentifier: string;
  tenantConfig: Record<string, any>;
}

export class TenantContextHolder {
  private static readonly storage = new AsyncLocalStorage<TenantContext>();

  static get(): TenantContext | undefined {
    return this.storage.getStore();
  }

  static run(context: TenantContext, callback: () => Promise<any>): Promise<any> {
    return this.storage.run(context, callback);
  }

  static getTenantId(): string | undefined {
    return this.get()?.tenantId;
  }

  static getTenantIdentifier(): string | undefined {
    return this.get()?.tenantIdentifier;
  }

  static getTenantConfig(): Record<string, any> | undefined {
    return this.get()?.tenantConfig;
  }
}
```

## Data Isolation

### MongoDB Multi-Tenancy

For MongoDB collections, we'll implement a tenant filter pattern:

```typescript
// src/models/mongodb/base.model.ts
import mongoose, { Schema, Document } from 'mongoose';
import { TenantContextHolder } from '../../utils/tenant-context';

// Base schema with tenant ID
export const tenantSchema = new Schema({
  tenantId: { type: String, required: true, index: true },
});

// Apply tenant filter to all queries
export function applyTenantFilter(schema: Schema) {
  // Add tenant filter to all find queries
  schema.pre(['find', 'findOne', 'findOneAndUpdate', 'findOneAndDelete', 'count', 'countDocuments'], function(next) {
    const tenantId = TenantContextHolder.getTenantId();
    
    if (tenantId) {
      if (!this.getQuery().tenantId) {
        this.where({ tenantId });
      }
    }
    
    next();
  });
  
  // Add tenant ID to all new documents
  schema.pre('save', function(next) {
    const tenantId = TenantContextHolder.getTenantId();
    
    if (tenantId && !this.tenantId) {
      this.tenantId = tenantId;
    }
    
    next();
  });
}

// Example usage in a model
const studentSchema = new Schema({
  ...tenantSchema.obj,  // Include tenant fields
  name: String,
  email: String,
  // other fields
});

applyTenantFilter(studentSchema);

export const Student = mongoose.model('Student', studentSchema);
```

### PostgreSQL Multi-Tenancy

For PostgreSQL, we'll use a schema-per-tenant approach with Row-Level Security:

```typescript
// src/services/database.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { Pool, PoolClient } from 'pg';
import { ConfigService } from '@nestjs/config';
import { TenantContextHolder } from '../utils/tenant-context';

@Injectable()
export class DatabaseService implements OnModuleInit {
  private pool: Pool;

  constructor(private configService: ConfigService) {
    this.pool = new Pool({
      host: this.configService.get('POSTGRES_HOST'),
      port: this.configService.get('POSTGRES_PORT'),
      user: this.configService.get('POSTGRES_USER'),
      password: this.configService.get('POSTGRES_PASSWORD'),
      database: this.configService.get('POSTGRES_DB'),
    });
  }

  async onModuleInit() {
    // Create extension for UUID generation if not exists
    await this.pool.query('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"');
    
    // Create tenant management schema and tables
    await this.pool.query(`
      CREATE SCHEMA IF NOT EXISTS management;
      
      CREATE TABLE IF NOT EXISTS management.tenants (
        id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
        identifier VARCHAR(50) UNIQUE NOT NULL,
        name VARCHAR(100) NOT NULL,
        schema_name VARCHAR(50) UNIQUE NOT NULL,
        status VARCHAR(20) NOT NULL DEFAULT 'active',
        created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
      );
    `);
  }

  async getTenantConnection(): Promise<PoolClient> {
    const client = await this.pool.connect();
    const tenantId = TenantContextHolder.getTenantId();
    
    if (tenantId) {
      // Get tenant schema name
      const result = await client.query(
        'SELECT schema_name FROM management.tenants WHERE id = $1',
        [tenantId]
      );
      
      if (result.rows.length > 0) {
        const schemaName = result.rows[0].schema_name;
        
        // Set search path to tenant schema
        await client.query(`SET search_path TO ${schemaName}, public`);
        
        // Set RLS context
        await client.query(`SET LOCAL "app.tenant_id" = '${tenantId}'`);
      }
    }
    
    return client;
  }

  async createTenantSchema(tenantId: string, schemaName: string): Promise<void> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Create schema for tenant
      await client.query(`CREATE SCHEMA IF NOT EXISTS ${schemaName}`);
      
      // Create tables in tenant schema
      await client.query(`
        SET search_path TO ${schemaName};
        
        CREATE TABLE IF NOT EXISTS payments (
          id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
          amount DECIMAL(10, 2) NOT NULL,
          currency VARCHAR(3) NOT NULL DEFAULT 'USD',
          status VARCHAR(20) NOT NULL,
          payment_method VARCHAR(50),
          payment_gateway VARCHAR(50),
          gateway_payment_id VARCHAR(100),
          student_id UUID NOT NULL,
          course_id UUID,
          description TEXT,
          metadata JSONB,
          created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
          updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
        );
        
        -- Enable Row Level Security
        ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
        
        -- Create policy for tenant isolation
        CREATE POLICY tenant_isolation_policy ON payments
          USING (current_setting('app.tenant_id')::UUID = '${tenantId}');
        
        -- Repeat for other tables...
      `);
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```

## Tenant Provisioning and Management

### Tenant Onboarding

```typescript
// src/services/tenant.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from './database.service';
import { ConfigService } from '@nestjs/config';
import { v4 as uuidv4 } from 'uuid';

interface CreateTenantDto {
  name: string;
  identifier: string;
  adminEmail: string;
  adminPassword: string;
  plan: 'basic' | 'premium' | 'enterprise';
  customDomain?: string;
}

@Injectable()
export class TenantService {
  constructor(
    private databaseService: DatabaseService,
    private configService: ConfigService,
  ) {}

  async createTenant(dto: CreateTenantDto): Promise<any> {
    const client = await this.databaseService.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Generate tenant ID and schema name
      const tenantId = uuidv4();
      const schemaName = `tenant_${dto.identifier}`;
      
      // Create tenant record
      const result = await client.query(
        `INSERT INTO management.tenants 
         (id, identifier, name, schema_name, status) 
         VALUES ($1, $2, $3, $4, 'active') 
         RETURNING *`,
        [tenantId, dto.identifier, dto.name, schemaName]
      );
      
      const tenant = result.rows[0];
      
      // Create tenant schema and tables
      await this.databaseService.createTenantSchema(tenantId, schemaName);
      
      // Create initial admin user for tenant
      // This would typically be done in MongoDB or your user management system
      
      // Create default configuration for tenant
      const defaultConfig = {
        plan: dto.plan,
        features: this.getFeaturesByPlan(dto.plan),
        branding: {
          primaryColor: '#3B82F6',
          secondaryColor: '#1E40AF',
          logo: null,
          favicon: null,
        },
        customDomain: dto.customDomain || null,
      };
      
      // Store tenant configuration
      await client.query(
        `INSERT INTO management.tenant_configurations 
         (tenant_id, configuration) 
         VALUES ($1, $2)`,
        [tenantId, JSON.stringify(defaultConfig)]
      );
      
      await client.query('COMMIT');
      
      return tenant;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private getFeaturesByPlan(plan: string): Record<string, boolean> {
    const features = {
      studentManagement: true,
      courseManagement: true,
      attendanceTracking: true,
      basicReporting: true,
      paymentProcessing: true,
      inventoryManagement: false,
      advancedReporting: false,
      apiAccess: false,
      whiteLabeling: false,
      multipleAdmins: false,
      customDomain: false,
      prioritySupport: false,
    };
    
    if (plan === 'premium' || plan === 'enterprise') {
      features.inventoryManagement = true;
      features.advancedReporting = true;
      features.apiAccess = true;
      features.multipleAdmins = true;
    }
    
    if (plan === 'enterprise') {
      features.whiteLabeling = true;
      features.customDomain = true;
      features.prioritySupport = true;
    }
    
    return features;
  }

  async findByIdentifier(identifier: string): Promise<any> {
    const result = await this.databaseService.pool.query(
      `SELECT t.*, tc.configuration 
       FROM management.tenants t 
       LEFT JOIN management.tenant_configurations tc ON t.id = tc.tenant_id 
       WHERE t.identifier = $1`,
      [identifier]
    );
    
    return result.rows[0] || null;
  }

  async findByCustomDomain(domain: string): Promise<any> {
    const result = await this.databaseService.pool.query(
      `SELECT t.*, tc.configuration 
       FROM management.tenants t 
       LEFT JOIN management.tenant_configurations tc ON t.id = tc.tenant_id 
       WHERE tc.configuration->>'customDomain' = $1`,
      [domain]
    );
    
    return result.rows[0] || null;
  }
}
```

## Tenant Configuration and Customization

### Configuration Management

```typescript
// src/services/tenant-config.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from './database.service';
import { TenantContextHolder } from '../utils/tenant-context';

@Injectable()
export class TenantConfigService {
  constructor(private databaseService: DatabaseService) {}

  async getConfiguration(): Promise<Record<string, any>> {
    const tenantId = TenantContextHolder.getTenantId();
    if (!tenantId) {
      return {};
    }
    
    // First check if config is in context
    const contextConfig = TenantContextHolder.getTenantConfig();
    if (contextConfig) {
      return contextConfig;
    }
    
    // Otherwise fetch from database
    const result = await this.databaseService.pool.query(
      `SELECT configuration 
       FROM management.tenant_configurations 
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return result.rows[0]?.configuration || {};
  }

  async updateConfiguration(config: Record<string, any>): Promise<void> {
    const tenantId = TenantContextHolder.getTenantId();
    if (!tenantId) {
      throw new Error('No tenant context found');
    }
    
    // Get current configuration
    const currentConfig = await this.getConfiguration();
    
    // Merge with new configuration
    const updatedConfig = {
      ...currentConfig,
      ...config,
    };
    
    // Update in database
    await this.databaseService.pool.query(
      `UPDATE management.tenant_configurations 
       SET configuration = $1, updated_at = NOW() 
       WHERE tenant_id = $2`,
      [JSON.stringify(updatedConfig), tenantId]
    );
  }

  async getFeature(featureName: string): Promise<boolean> {
    const config = await this.getConfiguration();
    return config.features?.[featureName] || false;
  }

  async getBranding(): Promise<Record<string, any>> {
    const config = await this.getConfiguration();
    return config.branding || {
      primaryColor: '#3B82F6',
      secondaryColor: '#1E40AF',
      logo: null,
      favicon: null,
    };
  }
}
```

### Frontend Customization

```typescript
// frontend/src/hooks/useTenantConfig.ts
import { useEffect, useState } from 'react';
import { useQuery } from 'react-query';
import { api } from '../services/api';

export function useTenantConfig() {
  const { data, isLoading, error } = useQuery(
    'tenantConfig',
    () => api.get('/api/tenant/config').then(res => res.data),
    { staleTime: 1000 * 60 * 5 } // Cache for 5 minutes
  );

  useEffect(() => {
    if (data?.branding) {
      // Apply tenant branding to CSS variables
      document.documentElement.style.setProperty('--primary-color', data.branding.primaryColor);
      document.documentElement.style.setProperty('--secondary-color', data.branding.secondaryColor);
      
      // Update favicon if custom one is provided
      if (data.branding.favicon) {
        const link = document.querySelector('link[rel="shortcut icon"]') as HTMLLinkElement;
        if (link) {
          link.href = data.branding.favicon;
        }
      }
      
      // Update document title
      if (data.name) {
        document.title = `${data.name} Portal`;
      }
    }
  }, [data]);

  return {
    config: data,
    isLoading,
    error,
    features: data?.features || {},
    branding: data?.branding || {},
    plan: data?.plan || 'basic',
    hasFeature: (featureName: string) => data?.features?.[featureName] || false,
  };
}
```

## Resource Isolation and Scaling

### Resource Quotas and Limits

```typescript
// src/middleware/resource-limits.middleware.ts
import { Injectable, NestMiddleware, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { TenantConfigService } from '../services/tenant-config.service';
import { TenantUsageService } from '../services/tenant-usage.service';

@Injectable()
export class ResourceLimitsMiddleware implements NestMiddleware {
  constructor(
    private tenantConfigService: TenantConfigService,
    private tenantUsageService: TenantUsageService,
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const tenantId = req.tenantId;
    
    if (!tenantId) {
      return next();
    }
    
    // Get tenant configuration
    const config = await this.tenantConfigService.getConfiguration();
    const plan = config.plan || 'basic';
    
    // Get resource limits based on plan
    const limits = this.getResourceLimits(plan);
    
    // Check if tenant has exceeded storage limits
    const storageUsage = await this.tenantUsageService.getStorageUsage(tenantId);
    if (storageUsage > limits.storageLimit) {
      throw new HttpException(
        'Storage limit exceeded. Please upgrade your plan.',
        HttpStatus.PAYMENT_REQUIRED
      );
    }
    
    // Check if tenant has exceeded API rate limits
    if (req.path.startsWith('/api/')) {
      const apiUsage = await this.tenantUsageService.getApiUsage(tenantId);
      if (apiUsage > limits.apiRequestsPerDay) {
        throw new HttpException(
          'API request limit exceeded. Please upgrade your plan.',
          HttpStatus.TOO_MANY_REQUESTS
        );
      }
      
      // Increment API usage counter
      await this.tenantUsageService.incrementApiUsage(tenantId);
    }
    
    // Check if tenant has exceeded user limits
    if (req.method === 'POST' && req.path === '/api/users') {
      const userCount = await this.tenantUsageService.getUserCount(tenantId);
      if (userCount >= limits.maxUsers) {
        throw new HttpException(
          'User limit exceeded. Please upgrade your plan.',
          HttpStatus.PAYMENT_REQUIRED
        );
      }
    }
    
    next();
  }

  private getResourceLimits(plan: string): Record<string, number> {
    switch (plan) {
      case 'enterprise':
        return {
          storageLimit: 100 * 1024 * 1024 * 1024, // 100 GB
          apiRequestsPerDay: 100000,
          maxUsers: 1000,
          maxConcurrentUsers: 500,
        };
      case 'premium':
        return {
          storageLimit: 20 * 1024 * 1024 * 1024, // 20 GB
          apiRequestsPerDay: 20000,
          maxUsers: 200,
          maxConcurrentUsers: 100,
        };
      case 'basic':
      default:
        return {
          storageLimit: 5 * 1024 * 1024 * 1024, // 5 GB
          apiRequestsPerDay: 5000,
          maxUsers: 50,
          maxConcurrentUsers: 25,
        };
    }
  }
}
```

### Tenant Usage Tracking

```typescript
// src/services/tenant-usage.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from './database.service';
import * as Redis from 'ioredis';

@Injectable()
export class TenantUsageService {
  private readonly redis: Redis.Redis;

  constructor(private databaseService: DatabaseService) {
    this.redis = new Redis.Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT, 10),
      password: process.env.REDIS_PASSWORD,
    });
  }

  async getStorageUsage(tenantId: string): Promise<number> {
    const result = await this.databaseService.pool.query(
      `SELECT SUM(size_bytes) as total_size 
       FROM management.tenant_storage_usage 
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0]?.total_size || '0', 10);
  }

  async getApiUsage(tenantId: string): Promise<number> {
    const today = new Date().toISOString().split('T')[0];
    const key = `tenant:${tenantId}:api_usage:${today}`;
    
    const usage = await this.redis.get(key);
    return parseInt(usage || '0', 10);
  }

  async incrementApiUsage(tenantId: string): Promise<void> {
    const today = new Date().toISOString().split('T')[0];
    const key = `tenant:${tenantId}:api_usage:${today}`;
    
    // Increment counter and set 24-hour expiry
    await this.redis.incr(key);
    await this.redis.expire(key, 24 * 60 * 60);
    
    // Also store in database for long-term analytics
    await this.databaseService.pool.query(
      `INSERT INTO management.tenant_api_usage 
       (tenant_id, usage_date, request_count) 
       VALUES ($1, $2, 1) 
       ON CONFLICT (tenant_id, usage_date) 
       DO UPDATE SET request_count = tenant_api_usage.request_count + 1`,
      [tenantId, today]
    );
  }

  async getUserCount(tenantId: string): Promise<number> {
    // This would typically query your user management system
    // For MongoDB example:
    /*
    const count = await this.userModel.countDocuments({ tenantId });
    return count;
    */
    
    // For demonstration purposes:
    const result = await this.databaseService.pool.query(
      `SELECT COUNT(*) as user_count 
       FROM management.tenant_users 
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0]?.user_count || '0', 10);
  }

  async trackStorageUsage(tenantId: string, bytes: number, operation: 'add' | 'remove'): Promise<void> {
    if (operation === 'add') {
      await this.databaseService.pool.query(
        `INSERT INTO management.tenant_storage_usage 
         (tenant_id, size_bytes, updated_at) 
         VALUES ($1, $2, NOW()) 
         ON CONFLICT (tenant_id) 
         DO UPDATE SET size_bytes = tenant_storage_usage.size_bytes + $2, 
                       updated_at = NOW()`,
        [tenantId, bytes]
      );
    } else {
      await this.databaseService.pool.query(
        `UPDATE management.tenant_storage_usage 
         SET size_bytes = GREATEST(0, size_bytes - $2), 
             updated_at = NOW() 
         WHERE tenant_id = $1`,
        [tenantId, bytes]
      );
    }
  }
}
```

## Tenant Billing and Subscription Management

### Subscription Management

```typescript
// src/services/subscription.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from './database.service';
import { StripeService } from './stripe.service';

interface CreateSubscriptionDto {
  tenantId: string;
  plan: 'basic' | 'premium' | 'enterprise';
  paymentMethodId: string;
  billingEmail: string;
  billingName: string;
  billingAddress: Record<string, any>;
}

@Injectable()
export class SubscriptionService {
  constructor(
    private databaseService: DatabaseService,
    private stripeService: StripeService,
  ) {}

  async createSubscription(dto: CreateSubscriptionDto): Promise<any> {
    // Get plan price ID from configuration
    const planPriceId = this.getPlanPriceId(dto.plan);
    
    // Create or get Stripe customer
    const customer = await this.stripeService.createCustomer({
      email: dto.billingEmail,
      name: dto.billingName,
      address: dto.billingAddress,
      metadata: { tenantId: dto.tenantId },
    });
    
    // Attach payment method to customer
    await this.stripeService.attachPaymentMethod(customer.id, dto.paymentMethodId);
    
    // Create subscription
    const subscription = await this.stripeService.createSubscription({
      customerId: customer.id,
      priceId: planPriceId,
      metadata: { tenantId: dto.tenantId },
    });
    
    // Store subscription in database
    const result = await this.databaseService.pool.query(
      `INSERT INTO management.tenant_subscriptions 
       (tenant_id, stripe_customer_id, stripe_subscription_id, plan, status, 
        current_period_start, current_period_end, created_at) 
       VALUES ($1, $2, $3, $4, $5, to_timestamp($6), to_timestamp($7), NOW()) 
       RETURNING *`,
      [
        dto.tenantId,
        customer.id,
        subscription.id,
        dto.plan,
        subscription.status,
        subscription.current_period_start,
        subscription.current_period_end,
      ]
    );
    
    // Update tenant configuration with new plan
    await this.databaseService.pool.query(
      `UPDATE management.tenant_configurations 
       SET configuration = jsonb_set(configuration, '{plan}', '"${dto.plan}"'), 
           configuration = jsonb_set(configuration, '{features}', '${JSON.stringify(this.getFeaturesByPlan(dto.plan))}') 
       WHERE tenant_id = $1`,
      [dto.tenantId]
    );
    
    return result.rows[0];
  }

  async cancelSubscription(tenantId: string): Promise<void> {
    // Get subscription from database
    const result = await this.databaseService.pool.query(
      `SELECT stripe_subscription_id 
       FROM management.tenant_subscriptions 
       WHERE tenant_id = $1 AND status NOT IN ('canceled', 'incomplete_expired')`,
      [tenantId]
    );
    
    if (result.rows.length === 0) {
      throw new Error('No active subscription found');
    }
    
    const subscriptionId = result.rows[0].stripe_subscription_id;
    
    // Cancel subscription in Stripe
    await this.stripeService.cancelSubscription(subscriptionId);
    
    // Update subscription status in database
    await this.databaseService.pool.query(
      `UPDATE management.tenant_subscriptions 
       SET status = 'canceled', updated_at = NOW() 
       WHERE stripe_subscription_id = $1`,
      [subscriptionId]
    );
  }

  async handleSubscriptionUpdated(stripeEvent: any): Promise<void> {
    const subscription = stripeEvent.data.object;
    
    // Update subscription in database
    await this.databaseService.pool.query(
      `UPDATE management.tenant_subscriptions 
       SET status = $1, 
           current_period_start = to_timestamp($2), 
           current_period_end = to_timestamp($3), 
           updated_at = NOW() 
       WHERE stripe_subscription_id = $4`,
      [
        subscription.status,
        subscription.current_period_start,
        subscription.current_period_end,
        subscription.id,
      ]
    );
    
    // If subscription is canceled or past due, update tenant status
    if (['canceled', 'unpaid', 'past_due'].includes(subscription.status)) {
      // Get tenant ID from subscription metadata or database
      const tenantId = subscription.metadata.tenantId || (
        await this.databaseService.pool.query(
          `SELECT tenant_id FROM management.tenant_subscriptions WHERE stripe_subscription_id = $1`,
          [subscription.id]
        )
      ).rows[0]?.tenant_id;
      
      if (tenantId) {
        // Update tenant status based on subscription status
        const tenantStatus = subscription.status === 'canceled' ? 'inactive' : 'limited';
        
        await this.databaseService.pool.query(
          `UPDATE management.tenants 
           SET status = $1, updated_at = NOW() 
           WHERE id = $2`,
          [tenantStatus, tenantId]
        );
      }
    }
  }

  private getPlanPriceId(plan: string): string {
    // These would be configured in your environment or database
    const priceIds = {
      basic: process.env.STRIPE_BASIC_PRICE_ID,
      premium: process.env.STRIPE_PREMIUM_PRICE_ID,
      enterprise: process.env.STRIPE_ENTERPRISE_PRICE_ID,
    };
    
    return priceIds[plan] || priceIds.basic;
  }

  private getFeaturesByPlan(plan: string): Record<string, boolean> {
    // Same implementation as in TenantService
    const features = {
      studentManagement: true,
      courseManagement: true,
      attendanceTracking: true,
      basicReporting: true,
      paymentProcessing: true,
      inventoryManagement: false,
      advancedReporting: false,
      apiAccess: false,
      whiteLabeling: false,
      multipleAdmins: false,
      customDomain: false,
      prioritySupport: false,
    };
    
    if (plan === 'premium' || plan === 'enterprise') {
      features.inventoryManagement = true;
      features.advancedReporting = true;
      features.apiAccess = true;
      features.multipleAdmins = true;
    }
    
    if (plan === 'enterprise') {
      features.whiteLabeling = true;
      features.customDomain = true;
      features.prioritySupport = true;
    }
    
    return features;
  }
}
```

## Conclusion

This multi-tenancy implementation provides a robust foundation for the Educational Institution Management SaaS platform, ensuring:

1. **Complete Data Isolation**: Each tenant's data is securely isolated from other tenants
2. **Flexible Customization**: Tenants can customize their experience with branding and feature sets
3. **Efficient Resource Utilization**: Shared infrastructure with appropriate resource limits per tenant
4. **Scalable Architecture**: The system can scale to handle thousands of tenants
5. **Secure Access Control**: Proper authentication and authorization at both application and database levels

The hybrid approach using MongoDB with tenant filtering and PostgreSQL with schema-per-tenant provides the best balance of flexibility and performance for different data types. The tenant context management ensures that tenant identification is maintained throughout the request lifecycle, while the resource limits and usage tracking prevent any single tenant from consuming excessive resources.

By implementing this multi-tenancy strategy, the Educational Institution Management SaaS platform can efficiently serve multiple educational institutions while providing each with a secure, customized, and performant experience.