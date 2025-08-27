# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Starting Development
```bash
npm start         # Start development server on http://localhost:3000
# or
npm run dev       # Same as above
```

### Building
```bash
npm run build     # Build production bundle to /build folder
```

### Testing
```bash
npm test          # Run tests with Jest/React Testing Library in watch mode
```

### Docker Deployment
```bash
docker build -t tiemtra-client .    # Build Docker image
docker run -p 80:80 tiemtra-client  # Run container
```

## Architecture Overview

This is a dual-purpose React application serving both an **e-commerce storefront** and an **admin dashboard** for a Vietnamese tea store (TiemTra).

### Dual Layout System
- **StoreLayout**: Customer-facing e-commerce interface with Header/Footer
- **DashboardLayout**: Admin interface with Sidebar/Topbar navigation
- Routes are split using `/admin/*` prefix for admin routes vs root-level for store

### Key Technologies
- **React 18** with TypeScript
- **Material-UI (MUI)** + **Mantine** for UI components  
- **Redux Toolkit** for authentication state management
- **React Query (@tanstack/react-query)** for server state management
- **React Router 6** for routing
- **Axios** with interceptors for API calls
- **CKEditor 5** for rich text editing
- **React Hook Form** for form management

### API Configuration
- Base API URL: `https://localhost:7021/api` (development)
- Production API proxy configured in nginx.conf to Azure backend
- JWT authentication with automatic token refresh
- Axios interceptors handle token attachment and refresh logic

### State Management Pattern
- **Redux Store**: Authentication state only (user, loading, error)
- **React Query**: All server data fetching and caching
- **Local Storage**: Cart data, tokens, user session persistence

### Route Structure
- **Store Routes** (StoreLayout):
  - `/` - Home page
  - `/san-pham` - Product listing 
  - `/san-pham/:code/:slug` - Product details
  - `/gio-hang` - Shopping cart
  - `/thanh-toan` - Checkout
  - Auth routes: `/login`, `/register`, `/verify-otp`, etc.

- **Admin Routes** (DashboardLayout + ProtectedRoute):
  - `/admin/dashboard` - Analytics dashboard
  - `/admin/product` - Product management
  - `/admin/category` - Category management  
  - `/admin/attribute` - Product attributes
  - `/admin/order` - Order management
  - `/admin/customer` - Customer management

### Authentication Flow
1. Login via Redux action (`loginApi`)
2. Tokens stored in localStorage
3. Axios interceptor attaches Bearer token to requests
4. 401 responses trigger automatic token refresh
5. Failed refresh redirects to login page
6. Admin routes protected by `ProtectedRoute` component requiring "Admin" role

### File Organization
- `/src/views/Store/*` - Customer-facing pages and components
- `/src/views/Admin/*` - Admin dashboard pages and components  
- `/src/views/Auth/*` - Authentication pages with Redux store
- `/src/services/api/*` - API service modules organized by domain
- `/src/components/Layouts/*` - Shared layout components for both admin and store
- `/src/utils/*` - Utility functions for formatting, validation, etc.

### Custom Webpack Configuration
- `config-overrides.js` handles SVG loading conflicts between app and CKEditor
- CKEditor SVGs use file-loader, app SVGs use @svgr/webpack

### Vietnamese Localization
- Vietnamese language used throughout the UI
- Custom utility functions for Vietnamese time formatting
- Vietnamese route paths (e.g., `/san-pham`, `/gio-hang`)

## Development Notes

- The codebase serves a Vietnamese tea e-commerce platform
- Admin features include full CRUD operations for products, categories, orders, and customers
- Store features include product browsing, cart management, and checkout
- Authentication supports OTP verification and password reset flows
- Rich product management with attributes, variations, and image galleries