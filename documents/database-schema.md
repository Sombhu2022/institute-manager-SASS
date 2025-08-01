# Database Schema Design

This document provides a detailed database schema design for the Educational Institution Management SaaS Platform. The system uses a hybrid database approach with MongoDB for flexible document storage and PostgreSQL for transactional data.

## MongoDB Collections

### Institutions Collection

Stores information about educational institutions using the platform.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "name": String,                // Institution name
  "logo": String,                // URL to institution logo
  "address": {
    "street": String,
    "city": String,
    "state": String,
    "country": String,
    "postalCode": String,
    "coordinates": {
      "latitude": Number,
      "longitude": Number
    }
  },
  "contactEmail": String,        // Primary contact email
  "contactPhone": String,        // Primary contact phone
  "website": String,             // Institution website
  "customDomain": String,        // Custom domain for white-labeling
  "subscriptionPlan": String,    // Basic, Professional, Enterprise
  "subscriptionStatus": String,  // Active, Inactive, Trial, Expired
  "subscriptionStartDate": Date,
  "subscriptionEndDate": Date,
  "paymentDetails": {
    "gatewayId": String,         // Payment gateway identifier
    "customerId": String,        // Customer ID in payment gateway
    "defaultPaymentMethodId": String
  },
  "settings": {
    "theme": {
      "primaryColor": String,
      "secondaryColor": String,
      "logoPosition": String,
      "fontFamily": String
    },
    "emailTemplates": {
      "welcome": String,
      "paymentReminder": String,
      "paymentReceipt": String,
      "attendanceReport": String
    },
    "smsTemplates": {
      "paymentReminder": String,
      "attendanceAlert": String
    },
    "paymentReminders": {
      "enabled": Boolean,
      "daysBeforeDue": Number,
      "daysAfterDue": [Number],
      "channels": [String]       // Email, SMS, Both
    },
    "attendanceSettings": {
      "qrCodeEnabled": Boolean,
      "linkEnabled": Boolean,
      "manualEnabled": Boolean,
      "qrCodeValidityMinutes": Number
    }
  },
  "users": [{
    "userId": ObjectId,          // Reference to Users collection
    "role": String,              // Admin, Manager, Staff
    "permissions": [String]      // Array of permission codes
  }],
  "createdAt": Date,
  "updatedAt": Date
}
```

### Users Collection

Stores user authentication and profile information.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "email": String,                // User email (unique)
  "passwordHash": String,         // Hashed password
  "salt": String,                 // Password salt
  "firstName": String,
  "lastName": String,
  "phone": String,
  "avatar": String,               // URL to avatar image
  "role": String,                 // Admin, Staff, Student
  "institutionId": ObjectId,      // Reference to institution
  "isActive": Boolean,
  "lastLogin": Date,
  "resetPasswordToken": String,
  "resetPasswordExpires": Date,
  "emailVerified": Boolean,
  "emailVerificationToken": String,
  "twoFactorEnabled": Boolean,
  "twoFactorSecret": String,
  "createdAt": Date,
  "updatedAt": Date
}
```

### Students Collection

