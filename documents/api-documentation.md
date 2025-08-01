# API Documentation

This document provides detailed information about the API endpoints for the Educational Institution Management SaaS Platform.

## Base URL

```
https://api.institute-manage.com/v1
```

For development:

```
http://localhost:3000/api/v1
```

## Authentication

The API uses JWT (JSON Web Token) for authentication. Include the token in the Authorization header for all authenticated requests.

```
Authorization: Bearer <token>
```

### Multi-tenancy

All API endpoints automatically filter data based on the authenticated user's institution. The institution ID is extracted from the JWT token.

## Error Handling

The API returns standard HTTP status codes and a JSON response with error details:

```json
{
  "status": "error",
  "code": "RESOURCE_NOT_FOUND",
  "message": "The requested resource was not found",
  "details": { ... } // Optional additional error details
}
```

Common error codes:

- `INVALID_REQUEST`: The request is malformed or contains invalid parameters
- `UNAUTHORIZED`: Authentication is required or has failed
- `FORBIDDEN`: The authenticated user doesn't have permission for the requested action
- `RESOURCE_NOT_FOUND`: The requested resource doesn't exist
- `VALIDATION_ERROR`: The request data failed validation
- `INTERNAL_ERROR`: An unexpected error occurred on the server

## Pagination

List endpoints support pagination with the following query parameters:

- `page`: Page number (1-based, default: 1)
- `limit`: Number of items per page (default: 20, max: 100)
- `sort`: Field to sort by (format: `field:direction`, e.g., `createdAt:desc`)

Paginated responses include metadata:

```json
{
  "status": "success",
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 45,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

## Filtering

List endpoints support filtering with query parameters:

- Simple filters: `?status=active&category=math`
- Date range filters: `?startDate=2023-01-01&endDate=2023-12-31`
- Advanced filters: `?filter=<JSON-encoded-filter-object>`

## API Endpoints

### Authentication

#### Register Institution

```
POST /auth/register
```

Registers a new institution and creates an admin user.

**Request Body:**

```json
{
  "institution": {
    "name": "ABC Academy",
    "address": {
      "street": "123 Education St",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "contactEmail": "admin@abcacademy.com",
    "contactPhone": "+1234567890",
    "website": "https://abcacademy.com"
  },
  "admin": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "admin@abcacademy.com",
    "password": "securePassword123",
    "phone": "+1234567890"
  },
  "subscriptionPlan": "basic"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "institution": {
      "id": "60a1b2c3d4e5f6g7h8i9j0k1",
      "name": "ABC Academy",
      "contactEmail": "admin@abcacademy.com",
      "subscriptionPlan": "basic",
      "subscriptionStatus": "active",
      "createdAt": "2023-06-01T12:00:00Z"
    },
    "admin": {
      "id": "60a1b2c3d4e5f6g7h8i9j0k2",
      "email": "admin@abcacademy.com",
      "role": "admin"
    },
    "token": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 3600
    }
  }
}
```

#### Login

```
POST /auth/login
```

Authenticate a user and get access tokens.

**Request Body:**

```json
{
  "email": "admin@abcacademy.com",
  "password": "securePassword123"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "user": {
      "id": "60a1b2c3d4e5f6g7h8i9j0k2",
      "email": "admin@abcacademy.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "admin",
      "institutionId": "60a1b2c3d4e5f6g7h8i9j0k1"
    },
    "institution": {
      "id": "60a1b2c3d4e5f6g7h8i9j0k1",
      "name": "ABC Academy",
      "subscriptionStatus": "active"
    },
    "token": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 3600
    }
  }
}
```

#### Refresh Token

```
POST /auth/refresh-token
```

Get a new access token using a refresh token.

**Request Body:**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 3600
  }
}
```

#### Forgot Password

```
POST /auth/forgot-password
```

Initiate password reset process.

**Request Body:**

```json
{
  "email": "admin@abcacademy.com"
}
```

**Response:**

```json
{
  "status": "success",
  "message": "Password reset instructions sent to your email"
}
```

#### Reset Password

```
POST /auth/reset-password
```

Complete password reset process.

**Request Body:**

```json
{
  "token": "reset-token-from-email",
  "password": "newSecurePassword123",
  "confirmPassword": "newSecurePassword123"
}
```

**Response:**

```json
{
  "status": "success",
  "message": "Password has been reset successfully"
}
```

### Institution Management

