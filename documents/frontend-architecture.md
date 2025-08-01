# Frontend Architecture

This document outlines the frontend architecture for the Educational Institution Management SaaS Platform, focusing on UI/UX design, component structure, state management, and responsive design principles.

## Technology Stack

### Core Technologies

- **Framework**: React.js with Next.js
- **Language**: TypeScript
- **Styling**: Tailwind CSS with custom theme configuration
- **State Management**: Redux Toolkit for global state, React Query for server state
- **Form Handling**: React Hook Form with Zod validation
- **Routing**: Next.js App Router
- **Authentication**: JWT with secure HTTP-only cookies

### UI Component Libraries

- **Component Framework**: Headless UI or Radix UI for accessible base components
- **Icons**: Heroicons or Phosphor Icons
- **Data Visualization**: Chart.js or Recharts for dashboards
- **Tables**: TanStack Table (React Table) for data grids
- **Date Handling**: date-fns for date manipulation
- **Notifications**: React Hot Toast or React Toastify
- **Modals/Dialogs**: Headless UI Dialog or custom implementation

## Application Structure

```
src/
├── app/                    # Next.js App Router pages
│   ├── (auth)/             # Authentication routes (login, register, etc.)
│   ├── (dashboard)/        # Protected dashboard routes
│   │   ├── students/       # Student management pages
│   │   ├── courses/        # Course management pages
│   │   ├── staff/          # Staff management pages
│   │   ├── inventory/      # Inventory management pages
│   │   ├── finance/        # Finance management pages
│   │   ├── attendance/     # Attendance management pages
│   │   └── settings/       # Settings pages
│   ├── (public)/           # Public routes
│   │   ├── register/       # Institution registration
│   │   └── student-portal/ # Student portal access
│   └── api/                # API routes (if using Next.js API)
├── components/             # Reusable UI components
│   ├── common/             # Common components (buttons, inputs, etc.)
│   ├── layout/             # Layout components
│   ├── dashboard/          # Dashboard-specific components
│   ├── students/           # Student management components
│   ├── courses/            # Course management components
│   ├── staff/              # Staff management components
│   ├── inventory/          # Inventory management components
│   ├── finance/            # Finance management components
│   ├── attendance/         # Attendance management components
│   └── settings/           # Settings components
├── hooks/                  # Custom React hooks
├── lib/                    # Utility functions and libraries
│   ├── api/                # API client and utilities
│   ├── auth/               # Authentication utilities
│   ├── validation/         # Validation schemas
│   └── utils/              # General utilities
├── store/                  # Redux store configuration
│   ├── slices/             # Redux slices
│   └── middleware/         # Redux middleware
├── styles/                 # Global styles and Tailwind configuration
├── types/                  # TypeScript type definitions
└── providers/              # Context providers
```

## Component Architecture

### Component Design Principles

1. **Atomic Design Methodology**
   - Atoms: Basic UI elements (buttons, inputs, icons)
   - Molecules: Simple component combinations (form fields, search bars)
   - Organisms: Complex UI sections (data tables, forms, cards)
   - Templates: Page layouts without specific content
   - Pages: Complete views with actual content

2. **Component Composition**
   - Prefer composition over inheritance
   - Use higher-order components and render props sparingly
   - Implement compound components for complex UI elements

3. **Reusability and Consistency**
   - Create a component library with consistent styling and behavior
   - Document components with PropTypes or TypeScript interfaces
   - Implement storybook for component documentation and testing

### Key Component Categories

#### Layout Components

```tsx
// DashboardLayout.tsx
import { ReactNode } from 'react';
import Sidebar from './Sidebar';
import Header from './Header';
import Footer from './Footer';

type DashboardLayoutProps = {
  children: ReactNode;
};

export default function DashboardLayout({ children }: DashboardLayoutProps) {
  return (
    <div className="flex h-screen bg-gray-50">
      <Sidebar />
      <div className="flex flex-col flex-1 overflow-hidden">
        <Header />
        <main className="flex-1 overflow-y-auto p-4 md:p-6">
          {children}
        </main>
        <Footer />
      </div>
    </div>
  );
}
```

#### Common UI Components