Stores detailed information about students.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "userId": ObjectId,             // Reference to Users collection
  "institutionId": ObjectId,      // Reference to institution
  "studentId": String,            // Institution-specific student ID
  "name": {
    "first": String,
    "middle": String,
    "last": String
  },
  "email": String,
  "phone": String,
  "dateOfBirth": Date,
  "gender": String,
  "address": {
    "street": String,
    "city": String,
    "state": String,
    "country": String,
    "postalCode": String
  },
  "guardianInfo": [{
    "name": String,
    "relationship": String,
    "phone": String,
    "email": String,
    "address": String,
    "isPrimary": Boolean
  }],
  "enrollmentDate": Date,
  "status": String,              // Active, Inactive, Graduated, On Leave
  "courses": [{
    "courseId": ObjectId,         // Reference to Courses collection
    "enrollmentDate": Date,
    "status": String,            // Enrolled, Completed, Dropped
    "progress": Number,          // Percentage of completion
    "grade": String,
    "notes": String
  }],
  "documents": [{
    "type": String,              // ID Proof, Address Proof, etc.
    "title": String,
    "url": String,               // File storage URL
    "mimeType": String,
    "size": Number,              // File size in bytes
    "uploadedAt": Date
  }],
  "notes": [{
    "content": String,
    "createdBy": ObjectId,       // Reference to Users collection
    "createdAt": Date
  }],
  "tags": [String],              // Custom tags for filtering
  "customFields": Object,        // Institution-specific custom fields
  "createdAt": Date,
  "updatedAt": Date
}
```

### Courses Collection

Stores information about courses offered by institutions.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "institutionId": ObjectId,      // Reference to institution
  "code": String,                 // Course code
  "name": String,                 // Course name
  "description": String,          // Course description
  "category": String,             // Course category
  "duration": {
    "value": Number,
    "unit": String               // Days, Weeks, Months
  },
  "startDate": Date,
  "endDate": Date,
  "fee": {
    "amount": Number,
    "currency": String,
    "installments": Boolean,
    "installmentDetails": [{
      "dueDate": Date,
      "amount": Number
    }]
  },
  "capacity": Number,             // Maximum number of students
  "enrolledCount": Number,        // Current number of enrolled students
  "schedule": [{
    "day": String,                // Monday, Tuesday, etc.
    "startTime": String,          // HH:MM format
    "endTime": String,            // HH:MM format
    "roomId": ObjectId,           // Reference to Rooms collection
    "instructorId": ObjectId      // Reference to Staff collection
  }],
  "instructors": [ObjectId],      // References to Staff collection
  "syllabus": [{
    "title": String,
    "description": String,
    "order": Number
  }],
  "materials": [{
    "title": String,
    "description": String,
    "type": String,               // Document, Video, Link
    "url": String,                // File storage URL
    "mimeType": String,
    "size": Number,               // File size in bytes
    "uploadedAt": Date
  }],
  "isActive": Boolean,
  "tags": [String],               // Custom tags for filtering
  "customFields": Object,         // Institution-specific custom fields
  "createdAt": Date,
  "updatedAt": Date
}
```

### Staff Collection

Stores information about staff members (teachers, administrators, etc.).

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "userId": ObjectId,             // Reference to Users collection
  "institutionId": ObjectId,      // Reference to institution
  "staffId": String,              // Institution-specific staff ID
  "name": {
    "first": String,
    "middle": String,
    "last": String
  },
  "email": String,
  "phone": String,
  "dateOfBirth": Date,
  "gender": String,
  "address": {
    "street": String,
    "city": String,
    "state": String,
    "country": String,
    "postalCode": String
  },
  "role": String,                 // Teacher, Administrator, etc.
  "department": String,
  "designation": String,
  "qualifications": [{
    "degree": String,
    "institution": String,
    "year": Number,
    "document": String            // URL to certificate
  }],
  "employmentDetails": {
    "type": String,               // Full-time, Part-time, Contract
    "joiningDate": Date,
    "terminationDate": Date,
    "salary": {
      "amount": Number,
      "currency": String,
      "paymentFrequency": String  // Monthly, Bi-weekly, etc.
    }
  },
  "courses": [ObjectId],          // References to Courses collection
  "schedule": [{
    "day": String,                // Monday, Tuesday, etc.
    "startTime": String,          // HH:MM format
    "endTime": String,            // HH:MM format
    "courseId": ObjectId,         // Reference to Courses collection
    "roomId": ObjectId            // Reference to Rooms collection
  }],
  "documents": [{
    "type": String,               // ID Proof, Contract, etc.
    "title": String,
    "url": String,                // File storage URL
    "mimeType": String,
    "size": Number,               // File size in bytes
    "uploadedAt": Date
  }],
  "notes": [{
    "content": String,
    "createdBy": ObjectId,        // Reference to Users collection
    "createdAt": Date
  }],
  "isActive": Boolean,
  "tags": [String],               // Custom tags for filtering
  "customFields": Object,         // Institution-specific custom fields
  "createdAt": Date,
  "updatedAt": Date
}
```

### Inventory Collection

Stores information about inventory items.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "institutionId": ObjectId,      // Reference to institution
  "name": String,                 // Item name
  "description": String,          // Item description
  "category": String,             // Item category
  "subcategory": String,          // Item subcategory
  "sku": String,                  // Stock keeping unit
  "barcode": String,              // Barcode if available
  "quantity": Number,             // Current quantity
  "minQuantity": Number,          // Minimum quantity threshold for alerts
  "unitPrice": {
    "amount": Number,
    "currency": String
  },
  "supplier": {
    "name": String,
    "contactPerson": String,
    "email": String,
    "phone": String,
    "address": String
  },
  "purchaseHistory": [{
    "date": Date,
    "quantity": Number,
    "unitPrice": {
      "amount": Number,
      "currency": String
    },
    "totalPrice": {
      "amount": Number,
      "currency": String
    },
    "invoiceNumber": String,
    "purchasedBy": ObjectId,      // Reference to Users collection
    "notes": String
  }],
  "location": {
    "building": String,
    "room": String,
    "shelf": String
  },
  "status": String,               // Available, Low Stock, Out of Stock
  "isActive": Boolean,
  "images": [String],             // URLs to item images
  "documents": [{
    "type": String,               // Invoice, Manual, etc.
    "title": String,
    "url": String,                // File storage URL
    "uploadedAt": Date
  }],
  "tags": [String],               // Custom tags for filtering
  "customFields": Object,         // Institution-specific custom fields
  "createdAt": Date,
  "updatedAt": Date
}
```