#### Get Institution Details

```
GET /institutions/current
```

Get details of the current institution.

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k1",
    "name": "ABC Academy",
    "logo": "https://storage.example.com/logos/abc-academy.png",
    "address": {
      "street": "123 Education St",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "contactEmail": "admin@abcacademy.com",
    "contactPhone": "+1234567890",
    "website": "https://abcacademy.com",
    "customDomain": "learn.abcacademy.com",
    "subscriptionPlan": "basic",
    "subscriptionStatus": "active",
    "subscriptionStartDate": "2023-06-01T12:00:00Z",
    "subscriptionEndDate": "2024-06-01T12:00:00Z",
    "settings": {
      "theme": {
        "primaryColor": "#4f46e5",
        "secondaryColor": "#f59e0b"
      }
    },
    "createdAt": "2023-06-01T12:00:00Z",
    "updatedAt": "2023-06-01T12:00:00Z"
  }
}
```

#### Update Institution Details

```
PUT /institutions/current
```

Update details of the current institution.

**Request Body:**

```json
{
  "name": "ABC Academy International",
  "address": {
    "street": "456 Learning Ave",
    "city": "Knowledge City",
    "state": "Learning State",
    "country": "Education Nation",
    "postalCode": "12345"
  },
  "contactEmail": "info@abcacademy.com",
  "contactPhone": "+1234567890",
  "website": "https://abcacademy.com"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k1",
    "name": "ABC Academy International",
    "address": {
      "street": "456 Learning Ave",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "contactEmail": "info@abcacademy.com",
    "contactPhone": "+1234567890",
    "website": "https://abcacademy.com",
    "updatedAt": "2023-06-02T12:00:00Z"
  }
}
```

#### Update Institution Branding

```
PUT /institutions/current/branding
```

Update branding settings of the current institution.

**Request Body:**

```json
{
  "logo": "base64-encoded-image-data",
  "theme": {
    "primaryColor": "#2563eb",
    "secondaryColor": "#f59e0b",
    "fontFamily": "Inter"
  }
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "logo": "https://storage.example.com/logos/abc-academy-new.png",
    "theme": {
      "primaryColor": "#2563eb",
      "secondaryColor": "#f59e0b",
      "fontFamily": "Inter"
    }
  }
}
```

#### Get Dashboard Statistics

```
GET /institutions/current/dashboard
```

Get dashboard statistics for the current institution.

**Query Parameters:**

- `period`: Time period for statistics (today, week, month, year)
- `startDate`: Custom start date for statistics
- `endDate`: Custom end date for statistics

**Response:**

```json
{
  "status": "success",
  "data": {
    "students": {
      "total": 1234,
      "active": 1200,
      "inactive": 34,
      "newThisPeriod": 45
    },
    "courses": {
      "total": 25,
      "active": 20,
      "upcoming": 5
    },
    "staff": {
      "total": 50,
      "teachers": 40,
      "administrators": 10
    },
    "finances": {
      "totalRevenue": 50000,
      "totalExpenses": 30000,
      "pendingPayments": 5000,
      "revenueByCategory": [
        { "category": "Tuition", "amount": 45000 },
        { "category": "Registration", "amount": 5000 }
      ],
      "expensesByCategory": [
        { "category": "Salaries", "amount": 25000 },
        { "category": "Utilities", "amount": 3000 },
        { "category": "Supplies", "amount": 2000 }
      ]
    },
    "attendance": {
      "averageAttendanceRate": 92.5,
      "attendanceByDay": [
        { "date": "2023-06-01", "rate": 94.2 },
        { "date": "2023-06-02", "rate": 91.8 },
        { "date": "2023-06-03", "rate": 93.5 }
      ]
    }
  }
}
```

#### Generate Registration Link/QR

```
POST /institutions/current/registration-link
```

Generate a unique registration link and QR code for student registration.

**Request Body:**

```json
{
  "expiresIn": 604800, // Optional: Expiry time in seconds (default: 1 week)
  "courseId": "60a1b2c3d4e5f6g7h8i9j0k3" // Optional: For course-specific registration
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "registrationLink": "https://institute-manage.com/register/abc123def456",
    "qrCodeUrl": "https://storage.example.com/qrcodes/abc123def456.png",
    "expiresAt": "2023-06-08T12:00:00Z"
  }
}
```

### Student Management

#### List Students

```
GET /students
```

Get a list of students with pagination and filtering.

**Query Parameters:**

- Standard pagination parameters
- `status`: Filter by status (active, inactive, etc.)
- `courseId`: Filter by course enrollment
- `search`: Search by name, email, or student ID

**Response:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "60a1b2c3d4e5f6g7h8i9j0k4",
      "studentId": "STU001",
      "name": {
        "first": "Jane",
        "last": "Smith"
      },
      "email": "jane.smith@example.com",
      "phone": "+1234567891",
      "status": "active",
      "enrollmentDate": "2023-05-15T10:00:00Z",
      "courses": [
        {
          "id": "60a1b2c3d4e5f6g7h8i9j0k3",
          "name": "Advanced Mathematics",
          "status": "enrolled"
        }
      ]
    },
    // More students...
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 45,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

#### Create Student

```
POST /students
```

Create a new student.

**Request Body:**

```json
{
  "name": {
    "first": "John",
    "middle": "Robert",
    "last": "Johnson"
  },
  "email": "john.johnson@example.com",
  "phone": "+1234567892",
  "dateOfBirth": "2000-01-15",
  "gender": "male",
  "address": {
    "street": "789 Student Ave",
    "city": "Knowledge City",
    "state": "Learning State",
    "country": "Education Nation",
    "postalCode": "12345"
  },
  "guardianInfo": [
    {
      "name": "Robert Johnson",
      "relationship": "father",
      "phone": "+1234567893",
      "email": "robert.johnson@example.com",
      "isPrimary": true
    }
  ],
  "courses": [
    {
      "courseId": "60a1b2c3d4e5f6g7h8i9j0k3"
    }
  ],
  "customFields": {
    "previousSchool": "XYZ School",
    "allergies": "None"
  }
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k5",
    "studentId": "STU002",
    "name": {
      "first": "John",
      "middle": "Robert",
      "last": "Johnson"
    },
    "email": "john.johnson@example.com",
    "phone": "+1234567892",
    "dateOfBirth": "2000-01-15",
    "gender": "male",
    "address": {
      "street": "789 Student Ave",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "guardianInfo": [
      {
        "name": "Robert Johnson",
        "relationship": "father",
        "phone": "+1234567893",
        "email": "robert.johnson@example.com",
        "isPrimary": true
      }
    ],
    "status": "active",
    "enrollmentDate": "2023-06-02T12:00:00Z",
    "courses": [
      {
        "courseId": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics",
        "enrollmentDate": "2023-06-02T12:00:00Z",
        "status": "enrolled"
      }
    ],
    "customFields": {
      "previousSchool": "XYZ School",
      "allergies": "None"
    },
    "createdAt": "2023-06-02T12:00:00Z"
  }
}
```

#### Get Student Details

```
GET /students/:id
```

Get detailed information about a specific student.

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k5",
    "studentId": "STU002",
    "name": {
      "first": "John",
      "middle": "Robert",
      "last": "Johnson"
    },
    "email": "john.johnson@example.com",
    "phone": "+1234567892",
    "dateOfBirth": "2000-01-15",
    "gender": "male",
    "address": {
      "street": "789 Student Ave",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "guardianInfo": [
      {
        "name": "Robert Johnson",
        "relationship": "father",
        "phone": "+1234567893",
        "email": "robert.johnson@example.com",
        "isPrimary": true
      }
    ],
    "status": "active",
    "enrollmentDate": "2023-06-02T12:00:00Z",
    "courses": [
      {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics",
        "code": "MATH301",
        "enrollmentDate": "2023-06-02T12:00:00Z",
        "status": "enrolled",
        "progress": 25,
        "schedule": [
          {
            "day": "Monday",
            "startTime": "10:00",
            "endTime": "12:00"
          },
          {
            "day": "Wednesday",
            "startTime": "10:00",
            "endTime": "12:00"
          }
        ]
      }
    ],
    "documents": [
      {
        "type": "ID Proof",
        "title": "National ID Card",
        "url": "https://storage.example.com/documents/student-id-proof.pdf",
        "uploadedAt": "2023-06-02T12:30:00Z"
      }
    ],
    "customFields": {
      "previousSchool": "XYZ School",
      "allergies": "None"
    },
    "createdAt": "2023-06-02T12:00:00Z",
    "updatedAt": "2023-06-02T12:30:00Z"
  }
}
```