```tsx
// Button.tsx
import { ButtonHTMLAttributes, ReactNode } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2",
  {
    variants: {
      variant: {
        primary: "bg-primary-600 text-white hover:bg-primary-700 focus:ring-primary-500",
        secondary: "bg-white text-gray-700 border border-gray-300 hover:bg-gray-50 focus:ring-primary-500",
        danger: "bg-red-600 text-white hover:bg-red-700 focus:ring-red-500",
      },
      size: {
        sm: "h-8 px-3 text-xs",
        md: "h-10 px-4",
        lg: "h-12 px-6 text-lg",
      },
      fullWidth: {
        true: "w-full",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "md",
      fullWidth: false,
    },
  }
);

type ButtonProps = ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof buttonVariants> & {
    children: ReactNode;
    isLoading?: boolean;
  };

export default function Button({
  children,
  variant,
  size,
  fullWidth,
  isLoading,
  className,
  ...props
}: ButtonProps) {
  return (
    <button
      className={buttonVariants({ variant, size, fullWidth, className })}
      disabled={isLoading || props.disabled}
      {...props}
    >
      {isLoading ? (
        <>
          <svg className="animate-spin -ml-1 mr-2 h-4 w-4 text-current" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          Loading...
        </>
      ) : (
        children
      )}
    </button>
  );
}
```

#### Data Display Components

```tsx
// DataTable.tsx
import {
  useReactTable,
  getCoreRowModel,
  getPaginationRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  flexRender,
  ColumnDef,
} from '@tanstack/react-table';
import { useState } from 'react';

type DataTableProps<T> = {
  data: T[];
  columns: ColumnDef<T, any>[];
  initialSortField?: string;
  searchPlaceholder?: string;
};

export default function DataTable<T>({ 
  data, 
  columns, 
  initialSortField,
  searchPlaceholder = 'Search...' 
}: DataTableProps<T>) {
  const [globalFilter, setGlobalFilter] = useState('');
  
  const table = useReactTable({
    data,
    columns,
    state: {
      globalFilter,
    },
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    initialState: {
      pagination: {
        pageSize: 10,
      },
      sorting: initialSortField ? [
        { id: initialSortField, desc: false }
      ] : [],
    },
  });

  return (
    <div className="bg-white shadow-md rounded-lg overflow-hidden">
      <div className="p-4 border-b border-gray-200 bg-gray-50">
        <div className="flex items-center justify-between">
          <input
            type="text"
            value={globalFilter ?? ''}
            onChange={(e) => setGlobalFilter(e.target.value)}
            placeholder={searchPlaceholder}
            className="px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500 w-full md:w-64"
          />
        </div>
      </div>
      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map((headerGroup) => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <th
                    key={header.id}
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div className="flex items-center">
                      {flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                      {header.column.getIsSorted() ? (
                        header.column.getIsSorted() === 'asc' ? (
                          <svg className="ml-1 w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 15l7-7 7 7" />
                          </svg>
                        ) : (
                          <svg className="ml-1 w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 9l-7 7-7-7" />
                          </svg>
                        )
                      ) : null}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.length > 0 ? (
              table.getRowModel().rows.map((row) => (
                <tr key={row.id} className="hover:bg-gray-50">
                  {row.getVisibleCells().map((cell) => (
                    <td key={cell.id} className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              ))
            ) : (
              <tr>
                <td colSpan={columns.length} className="px-6 py-4 text-center text-sm text-gray-500">
                  No results found
                </td>
              </tr>
            )}
          </tbody>
        </table>
      </div>
      <div className="px-4 py-3 bg-gray-50 border-t border-gray-200 sm:px-6">
        <div className="flex items-center justify-between">
          <div className="flex items-center">
            <span className="text-sm text-gray-700">
              Page{' '}
              <span className="font-medium">{table.getState().pagination.pageIndex + 1}</span>{' '}
              of{' '}
              <span className="font-medium">{table.getPageCount()}</span>
            </span>
            <select
              value={table.getState().pagination.pageSize}
              onChange={(e) => table.setPageSize(Number(e.target.value))}
              className="ml-2 border-gray-300 rounded-md text-sm focus:outline-none focus:ring-primary-500 focus:border-primary-500"
            >
              {[10, 20, 30, 40, 50].map((pageSize) => (
                <option key={pageSize} value={pageSize}>
                  Show {pageSize}
                </option>
              ))}
            </select>
          </div>
          <div className="flex items-center space-x-2">
            <button
              onClick={() => table.previousPage()}
              disabled={!table.getCanPreviousPage()}
              className="px-3 py-1 border border-gray-300 rounded-md text-sm font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed"
            >
              Previous
            </button>
            <button
              onClick={() => table.nextPage()}
              disabled={!table.getCanNextPage()}
              className="px-3 py-1 border border-gray-300 rounded-md text-sm font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed"
            >
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

#### Form Components

```tsx
// FormField.tsx
import { ReactNode } from 'react';
import { useFormContext } from 'react-hook-form';