### Attendance Collection

Stores attendance records for students.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "institutionId": ObjectId,      // Reference to institution
  "courseId": ObjectId,           // Reference to Courses collection
  "date": Date,                   // Date of attendance
  "scheduleId": ObjectId,         // Reference to course schedule
  "qrCode": String,               // Generated QR code for attendance
  "qrCodeExpiry": Date,           // Expiry time for QR code
  "attendanceLink": String,       // Unique link for attendance
  "records": [{
    "studentId": ObjectId,         // Reference to Students collection
    "status": String,             // Present, Absent, Late, Excused
    "checkInTime": Date,          // Time of check-in
    "checkOutTime": Date,         // Time of check-out (if applicable)
    "duration": Number,           // Duration in minutes
    "method": String,             // QR, Link, Manual
    "location": {
      "latitude": Number,
      "longitude": Number,
      "accuracy": Number
    },
    "device": String,             // Device information
    "notes": String,
    "markedBy": ObjectId,         // Reference to Users collection (if manually marked)
    "markedAt": Date
  }],
  "summary": {
    "totalStudents": Number,
    "present": Number,
    "absent": Number,
    "late": Number,
    "excused": Number
  },
  "notes": String,
  "createdAt": Date,
  "updatedAt": Date
}
```

### Rooms Collection

Stores information about physical or virtual rooms.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "institutionId": ObjectId,      // Reference to institution
  "name": String,                 // Room name
  "type": String,                 // Classroom, Lab, Virtual, etc.
  "building": String,             // Building name
  "floor": String,                // Floor number
  "capacity": Number,             // Maximum capacity
  "facilities": [String],         // Projector, Whiteboard, etc.
  "isActive": Boolean,
  "createdAt": Date,
  "updatedAt": Date
}
```

### Notifications Collection

Stores notification records.

```javascript
{
  "_id": ObjectId,                // Unique identifier
  "institutionId": ObjectId,      // Reference to institution
  "type": String,                 // Payment Reminder, Attendance Alert, etc.
  "title": String,                // Notification title
  "content": String,              // Notification content
  "recipients": [{
    "userId": ObjectId,           // Reference to Users collection
    "role": String,               // Student, Staff, Admin
    "channels": [String],         // Email, SMS, In-app
    "status": {
      "email": String,            // Sent, Delivered, Failed
      "sms": String,              // Sent, Delivered, Failed
      "inApp": String             // Delivered, Read
    },
    "sentAt": Date,
    "readAt": Date
  }],
  "metadata": Object,             // Additional context-specific data
  "createdBy": ObjectId,          // Reference to Users collection
  "createdAt": Date,
  "updatedAt": Date
}
```

## PostgreSQL Tables

### Payments Table

Stores payment records.

```sql
CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  student_id UUID NOT NULL,
  course_id UUID,
  invoice_id UUID,
  amount DECIMAL(10, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  payment_date TIMESTAMP NOT NULL,
  due_date TIMESTAMP,
  payment_method VARCHAR(50),
  payment_gateway VARCHAR(50),
  transaction_id VARCHAR(100),
  gateway_response JSONB,
  status VARCHAR(20) NOT NULL,  -- Pending, Completed, Failed, Refunded
  receipt_number VARCHAR(50),
  receipt_url VARCHAR(255),
  notes TEXT,
  created_by UUID,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_institution_id ON payments(institution_id);
CREATE INDEX idx_payments_student_id ON payments(student_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_payment_date ON payments(payment_date);
```

### Expenses Table

Stores expense records.

```sql
CREATE TABLE expenses (
  id SERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  category VARCHAR(50) NOT NULL,
  subcategory VARCHAR(50),
  amount DECIMAL(10, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  payment_date TIMESTAMP NOT NULL,
  recipient VARCHAR(100),
  recipient_type VARCHAR(50),  -- Staff, Vendor, Utility, etc.
  description TEXT,
  payment_method VARCHAR(50),
  reference_number VARCHAR(100),
  receipt_url VARCHAR(255),
  recurring BOOLEAN DEFAULT FALSE,
  recurring_frequency VARCHAR(20),  -- Monthly, Quarterly, etc.
  recurring_end_date TIMESTAMP,
  staff_id UUID,  -- If expense is related to staff
  inventory_id UUID,  -- If expense is related to inventory
  created_by UUID,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_expenses_institution_id ON expenses(institution_id);
CREATE INDEX idx_expenses_category ON expenses(category);
CREATE INDEX idx_expenses_payment_date ON expenses(payment_date);
```