#### Update Student

```
PUT /students/:id
```

Update student information.

**Request Body:**

```json
{
  "name": {
    "first": "John",
    "middle": "Robert",
    "last": "Johnson-Smith"
  },
  "phone": "+1234567894",
  "address": {
    "street": "101 New Address St",
    "city": "Knowledge City",
    "state": "Learning State",
    "country": "Education Nation",
    "postalCode": "12345"
  },
  "guardianInfo": [
    {
      "name": "Robert Johnson",
      "relationship": "father",
      "phone": "+1234567893",
      "email": "robert.johnson@example.com",
      "isPrimary": true
    },
    {
      "name": "Mary Johnson",
      "relationship": "mother",
      "phone": "+1234567895",
      "email": "mary.johnson@example.com",
      "isPrimary": false
    }
  ],
  "status": "active",
  "customFields": {
    "previousSchool": "XYZ School",
    "allergies": "Peanuts"
  }
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k5",
    "name": {
      "first": "John",
      "middle": "Robert",
      "last": "Johnson-Smith"
    },
    "phone": "+1234567894",
    "address": {
      "street": "101 New Address St",
      "city": "Knowledge City",
      "state": "Learning State",
      "country": "Education Nation",
      "postalCode": "12345"
    },
    "guardianInfo": [
      {
        "name": "Robert Johnson",
        "relationship": "father",
        "phone": "+1234567893",
        "email": "robert.johnson@example.com",
        "isPrimary": true
      },
      {
        "name": "Mary Johnson",
        "relationship": "mother",
        "phone": "+1234567895",
        "email": "mary.johnson@example.com",
        "isPrimary": false
      }
    ],
    "status": "active",
    "customFields": {
      "previousSchool": "XYZ School",
      "allergies": "Peanuts"
    },
    "updatedAt": "2023-06-03T12:00:00Z"
  }
}
```