type FormFieldProps = {
  name: string;
  label: string;
  children: ReactNode;
  required?: boolean;
  description?: string;
};

export default function FormField({
  name,
  label,
  children,
  required,
  description,
}: FormFieldProps) {
  const { formState: { errors } } = useFormContext();
  const error = errors[name];

  return (
    <div className="mb-4">
      <label htmlFor={name} className="block text-sm font-medium text-gray-700 mb-1">
        {label}
        {required && <span className="text-red-500 ml-1">*</span>}
      </label>
      {children}
      {description && (
        <p className="mt-1 text-xs text-gray-500">{description}</p>
      )}
      {error && (
        <p className="mt-1 text-xs text-red-500">
          {error.message?.toString()}
        </p>
      )}
    </div>
  );
}
```

#### Dashboard Components

```tsx
// StatisticsCard.tsx
type StatisticsCardProps = {
  title: string;
  value: string | number;
  icon: React.ReactNode;
  change?: {
    value: number;
    isPositive: boolean;
  };
  description?: string;
};

export default function StatisticsCard({
  title,
  value,
  icon,
  change,
  description,
}: StatisticsCardProps) {
  return (
    <div className="bg-white rounded-lg shadow-md p-6 border border-gray-200">
      <div className="flex items-center justify-between mb-4">
        <h3 className="text-lg font-semibold text-gray-800">{title}</h3>
        <span className="p-2 bg-primary-100 text-primary-700 rounded-full">
          {icon}
        </span>
      </div>
      <p className="text-3xl font-bold text-gray-900">{value}</p>
      {change && (
        <div className="flex items-center mt-2">
          <span
            className={`text-sm flex items-center ${
              change.isPositive ? 'text-success-500' : 'text-error-500'
            }`}
          >
            {change.isPositive ? (
              <svg className="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 10l7-7m0 0l7 7m-7-7v18" />
              </svg>
            ) : (
              <svg className="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 14l-7 7m0 0l-7-7m7 7V3" />
              </svg>
            )}
            {Math.abs(change.value)}%
          </span>
          <span className="text-sm text-gray-500 ml-2">from last month</span>
        </div>
      )}
      {description && <p className="mt-2 text-sm text-gray-500">{description}</p>}
    </div>
  );
}
```

## State Management

### Global State Management

Redux Toolkit is used for global application state, with a focus on minimizing the amount of state stored globally.

```tsx
// store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { setupListeners } from '@reduxjs/toolkit/query';
import authReducer from './slices/authSlice';
import uiReducer from './slices/uiSlice';
import { api } from './api';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    ui: uiReducer,
    [api.reducerPath]: api.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware),
});

setupListeners(store.dispatch);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Server State Management

React Query is used for server state management, providing caching, background updates, and optimistic updates.

```tsx
// hooks/useStudents.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '../lib/api';
import type { Student, StudentCreateInput, StudentUpdateInput } from '../types';

export function useStudents(filters?: Record<string, any>) {
  return useQuery({
    queryKey: ['students', filters],
    queryFn: () => api.students.list(filters),
  });
}

export function useStudent(id: string) {
  return useQuery({
    queryKey: ['students', id],
    queryFn: () => api.students.get(id),
    enabled: !!id,
  });
}

export function useCreateStudent() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: StudentCreateInput) => api.students.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['students'] });
    },
  });
}

export function useUpdateStudent() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: StudentUpdateInput }) => 
      api.students.update(id, data),
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['students'] });
      queryClient.invalidateQueries({ queryKey: ['students', data.id] });
    },
  });
}

export function useDeleteStudent() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (id: string) => api.students.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['students'] });
    },
  });
}
```

### Form State Management

React Hook Form is used for form state management, with Zod for validation.