### Invoices Table

Stores invoice records.

```sql
CREATE TABLE invoices (
  id SERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  student_id UUID NOT NULL,
  invoice_number VARCHAR(50) NOT NULL,
  issue_date TIMESTAMP NOT NULL,
  due_date TIMESTAMP NOT NULL,
  total_amount DECIMAL(10, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  discount_amount DECIMAL(10, 2) DEFAULT 0,
  tax_amount DECIMAL(10, 2) DEFAULT 0,
  paid_amount DECIMAL(10, 2) DEFAULT 0,
  balance_amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,  -- Draft, Sent, Partially Paid, Paid, Overdue, Cancelled
  payment_terms TEXT,
  notes TEXT,
  pdf_url VARCHAR(255),
  created_by UUID,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_invoices_institution_id ON invoices(institution_id);
CREATE INDEX idx_invoices_student_id ON invoices(student_id);
CREATE INDEX idx_invoices_invoice_number ON invoices(invoice_number);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date);
```

### Invoice Items Table

Stores line items for invoices.

```sql
CREATE TABLE invoice_items (
  id SERIAL PRIMARY KEY,
  invoice_id INTEGER REFERENCES invoices(id) ON DELETE CASCADE,
  item_type VARCHAR(50) NOT NULL,  -- Course Fee, Registration Fee, etc.
  description VARCHAR(255) NOT NULL,
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(10, 2) NOT NULL,
  discount_percentage DECIMAL(5, 2) DEFAULT 0,
  discount_amount DECIMAL(10, 2) DEFAULT 0,
  tax_percentage DECIMAL(5, 2) DEFAULT 0,
  tax_amount DECIMAL(10, 2) DEFAULT 0,
  total_price DECIMAL(10, 2) NOT NULL,
  course_id UUID,  -- If item is related to a course
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_invoice_items_invoice_id ON invoice_items(invoice_id);
```

### Financial Transactions Table

Stores all financial transactions for comprehensive reporting.

```sql
CREATE TABLE financial_transactions (
  id SERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  transaction_type VARCHAR(50) NOT NULL,  -- Payment, Expense, Refund, etc.
  amount DECIMAL(10, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  transaction_date TIMESTAMP NOT NULL,
  description TEXT,
  category VARCHAR(50),
  reference_id UUID,  -- ID of related payment, expense, etc.
  reference_type VARCHAR(50),  -- Payment, Expense, etc.
  payment_method VARCHAR(50),
  transaction_id VARCHAR(100),
  status VARCHAR(20) NOT NULL,
  created_by UUID,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_financial_transactions_institution_id ON financial_transactions(institution_id);
CREATE INDEX idx_financial_transactions_transaction_type ON financial_transactions(transaction_type);
CREATE INDEX idx_financial_transactions_transaction_date ON financial_transactions(transaction_date);
```

### Payment Reminders Table

Stores payment reminder records.

```sql
CREATE TABLE payment_reminders (
  id SERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  invoice_id INTEGER REFERENCES invoices(id) ON DELETE CASCADE,
  student_id UUID NOT NULL,
  reminder_date TIMESTAMP NOT NULL,
  due_date TIMESTAMP NOT NULL,
  amount_due DECIMAL(10, 2) NOT NULL,
  reminder_type VARCHAR(50) NOT NULL,  -- Before Due, After Due
  days_relative_to_due INTEGER NOT NULL,  -- Negative for before, positive for after
  status VARCHAR(20) NOT NULL,  -- Scheduled, Sent, Failed, Cancelled
  notification_id UUID,  -- Reference to Notifications collection
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payment_reminders_institution_id ON payment_reminders(institution_id);
CREATE INDEX idx_payment_reminders_invoice_id ON payment_reminders(invoice_id);
CREATE INDEX idx_payment_reminders_student_id ON payment_reminders(student_id);
CREATE INDEX idx_payment_reminders_reminder_date ON payment_reminders(reminder_date);
CREATE INDEX idx_payment_reminders_status ON payment_reminders(status);
```

## Database Relationships

### MongoDB Relationships

1. **Institution to Users**: One-to-many
   - An institution has many users
   - Each user belongs to one institution

2. **User to Student/Staff**: One-to-one
   - A user can have one student or staff profile
   - Each student/staff profile is linked to one user