#### Get Student Attendance

```
GET /students/:id/attendance
```

Get attendance records for a specific student.

**Query Parameters:**

- Standard pagination parameters
- `startDate`: Filter by start date
- `endDate`: Filter by end date
- `courseId`: Filter by course

**Response:**

```json
{
  "status": "success",
  "data": [
    {
      "date": "2023-06-01",
      "course": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics",
        "code": "MATH301"
      },
      "status": "present",
      "checkInTime": "2023-06-01T10:05:00Z",
      "checkOutTime": "2023-06-01T12:00:00Z",
      "duration": 115,
      "method": "qr"
    },
    {
      "date": "2023-06-03",
      "course": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics",
        "code": "MATH301"
      },
      "status": "absent",
      "notes": "Sick leave"
    }
    // More attendance records...
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 10,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPrevPage": false
  },
  "summary": {
    "totalClasses": 10,
    "present": 9,
    "absent": 1,
    "late": 0,
    "excused": 0,
    "attendanceRate": 90
  }
}
```

#### Get Student Payments

```
GET /students/:id/payments
```

Get payment records for a specific student.

**Query Parameters:**

- Standard pagination parameters
- `status`: Filter by payment status
- `startDate`: Filter by start date
- `endDate`: Filter by end date

**Response:**

```json
{
  "status": "success",
  "data": [
    {
      "id": 1001,
      "amount": 500,
      "currency": "USD",
      "paymentDate": "2023-06-01T14:30:00Z",
      "dueDate": "2023-06-01T00:00:00Z",
      "paymentMethod": "credit_card",
      "status": "completed",
      "receiptNumber": "REC1001",
      "receiptUrl": "https://storage.example.com/receipts/rec1001.pdf",
      "course": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics"
      }
    },
    {
      "id": 1002,
      "amount": 500,
      "currency": "USD",
      "paymentDate": null,
      "dueDate": "2023-07-01T00:00:00Z",
      "status": "pending",
      "course": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics"
      }
    }
    // More payment records...
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 2,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPrevPage": false
  },
  "summary": {
    "totalPaid": 500,
    "totalPending": 500,
    "totalOverdue": 0,
    "currency": "USD"
  }
}
```

### Course Management

#### List Courses

```
GET /courses
```

Get a list of courses with pagination and filtering.

**Query Parameters:**

- Standard pagination parameters
- `status`: Filter by status (active, inactive, etc.)
- `search`: Search by name or code

**Response:**