```tsx
// lib/validation/studentSchema.ts
import { z } from 'zod';

export const studentSchema = z.object({
  name: z.object({
    first: z.string().min(1, 'First name is required'),
    middle: z.string().optional(),
    last: z.string().min(1, 'Last name is required'),
  }),
  email: z.string().email('Invalid email address'),
  phone: z.string().min(1, 'Phone number is required'),
  dateOfBirth: z.string().optional(),
  gender: z.enum(['male', 'female', 'other', '']).optional(),
  address: z.object({
    street: z.string().min(1, 'Street is required'),
    city: z.string().min(1, 'City is required'),
    state: z.string().min(1, 'State is required'),
    country: z.string().min(1, 'Country is required'),
    postalCode: z.string().min(1, 'Postal code is required'),
  }),
  guardianInfo: z.array(
    z.object({
      name: z.string().min(1, 'Guardian name is required'),
      relationship: z.string().min(1, 'Relationship is required'),
      phone: z.string().min(1, 'Phone number is required'),
      email: z.string().email('Invalid email address').optional(),
      isPrimary: z.boolean().default(false),
    })
  ).optional(),
  courses: z.array(
    z.object({
      courseId: z.string().min(1, 'Course is required'),
    })
  ).optional(),
});

export type StudentFormValues = z.infer<typeof studentSchema>;
```

```tsx
// components/students/StudentForm.tsx
import { useForm, FormProvider } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { studentSchema, type StudentFormValues } from '../../lib/validation/studentSchema';
import FormField from '../common/FormField';
import Button from '../common/Button';

type StudentFormProps = {
  initialValues?: Partial<StudentFormValues>;
  onSubmit: (data: StudentFormValues) => void;
  isSubmitting?: boolean;
};

export default function StudentForm({
  initialValues,
  onSubmit,
  isSubmitting,
}: StudentFormProps) {
  const methods = useForm<StudentFormValues>({
    resolver: zodResolver(studentSchema),
    defaultValues: initialValues || {
      name: { first: '', middle: '', last: '' },
      address: { street: '', city: '', state: '', country: '', postalCode: '' },
      guardianInfo: [{ name: '', relationship: '', phone: '', email: '', isPrimary: true }],
      courses: [],
    },
  });

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)} className="space-y-6">
        <div className="bg-white shadow-md rounded-lg p-6">
          <h2 className="text-xl font-semibold text-gray-800 mb-6">Personal Information</h2>
          <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
            <FormField name="name.first" label="First Name" required>
              <input
                id="name.first"
                type="text"
                {...methods.register('name.first')}
                className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              />
            </FormField>
            <FormField name="name.middle" label="Middle Name">
              <input
                id="name.middle"
                type="text"
                {...methods.register('name.middle')}
                className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              />
            </FormField>
            <FormField name="name.last" label="Last Name" required>
              <input
                id="name.last"
                type="text"
                {...methods.register('name.last')}
                className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              />
            </FormField>
          </div>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mt-6">
            <FormField name="email" label="Email Address" required>
              <input
                id="email"
                type="email"
                {...methods.register('email')}
                className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              />
            </FormField>
            <FormField name="phone" label="Phone Number" required>
              <input
                id="phone"
                type="tel"
                {...methods.register('phone')}
                className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              />
            </FormField>
          </div>
          
          {/* Additional form fields would go here */}
        </div>
        
        <div className="flex justify-end space-x-3">
          <Button
            type="button"
            variant="secondary"
            onClick={() => methods.reset()}
          >
            Reset
          </Button>
          <Button
            type="submit"
            isLoading={isSubmitting}
          >
            Save Student
          </Button>
        </div>
      </form>
    </FormProvider>
  );
}
```

## Responsive Design

### Breakpoint Strategy

Tailwind CSS breakpoints are used for responsive design:

- `sm`: 640px and above
- `md`: 768px and above
- `lg`: 1024px and above
- `xl`: 1280px and above
- `2xl`: 1536px and above

### Mobile-First Approach

All components are designed with a mobile-first approach, with responsive adjustments for larger screens.

```tsx
// Example of responsive design in a component
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6">
  <StatisticsCard title="Total Students" value={1234} icon={<UserIcon />} />
  <StatisticsCard title="Active Courses" value={25} icon={<BookIcon />} />
  <StatisticsCard title="Revenue" value="$50,000" icon={<CurrencyDollarIcon />} />
</div>
```

### Responsive Navigation