3. **Institution to Courses**: One-to-many
   - An institution offers many courses
   - Each course belongs to one institution

4. **Course to Students**: Many-to-many
   - A course can have many students
   - A student can be enrolled in many courses

5. **Course to Staff (Instructors)**: Many-to-many
   - A course can have many instructors
   - An instructor can teach many courses

6. **Institution to Inventory**: One-to-many
   - An institution has many inventory items
   - Each inventory item belongs to one institution

7. **Course to Attendance**: One-to-many
   - A course has many attendance records
   - Each attendance record belongs to one course

### PostgreSQL Relationships

1. **Institution to Payments/Expenses/Invoices**: One-to-many
   - An institution has many financial records
   - Each financial record belongs to one institution

2. **Student to Payments/Invoices**: One-to-many
   - A student has many payments and invoices
   - Each payment/invoice belongs to one student

3. **Invoice to Invoice Items**: One-to-many
   - An invoice has many line items
   - Each line item belongs to one invoice

4. **Invoice to Payments**: One-to-many
   - An invoice can have multiple payments
   - Each payment is associated with one invoice

5. **Invoice to Payment Reminders**: One-to-many
   - An invoice can have multiple payment reminders
   - Each payment reminder is associated with one invoice

## Indexing Strategy

### MongoDB Indexes

1. **Institutions Collection**:
   - `name`: For searching institutions by name
   - `customDomain`: For domain-based routing
   - `subscriptionStatus`: For filtering active/inactive institutions

2. **Users Collection**:
   - `email`: For user lookup during authentication
   - `institutionId`: For filtering users by institution
   - `role`: For role-based queries

3. **Students Collection**:
   - `userId`: For linking to user accounts
   - `institutionId`: For filtering students by institution
   - `studentId`: For institution-specific student ID lookup
   - `courses.courseId`: For finding students enrolled in specific courses 
   - `status`: For filtering active/inactive students

4. **Courses Collection**:
   - `institutionId`: For filtering courses by institution
   - `code`: For course code lookup
   - `instructors`: For finding courses taught by specific instructors
   - `startDate` and `endDate`: For filtering current/upcoming/past courses

5. **Staff Collection**:
   - `userId`: For linking to user accounts
   - `institutionId`: For filtering staff by institution
   - `staffId`: For institution-specific staff ID lookup
   - `role`: For filtering by role (teacher, administrator, etc.)
   - `courses`: For finding staff assigned to specific courses

6. **Inventory Collection**:
   - `institutionId`: For filtering inventory by institution
   - `category` and `subcategory`: For filtering by category
   - `status`: For filtering by availability status
   - `quantity`: For identifying low stock items

7. **Attendance Collection**:
   - `institutionId`: For filtering attendance by institution
   - `courseId`: For filtering attendance by course
   - `date`: For filtering attendance by date
   - `records.studentId`: For finding attendance records for specific students

### PostgreSQL Indexes

Indexes are defined in the table creation scripts above.

## Multi-Tenancy Implementation

### Data Isolation

1. **Application-Level Filtering**:
   - All queries include `institutionId` filter
   - Middleware validates that users can only access their institution's data

2. **Database Connection Pooling**:
   - Connection pools are managed per institution for high-volume tenants
   - Default shared connection pool for standard tenants

3. **Sharding Strategy**:
   - MongoDB collections sharded by `institutionId` for horizontal scaling
   - PostgreSQL tables partitioned by `institution_id` for large institutions

### Tenant Configuration

1. **Institution Settings**:
   - Stored in the Institutions collection
   - Cached for performance optimization

2. **Custom Fields**:
   - Flexible schema in MongoDB allows for institution-specific fields
   - `customFields` object in relevant collections

## Data Migration and Backup Strategy

1. **Regular Backups**:
   - Daily full backups of all databases
   - Hourly incremental backups
   - Point-in-time recovery capability

2. **Data Migration Tools**:
   - Import/export utilities for onboarding existing institutions
   - Schema migration scripts for version upgrades

3. **Tenant Isolation in Backups**:
   - Ability to restore data for a single institution
   - Encrypted backups with tenant-specific keys

## Performance Considerations

1. **Caching Strategy**:
   - Redis cache for frequently accessed data
   - Cache institution settings and configurations
   - Cache course and student lists for dashboard performance

2. **Query Optimization**:
   - Use projection to limit fields returned
   - Use pagination for large result sets
   - Use aggregation pipeline for complex reports

3. **Data Archiving**:
   - Archive old attendance records and financial transactions
   - Maintain summary data for reporting
   - Implement data retention policies per institution