```json
{
  "status": "success",
  "data": [
    {
      "id": "60a1b2c3d4e5f6g7h8i9j0k3",
      "code": "MATH301",
      "name": "Advanced Mathematics",
      "description": "Advanced topics in mathematics including calculus and linear algebra",
      "category": "Mathematics",
      "duration": {
        "value": 3,
        "unit": "months"
      },
      "startDate": "2023-06-01T00:00:00Z",
      "endDate": "2023-08-31T00:00:00Z",
      "fee": {
        "amount": 1500,
        "currency": "USD"
      },
      "enrolledCount": 25,
      "capacity": 30,
      "isActive": true
    },
    // More courses...
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 25,
    "totalPages": 2,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

#### Create Course

```
POST /courses
```

Create a new course.

**Request Body:**

```json
{
  "code": "ENG201",
  "name": "Business English",
  "description": "English language course focused on business communication",
  "category": "Language",
  "duration": {
    "value": 2,
    "unit": "months"
  },
  "startDate": "2023-07-01",
  "endDate": "2023-08-31",
  "fee": {
    "amount": 1200,
    "currency": "USD",
    "installments": true,
    "installmentDetails": [
      {
        "dueDate": "2023-07-01",
        "amount": 600
      },
      {
        "dueDate": "2023-08-01",
        "amount": 600
      }
    ]
  },
  "capacity": 20,
  "schedule": [
    {
      "day": "Tuesday",
      "startTime": "14:00",
      "endTime": "16:00",
      "roomId": "60a1b2c3d4e5f6g7h8i9j0k6"
    },
    {
      "day": "Thursday",
      "startTime": "14:00",
      "endTime": "16:00",
      "roomId": "60a1b2c3d4e5f6g7h8i9j0k6"
    }
  ],
  "instructors": ["60a1b2c3d4e5f6g7h8i9j0k7"],
  "syllabus": [
    {
      "title": "Introduction to Business Communication",
      "description": "Overview of business communication principles",
      "order": 1
    },
    {
      "title": "Email and Letter Writing",
      "description": "Professional email and letter writing techniques",
      "order": 2
    }
  ]
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": "60a1b2c3d4e5f6g7h8i9j0k8",
    "code": "ENG201",
    "name": "Business English",
    "description": "English language course focused on business communication",
    "category": "Language",
    "duration": {
      "value": 2,
      "unit": "months"
    },
    "startDate": "2023-07-01T00:00:00Z",
    "endDate": "2023-08-31T00:00:00Z",
    "fee": {
      "amount": 1200,
      "currency": "USD",
      "installments": true,
      "installmentDetails": [
        {
          "dueDate": "2023-07-01T00:00:00Z",
          "amount": 600
        },
        {
          "dueDate": "2023-08-01T00:00:00Z",
          "amount": 600
        }
      ]
    },
    "capacity": 20,
    "enrolledCount": 0,
    "schedule": [
      {
        "day": "Tuesday",
        "startTime": "14:00",
        "endTime": "16:00",
        "roomId": "60a1b2c3d4e5f6g7h8i9j0k6",
        "room": {
          "name": "Classroom 101"
        }
      },
      {
        "day": "Thursday",
        "startTime": "14:00",
        "endTime": "16:00",
        "roomId": "60a1b2c3d4e5f6g7h8i9j0k6",
        "room": {
          "name": "Classroom 101"
        }
      }
    ],
    "instructors": [
      {
        "id": "60a1b2c3d4e5f6g7h8i9j0k7",
        "name": {
          "first": "David",
          "last": "Wilson"
        }
      }
    ],
    "syllabus": [
      {
        "title": "Introduction to Business Communication",
        "description": "Overview of business communication principles",
        "order": 1
      },
      {
        "title": "Email and Letter Writing",
        "description": "Professional email and letter writing techniques",
        "order": 2
      }
    ],
    "isActive": true,
    "createdAt": "2023-06-03T12:00:00Z"
  }
}
```

### Finance Management

#### Get Financial Dashboard

```
GET /finance/dashboard
```

Get financial dashboard data.

**Query Parameters:**

- `period`: Time period for statistics (today, week, month, year)
- `startDate`: Custom start date for statistics
- `endDate`: Custom end date for statistics

**Response:**

```json
{
  "status": "success",
  "data": {
    "summary": {
      "totalRevenue": 50000,
      "totalExpenses": 30000,
      "netIncome": 20000,
      "pendingPayments": 5000,
      "overduePayments": 2000,
      "currency": "USD"
    },
    "revenueByCategory": [
      { "category": "Tuition", "amount": 45000 },
      { "category": "Registration", "amount": 5000 }
    ],
    "expensesByCategory": [
      { "category": "Salaries", "amount": 25000 },
      { "category": "Utilities", "amount": 3000 },
      { "category": "Supplies", "amount": 2000 }
    ],
    "revenueByMonth": [
      { "month": "2023-01", "amount": 40000 },
      { "month": "2023-02", "amount": 42000 },
      { "month": "2023-03", "amount": 45000 },
      { "month": "2023-04", "amount": 48000 },
      { "month": "2023-05", "amount": 50000 },
      { "month": "2023-06", "amount": 50000 }
    ],
    "expensesByMonth": [
      { "month": "2023-01", "amount": 28000 },
      { "month": "2023-02", "amount": 29000 },
      { "month": "2023-03", "amount": 29500 },
      { "month": "2023-04", "amount": 30000 },
      { "month": "2023-05", "amount": 30000 },
      { "month": "2023-06", "amount": 30000 }
    ],
    "upcomingPayments": [
      {
        "id": 1002,
        "dueDate": "2023-07-01T00:00:00Z",
        "amount": 600,
        "currency": "USD",
        "student": {
          "id": "60a1b2c3d4e5f6g7h8i9j0k5",
          "name": {
            "first": "John",
            "last": "Johnson-Smith"
          }
        },
        "course": {
          "id": "60a1b2c3d4e5f6g7h8i9j0k3",
          "name": "Advanced Mathematics"
        }
      },
      // More upcoming payments...
    ],
    "overduePayments": [
      // Overdue payments...
    ]
  }
}
```

#### List Payments

```
GET /finance/payments
```

Get a list of payments with pagination and filtering.

**Query Parameters:**

- Standard pagination parameters
- `status`: Filter by payment status
- `startDate`: Filter by start date
- `endDate`: Filter by end date
- `studentId`: Filter by student
- `courseId`: Filter by course

**Response:**

```json
{
  "status": "success",
  "data": [
    {
      "id": 1001,
      "amount": 500,
      "currency": "USD",
      "paymentDate": "2023-06-01T14:30:00Z",
      "dueDate": "2023-06-01T00:00:00Z",
      "paymentMethod": "credit_card",
      "status": "completed",
      "receiptNumber": "REC1001",
      "receiptUrl": "https://storage.example.com/receipts/rec1001.pdf",
      "student": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k5",
        "name": {
          "first": "John",
          "last": "Johnson-Smith"
        }
      },
      "course": {
        "id": "60a1b2c3d4e5f6g7h8i9j0k3",
        "name": "Advanced Mathematics"
      }
    },
    // More payments...
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 45,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

#### Create Payment

```
POST /finance/payments
```

Record a new payment.

**Request Body:**

```json
{
  "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
  "courseId": "60a1b2c3d4e5f6g7h8i9j0k3",
  "invoiceId": 1003,
  "amount": 600,
  "currency": "USD",
  "paymentDate": "2023-06-03T15:45:00Z",
  "paymentMethod": "bank_transfer",
  "transactionId": "TRX123456",
  "notes": "Second installment payment"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "id": 1004,
    "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
    "courseId": "60a1b2c3d4e5f6g7h8i9j0k3",
    "invoiceId": 1003,
    "amount": 600,
    "currency": "USD",
    "paymentDate": "2023-06-03T15:45:00Z",
    "paymentMethod": "bank_transfer",
    "transactionId": "TRX123456",
    "status": "completed",
    "receiptNumber": "REC1004",
    "receiptUrl": "https://storage.example.com/receipts/rec1004.pdf",
    "notes": "Second installment payment",
    "createdAt": "2023-06-03T15:45:00Z"
  }
}
```

### Attendance Management

#### Generate QR Code for Attendance

```
POST /attendance/generate-qr
```

Generate a QR code for attendance marking.

**Request Body:**

```json
{
  "courseId": "60a1b2c3d4e5f6g7h8i9j0k3",
  "date": "2023-06-04",
  "validityMinutes": 15
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "qrCodeUrl": "https://storage.example.com/qrcodes/attendance-abc123.png",
    "attendanceLink": "https://institute-manage.com/attendance/scan/abc123",
    "expiresAt": "2023-06-04T10:15:00Z"
  }
}
```

#### Record Attendance via QR Scan

```
POST /attendance/scan
```

Record attendance when a student scans a QR code.

**Request Body:**

```json
{
  "token": "abc123",
  "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "accuracy": 10
  },
  "device": "iPhone 12, iOS 15.4"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
    "studentName": "John Johnson-Smith",
    "courseId": "60a1b2c3d4e5f6g7h8i9j0k3",
    "courseName": "Advanced Mathematics",
    "date": "2023-06-04",
    "checkInTime": "2023-06-04T10:05:00Z",
    "status": "present",
    "method": "qr"
  }
}
```

### Payment Integration

#### Create Payment Session

```
POST /payments/create-session
```

Create a payment session with a payment gateway.

**Request Body:**

```json
{
  "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
  "invoiceId": 1005,
  "amount": 600,
  "currency": "USD",
  "description": "Payment for Business English course",
  "successUrl": "https://institute-manage.com/payment/success",
  "cancelUrl": "https://institute-manage.com/payment/cancel"
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "sessionId": "cs_test_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "paymentUrl": "https://checkout.stripe.com/pay/cs_test_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "expiresAt": "2023-06-04T11:30:00Z"
  }
}
```

#### Verify Payment Status

```
GET /payments/verify/:paymentId
```

Verify the status of a payment.

**Response:**

```json
{
  "status": "success",
  "data": {
    "paymentId": "pi_1J2K3L4M5N6O7P8Q9R0S1T2U",
    "sessionId": "cs_test_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
    "amount": 600,
    "currency": "USD",
    "status": "completed",
    "paymentMethod": "credit_card",
    "paymentDate": "2023-06-04T11:15:00Z",
    "receiptUrl": "https://storage.example.com/receipts/rec1005.pdf"
  }
}
```

### Notification System

#### Send Email Notification

```
POST /notifications/send-email
```

Send an email notification to students or staff.

**Request Body:**

```json
{
  "recipients": [
    {
      "type": "student",
      "id": "60a1b2c3d4e5f6g7h8i9j0k5"
    }
  ],
  "subject": "Payment Reminder",
  "templateId": "payment-reminder",
  "templateData": {
    "studentName": "John",
    "courseName": "Business English",
    "dueDate": "2023-07-01",
    "amount": 600,
    "currency": "USD",
    "paymentLink": "https://institute-manage.com/pay/inv1005"
  }
}
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "notificationId": "60a1b2c3d4e5f6g7h8i9j0k9",
    "recipients": [
      {
        "id": "60a1b2c3d4e5f6g7h8i9j0k5",
        "email": "john.johnson@example.com",
        "status": "sent"
      }
    ],
    "sentAt": "2023-06-04T12:00:00Z"
  }
}
```

## Webhook Events

The API provides webhooks for real-time event notifications. Configure webhook endpoints in the institution settings.

### Available Events

- `payment.completed`: Triggered when a payment is completed
- `payment.failed`: Triggered when a payment fails
- `student.enrolled`: Triggered when a student enrolls in a course
- `attendance.marked`: Triggered when attendance is marked
- `invoice.created`: Triggered when an invoice is created
- `invoice.paid`: Triggered when an invoice is fully paid
- `reminder.sent`: Triggered when a payment reminder is sent

### Webhook Payload

```json
{
  "event": "payment.completed",
  "institutionId": "60a1b2c3d4e5f6g7h8i9j0k1",
  "timestamp": "2023-06-04T12:30:00Z",
  "data": {
    "paymentId": 1004,
    "studentId": "60a1b2c3d4e5f6g7h8i9j0k5",
    "amount": 600,
    "currency": "USD",
    "status": "completed"
  }
}
```

## Rate Limiting

The API implements rate limiting to prevent abuse. Rate limits are applied per API key and vary by endpoint.

- Authentication endpoints: 10 requests per minute
- Read endpoints: 60 requests per minute
- Write endpoints: 30 requests per minute

Rate limit information is included in the response headers:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
X-RateLimit-Reset: 1623456789
```

When rate limits are exceeded, the API returns a 429 Too Many Requests response.

## Versioning

The API is versioned using URL path versioning (e.g., `/v1/students`). When breaking changes are introduced, a new version is released while maintaining support for previous versions for a deprecation period.

## Changelog

### v1.0.0 (2023-06-01)

- Initial release of the API
- Authentication endpoints
- Institution management
- Student management
- Course management
- Finance management
- Attendance management
- Payment integration
- Notification system