```tsx
// components/layout/Sidebar.tsx
import { useState } from 'react';
import Link from 'next/link';
import { usePathname } from 'next/navigation';

type NavItem = {
  name: string;
  href: string;
  icon: React.ReactNode;
};

const navigation: NavItem[] = [
  // Navigation items
];

export default function Sidebar() {
  const pathname = usePathname();
  const [isMobileMenuOpen, setIsMobileMenuOpen] = useState(false);

  return (
    <>
      {/* Mobile menu button */}
      <button
        type="button"
        className="md:hidden fixed top-4 left-4 z-50 p-2 rounded-md text-gray-400 hover:text-white hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-inset focus:ring-white"
        onClick={() => setIsMobileMenuOpen(!isMobileMenuOpen)}
      >
        <span className="sr-only">Open sidebar</span>
        {isMobileMenuOpen ? (
          <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
          </svg>
        ) : (
          <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 6h16M4 12h16M4 18h16" />
          </svg>
        )}
      </button>

      {/* Mobile menu */}
      <div
        className={`fixed inset-0 z-40 md:hidden ${isMobileMenuOpen ? 'block' : 'hidden'}`}
      >
        <div className="fixed inset-0 bg-gray-600 bg-opacity-75" onClick={() => setIsMobileMenuOpen(false)}></div>
        <div className="fixed inset-y-0 left-0 flex flex-col max-w-xs w-full bg-white shadow-xl">
          <div className="flex-1 flex flex-col pt-5 pb-4 overflow-y-auto">
            <div className="flex items-center justify-center h-16 flex-shrink-0 px-4">
              <img className="h-8 w-auto" src="/logo.svg" alt="Logo" />
            </div>
            <nav className="mt-5 flex-1 px-2 space-y-1">
              {navigation.map((item) => (
                <Link
                  key={item.name}
                  href={item.href}
                  className={`group flex items-center px-2 py-2 text-sm font-medium rounded-md ${
                    pathname.startsWith(item.href)
                      ? 'bg-primary-100 text-primary-700'
                      : 'text-gray-600 hover:bg-gray-50 hover:text-gray-900'
                  }`}
                >
                  <div className="mr-3 h-6 w-6">{item.icon}</div>
                  {item.name}
                </Link>
              ))}
            </nav>
          </div>
        </div>
      </div>

      {/* Desktop sidebar */}
      <div className="hidden md:flex md:flex-col md:w-64 md:fixed md:inset-y-0 bg-white shadow-md">
        <div className="flex-1 flex flex-col pt-5 pb-4 overflow-y-auto">
          <div className="flex items-center justify-center h-16 flex-shrink-0 px-4">
            <img className="h-8 w-auto" src="/logo.svg" alt="Logo" />
          </div>
          <nav className="mt-5 flex-1 px-2 space-y-1">
            {navigation.map((item) => (
              <Link
                key={item.name}
                href={item.href}
                className={`group flex items-center px-2 py-2 text-sm font-medium rounded-md ${
                  pathname.startsWith(item.href)
                    ? 'bg-primary-100 text-primary-700'
                    : 'text-gray-600 hover:bg-gray-50 hover:text-gray-900'
                }`}
              >
                <div className="mr-3 h-6 w-6">{item.icon}</div>
                {item.name}
              </Link>
            ))}
          </nav>
        </div>
      </div>
    </>
  );
}
```

## Theme Customization

### Tailwind Configuration

```js
// tailwind.config.js
const colors = require('tailwindcss/colors');

module.exports = {
  content: [
    './src/app/**/*.{js,ts,jsx,tsx}',
    './src/components/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: colors.indigo,
        accent: colors.amber,
        success: colors.green,
        error: colors.red,
        warning: colors.yellow,
        info: colors.blue,
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
      boxShadow: {
        card: '0 2px 4px rgba(0, 0, 0, 0.05), 0 1px 2px rgba(0, 0, 0, 0.1)',
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
};
```

### Dynamic Theming

```tsx
// providers/ThemeProvider.tsx
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';

type ThemeContextType = {
  primaryColor: string;
  secondaryColor: string;
  setPrimaryColor: (color: string) => void;
  setSecondaryColor: (color: string) => void;
};

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [primaryColor, setPrimaryColor] = useState('#4f46e5');
  const [secondaryColor, setSecondaryColor] = useState('#f59e0b');

  useEffect(() => {
    // Apply theme colors to CSS variables
    document.documentElement.style.setProperty('--color-primary', primaryColor);
    document.documentElement.style.setProperty('--color-secondary', secondaryColor);
  }, [primaryColor, secondaryColor]);

  return (
    <ThemeContext.Provider
      value={{
        primaryColor,
        secondaryColor,
        setPrimaryColor,
        setSecondaryColor,
      }}
    >
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
}
```

## Performance Optimization

### Code Splitting

Next.js App Router provides automatic code splitting at the page level. Additional code splitting can be implemented using dynamic imports.

```tsx
// Dynamic import example
import dynamic from 'next/dynamic';

const DynamicChart = dynamic(() => import('../components/charts/Chart'), {
  loading: () => <div className="h-64 w-full bg-gray-100 animate-pulse rounded-lg"></div>,
  ssr: false, // Disable server-side rendering for components that use browser APIs
});
```

### Image Optimization

Next.js Image component is used for automatic image optimization.

```tsx
import Image from 'next/image';

export default function ProfileImage({ src, alt }: { src: string; alt: string }) {
  return (
    <div className="relative h-12 w-12 rounded-full overflow-hidden">
      <Image
        src={src}
        alt={alt}
        fill
        sizes="(max-width: 768px) 100vw, 48px"
        className="object-cover"
      />
    </div>
  );
}
```

### Memoization

React's `useMemo` and `useCallback` hooks are used to optimize expensive calculations and prevent unnecessary re-renders.

```tsx
import { useMemo, useCallback } from 'react';

export default function StudentStatistics({ students }) {
  // Memoize expensive calculation
  const statistics = useMemo(() => {
    return {
      total: students.length,
      active: students.filter(s => s.status === 'active').length,
      inactive: students.filter(s => s.status === 'inactive').length,
      maleCount: students.filter(s => s.gender === 'male').length,
      femaleCount: students.filter(s => s.gender === 'female').length,
      averageAge: students.reduce((sum, s) => sum + calculateAge(s.dateOfBirth), 0) / students.length,
    };
  }, [students]);

  // Memoize callback function
  const handleExport = useCallback(() => {
    // Export logic
  }, [students]);

  return (
    <div>
      {/* Display statistics */}
    </div>
  );
}
```

## Accessibility

### ARIA Attributes

All interactive components include appropriate ARIA attributes for accessibility.

```tsx
// Accessible dropdown example
import { useState, useRef } from 'react';
import { useOnClickOutside } from '../../hooks/useOnClickOutside';

export default function Dropdown({ label, options, value, onChange }) {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef(null);

  useOnClickOutside(dropdownRef, () => setIsOpen(false));

  const selectedOption = options.find(option => option.value === value);

  return (
    <div className="relative" ref={dropdownRef}>
      <button
        type="button"
        className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500 bg-white text-left"
        onClick={() => setIsOpen(!isOpen)}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-labelledby="dropdown-label"
      >
        <span id="dropdown-label" className="sr-only">{label}</span>
        <span>{selectedOption ? selectedOption.label : 'Select an option'}</span>
        <span className="absolute inset-y-0 right-0 flex items-center pr-2 pointer-events-none">
          <svg className="h-5 w-5 text-gray-400" viewBox="0 0 20 20" fill="none" stroke="currentColor">
            <path d="M7 7l3 3 3-3" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round" />
          </svg>
        </span>
      </button>

      {isOpen && (
        <ul
          className="absolute z-10 mt-1 w-full bg-white shadow-lg max-h-60 rounded-md py-1 text-base ring-1 ring-black ring-opacity-5 overflow-auto focus:outline-none sm:text-sm"
          tabIndex={-1}
          role="listbox"
          aria-labelledby="dropdown-label"
        >
          {options.map((option) => (
            <li
              key={option.value}
              className={`cursor-default select-none relative py-2 pl-3 pr-9 hover:bg-gray-100 ${value === option.value ? 'bg-primary-100 text-primary-900' : 'text-gray-900'}`}
              role="option"
              aria-selected={value === option.value}
              onClick={() => {
                onChange(option.value);
                setIsOpen(false);
              }}
            >
              <span className={`block truncate ${value === option.value ? 'font-medium' : 'font-normal'}`}>
                {option.label}
              </span>
              {value === option.value && (
                <span className="absolute inset-y-0 right-0 flex items-center pr-4 text-primary-600">
                  <svg className="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                    <path fillRule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clipRule="evenodd" />
                  </svg>
                </span>
              )}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Keyboard Navigation

All interactive components support keyboard navigation for accessibility.

```tsx
// Keyboard-accessible tabs example
import { useState, useRef, useEffect } from 'react';

type Tab = {
  id: string;
  label: string;
  content: React.ReactNode;
};

type TabsProps = {
  tabs: Tab[];
  defaultTabId?: string;
};

export default function Tabs({ tabs, defaultTabId }: TabsProps) {
  const [activeTabId, setActiveTabId] = useState(defaultTabId || tabs[0].id);
  const tabRefs = useRef<Map<string, HTMLButtonElement>>(new Map());

  const handleKeyDown = (e: React.KeyboardEvent, tabId: string) => {
    const tabsArray = tabs.map(tab => tab.id);
    const currentIndex = tabsArray.indexOf(tabId);

    switch (e.key) {
      case 'ArrowRight':
        e.preventDefault();
        const nextIndex = (currentIndex + 1) % tabsArray.length;
        setActiveTabId(tabsArray[nextIndex]);
        tabRefs.current.get(tabsArray[nextIndex])?.focus();
        break;
      case 'ArrowLeft':
        e.preventDefault();
        const prevIndex = (currentIndex - 1 + tabsArray.length) % tabsArray.length;
        setActiveTabId(tabsArray[prevIndex]);
        tabRefs.current.get(tabsArray[prevIndex])?.focus();
        break;
      default:
        break;
    }
  };

  return (
    <div>
      <div className="border-b border-gray-200">
        <div className="-mb-px flex" role="tablist" aria-orientation="horizontal">
          {tabs.map((tab) => (
            <button
              key={tab.id}
              id={`tab-${tab.id}`}
              role="tab"
              aria-selected={activeTabId === tab.id}
              aria-controls={`tabpanel-${tab.id}`}
              tabIndex={activeTabId === tab.id ? 0 : -1}
              className={`py-2 px-4 text-sm font-medium border-b-2 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2 ${activeTabId === tab.id ? 'border-primary-500 text-primary-600' : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'}`}
              onClick={() => setActiveTabId(tab.id)}
              onKeyDown={(e) => handleKeyDown(e, tab.id)}
              ref={(el) => {
                if (el) tabRefs.current.set(tab.id, el);
              }}
            >
              {tab.label}
            </button>
          ))}
        </div>
      </div>
      <div className="mt-4">
        {tabs.map((tab) => (
          <div
            key={tab.id}
            id={`tabpanel-${tab.id}`}
            role="tabpanel"
            aria-labelledby={`tab-${tab.id}`}
            hidden={activeTabId !== tab.id}
          >
            {tab.content}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Multi-Tenant Customization

### Institution-Specific Theming

```tsx
// hooks/useInstitutionTheme.ts
import { useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';
import { api } from '../lib/api';

export function useInstitutionTheme() {
  const { data: institution } = useQuery({
    queryKey: ['institution', 'current'],
    queryFn: () => api.institutions.getCurrent(),
  });

  useEffect(() => {
    if (institution?.settings?.theme) {
      const { primaryColor, secondaryColor, fontFamily } = institution.settings.theme;
      
      if (primaryColor) {
        document.documentElement.style.setProperty('--color-primary', primaryColor);
      }
      
      if (secondaryColor) {
        document.documentElement.style.setProperty('--color-secondary', secondaryColor);
      }
      
      if (fontFamily) {
        document.documentElement.style.setProperty('--font-family', fontFamily);
      }
      
      // Apply logo if available
      if (institution.logo) {
        const favicon = document.querySelector('link[rel="icon"]');
        if (favicon) {
          favicon.setAttribute('href', institution.logo);
        }
      }
    }
  }, [institution]);

  return institution?.settings?.theme;
}
```

### Custom Fields Support

```tsx
// components/common/CustomFields.tsx
import { useState } from 'react';
import { useFormContext } from 'react-hook-form';
import FormField from './FormField';

type CustomFieldDefinition = {
  id: string;
  label: string;
  type: 'text' | 'number' | 'date' | 'select' | 'checkbox';
  options?: { value: string; label: string }[];
  required?: boolean;
};

type CustomFieldsProps = {
  fields: CustomFieldDefinition[];
  basePath: string; // Path in the form data where custom fields are stored
};

export default function CustomFields({ fields, basePath }: CustomFieldsProps) {
  const { register, formState: { errors } } = useFormContext();

  if (!fields || fields.length === 0) {
    return null;
  }

  return (
    <div className="mt-6">
      <h3 className="text-lg font-medium text-gray-900 mb-4">Additional Information</h3>
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {fields.map((field) => {
          const fieldPath = `${basePath}.${field.id}`;
          
          return (
            <FormField
              key={field.id}
              name={fieldPath}
              label={field.label}
              required={field.required}
            >
              {field.type === 'text' && (
                <input
                  type="text"
                  id={fieldPath}
                  {...register(fieldPath, { required: field.required })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
                />
              )}
              
              {field.type === 'number' && (
                <input
                  type="number"
                  id={fieldPath}
                  {...register(fieldPath, { required: field.required })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
                />
              )}
              
              {field.type === 'date' && (
                <input
                  type="date"
                  id={fieldPath}
                  {...register(fieldPath, { required: field.required })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
                />
              )}
              
              {field.type === 'select' && field.options && (
                <select
                  id={fieldPath}
                  {...register(fieldPath, { required: field.required })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
                >
                  <option value="">Select an option</option>
                  {field.options.map((option) => (
                    <option key={option.value} value={option.value}>
                      {option.label}
                    </option>
                  ))}
                </select>
              )}
              
              {field.type === 'checkbox' && (
                <div className="flex items-center">
                  <input
                    type="checkbox"
                    id={fieldPath}
                    {...register(fieldPath)}
                    className="h-4 w-4 text-primary-600 focus:ring-primary-500 border-gray-300 rounded"
                  />
                </div>
              )}
            </FormField>
          );
        })}
      </div>
    </div>
  );
}
```

## Testing Strategy

### Component Testing

React Testing Library and Jest are used for component testing.

```tsx
// components/common/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import Button from './Button';

describe('Button', () => {
  it('renders correctly', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });

  it('calls onClick handler when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    fireEvent.click(screen.getByRole('button', { name: /click me/i }));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('displays loading state', () => {
    render(<Button isLoading>Click me</Button>);
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('applies variant styles correctly', () => {
    const { rerender } = render(<Button variant="primary">Primary</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-primary-600');

    rerender(<Button variant="secondary">Secondary</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-white');

    rerender(<Button variant="danger">Danger</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-red-600');
  });
});
```

### Integration Testing

Cypress is used for integration testing.

```js
// cypress/e2e/student-management.cy.js
describe('Student Management', () => {
  beforeEach(() => {
    cy.login('admin@example.com', 'password');
    cy.visit('/students');
  });

  it('displays the student list', () => {
    cy.get('h1').should('contain', 'Students');
    cy.get('table').should('exist');
    cy.get('tbody tr').should('have.length.at.least', 1);
  });

  it('can add a new student', () => {
    cy.get('button').contains('Add Student').click();
    cy.url().should('include', '/students/new');

    // Fill out the form
    cy.get('input[name="name.first"]').type('John');
    cy.get('input[name="name.last"]').type('Doe');
    cy.get('input[name="email"]').type('john.doe@example.com');
    cy.get('input[name="phone"]').type('1234567890');
    // Fill out other required fields

    cy.get('button').contains('Save Student').click();

    // Verify success
    cy.url().should('include', '/students');
    cy.get('div').contains('Student added successfully').should('exist');
    cy.get('table').contains('John Doe').should('exist');
  });

  it('can view student details', () => {
    cy.get('table tbody tr').first().click();
    cy.url().should('match', /\/students\/[\w-]+$/);
    cy.get('h2').should('exist');
    cy.get('button').contains('Edit').should('exist');
  });

  it('can edit a student', () => {
    cy.get('table tbody tr').first().click();
    cy.get('button').contains('Edit').click();
    cy.url().should('match', /\/students\/[\w-]+\/edit$/);

    cy.get('input[name="phone"]').clear().type('9876543210');
    cy.get('button').contains('Save Student').click();

    cy.url().should('match', /\/students\/[\w-]+$/);
    cy.get('div').contains('Student updated successfully').should('exist');
    cy.get('div').contains('9876543210').should('exist');
  });
});
```

## Deployment Strategy

### Build Process

```json
// package.json (scripts section)
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "jest",
    "test:watch": "jest --watch",
    "cypress": "cypress open",
    "cypress:headless": "cypress run",
    "analyze": "ANALYZE=true next build"
  }
}
```

### Containerization

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

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000

CMD ["node", "server.js"]
```

## Conclusion

This frontend architecture provides a solid foundation for building a scalable, maintainable, and user-friendly educational institution management SaaS platform. The combination of React.js, Next.js, TypeScript, and Tailwind CSS offers a modern and efficient development experience, while the component-based design ensures reusability and consistency across the application.

The architecture is designed with multi-tenancy in mind, allowing for institution-specific customization of themes, branding, and data fields. Performance optimization techniques such as code splitting, memoization, and image optimization ensure a fast and responsive user experience, even as the application scales.

Accessibility is a core consideration, with all components designed to be keyboard navigable and screen reader friendly. The testing strategy includes both component-level unit tests and integration tests to ensure reliability and maintainability.

By following this architecture, developers can build a robust and feature-rich frontend for the educational institution management SaaS platform that meets the needs of diverse educational institutions.