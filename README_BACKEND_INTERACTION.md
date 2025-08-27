# Hướng dẫn tương tác với Backend - Dự án TiemTra

## 📋 Tổng quan Architecture

Dự án này là một ứng dụng React TypeScript phục vụ hai mục đích chính:
- **Giao diện khách hàng**: Trang web bán hàng trà 
- **Giao diện quản trị**: Dashboard quản lý sản phẩm, đơn hàng, khách hàng

## 🌐 Cấu hình kết nối Backend

### URL Base
```typescript
// Development
const API_URL = "https://localhost:7021/api"

// Production (qua nginx proxy)
/api/ -> https://tiemtra-api-f5eue2bseraxefh8.southeastasia-01.azurewebsites.net
```

### Cấu trúc URL API
```
/api/auth/*          - Authentication endpoints
/api/admin/*         - Admin management endpoints  
/api/store/*         - Store/Customer endpoints
```

## 🔧 Hệ thống Service Layer

### 1. Axios Instance (`src/services/extended/axiosInstance.ts`)

Đây là **trái tim** của việc giao tiếp với backend:

```typescript
const requester = axios.create({
  baseURL: API_URL,
  headers: { "Content-Type": "application/json" }
});

// Request Interceptor - Tự động thêm token
requester.interceptors.request.use((config) => {
  const excludedPaths = ["/auth/login", "/auth/register", "/auth/verify-otp"];
  if (!shouldExclude) {
    const token = getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
  }
  return config;
});

// Response Interceptor - Tự động refresh token
requester.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response.status === 401 && !originalRequest._retry) {
      // Thử refresh token
      const refreshToken = getRefreshToken();
      const response = await authApi.refreshToken({ refreshToken });
      setTokens(response.data.token, response.data.refreshToken);
      
      // Retry request với token mới
      originalRequest.headers.Authorization = `Bearer ${response.data.token}`;
      return axios(originalRequest);
    }
    return Promise.reject(error);
  }
);
```

**Tính năng chính:**
- ✅ Tự động attach JWT token vào mọi request (trừ auth endpoints)
- ✅ Tự động refresh token khi hết hạn (401 error)
- ✅ Redirect về login page khi refresh thất bại

### 2. Constants Definition (`src/domain/constants/index.ts`)

Tập trung quản lý tất cả URL endpoints:

```typescript
const URL_ADMIN = `${BASE_URL}/api/admin`
const URL_AUTH = `${BASE_URL}/api`  
const URL_STORE = `${BASE_URL}/api/store`

export const AUTHENTICATION = {
  URL_API: {
    LOGIN_API: `${URL_AUTH}/auth/login`,
    REGISTER_API: `${URL_AUTH}/auth/register`,
    VERIFY_OTP: `${URL_AUTH}/auth/verify-otp`,
    REFRESH_TOKEN_API: `${URL_AUTH}/auth/refresh-token`,
    FORGOT_PASSWORD_API: `${URL_AUTH}/auth/forgot-password`,
    RESET_PASSWORD_API: `${URL_AUTH}/auth/reset-password`,
  }
}
```

**Lợi ích:**
- ✅ Tập trung quản lý URL, dễ maintain
- ✅ TypeScript support với dynamic URL functions  
- ✅ Phân chia rõ ràng theo domain (admin/store/auth)

## 🏗️ Chi tiết từng Service Module

### 1. Authentication Service (`src/services/api/Authentication/index.ts`)

```typescript
const authApi = {
  login: (params: { email: string; password: string }) =>
    requester.post(AUTHENTICATION.URL_API.LOGIN_API, params),
    
  register: (params: { fullName: string; email: string; password: string; phoneNumber: string }) =>
    requester.post(AUTHENTICATION.URL_API.REGISTER_API, params),
    
  verifyOtp: (params: { email: string; otp: string }) =>
    requester.post(AUTHENTICATION.URL_API.VERIFY_OTP, params),
    
  refreshToken: (params: { refreshToken: string }) =>
    requester.post(AUTHENTICATION.URL_API.REFRESH_TOKEN_API, params),
    
  forgotPassword: (params: { email: string }) =>
    requester.post(AUTHENTICATION.URL_API.FORGOT_PASSWORD_API, params),
    
  resetPassword: (params: { email: string; otp: string; newPassword: string }) =>
    requester.post(AUTHENTICATION.URL_API.RESET_PASSWORD_API, params),
};
```

**Workflow xác thực:**
1. `login` → Nhận token + refresh token + user data
2. Lưu vào localStorage: `access_token`, `refresh_token`, `user`
3. Middleware tự động attach token vào requests
4. Token hết hạn → Tự động refresh
5. Refresh thất bại → Redirect login

### 2. Product Service (`src/services/api/Products/indext.tsx`)

**Admin Product Operations:**
```typescript
const productApi = {
  // Tạo mã sản phẩm tự động
  generateProductCode: () => 
    requester.get(ADMIN_PRODUCT.URL_API.GENERATE_PRODUCT_CODE),
    
  // Upload ảnh sản phẩm
  uploadImage: (file: File) => {
    const formData = new FormData();
    formData.append("file", file);
    return requester.post(ADMIN_PRODUCT.URL_API.UP_PRODUCT_IMAGE, formData, {
      headers: { "Content-Type": "multipart/form-data" }
    });
  },
  
  // CRUD operations
  createProduct: (data: CreateProductRequest) =>
    requester.post(ADMIN_PRODUCT.URL_API.CREATE_PRODUCT, data),
    
  getPagingProduct: (params: IProductFilter) => 
    requester.get(ADMIN_PRODUCT.URL_API.GET_ALL_PRODUCT, { params }),
    
  updateProduct: (id: string, data: CreateProductRequest) =>
    requester.put(ADMIN_PRODUCT.URL_API.UPDATE_PRODUCT(id), data),
    
  getByIdApi: (params: ProductRequest) => 
    requester.get(ADMIN_PRODUCT.URL_API.GET_BY_ID, { params }),
}
```

**Store Product Operations:**
```typescript
// Khách hàng xem sản phẩm
storeGetAllProduct: (params: IProductFilter) => 
  requester.get(STORE_PPRODUCT.URL_API.GET_ALL_PRODUCT, { params }),

storeGetProductByCode: (productCode: string) => 
  requester.get(STORE_PPRODUCT.URL_API.GET_PRODUCT_BY_CODE(productCode))
```

**Interface chính:**
```typescript
interface CreateProductRequest {
  productCode: string;
  productName: string;
  privewImageUrl: string;
  price?: number | null;
  stock?: number | null;
  description?: string;
  categoryId?: number;
  productImageUrls: string[];
  productAttributes?: ProductAttribute[];
  productVariations?: ProductVariation[];
  productStatus?: ProductStatus;
}

enum ProductStatus {
  Draft = 0,        // Nháp
  Active = 1,       // Đang bán
  Inactive = 2,     // Ngừng bán
  OutOfStock = 3,   // Hết hàng
}
```

### 3. Category Service (`src/services/api/Category/index.tsx`)

```typescript
const categoryApi = {
  // Admin category management
  getPagingApi: (params: ICategoryRequest) =>
    requester.get(CATEGORY.URL_API.GET_ALL_API, { params }),
    
  addCategoryApi: (data: IAddCategoryRequest) =>
    requester.post(CATEGORY.URL_API.CREATE_API, data),
    
  updateCategoryApi: (id: number, data: IAddCategoryRequest) =>
    requester.put(CATEGORY.URL_API.UPDATE_API(id), data),
    
  // Bulk operations
  checkCanDeleteManyCategories: (ids: number[]) =>
    requester.post(CATEGORY.URL_API.CHECK_DELETE_BY_IDS, ids),
    
  deleteManyCategories: (ids: number[]) =>
    requester.delete(CATEGORY.URL_API.DELETE_API_BY_IDS, { data: ids }),
    
  // Category attributes relationship
  getAttributeById: (id: number) =>
    requester.get(CATEGORY.URL_API.ATTRIBUTE_BY_ID(id)),
    
  setAttributesForCategory: (data: { categoryId: number; attributeIds: number[] }) =>
    requester.post(CATEGORY.URL_API.SET_ATTRIBUTES_API, data),
    
  // Store-facing APIs
  getAllTree: () => 
    requester.get(STORE_CATEGORY.URL_API.GET_ALL), // Tree structure cho menu
};
```

### 4. Cart Service (`src/services/api/Cart/index.tsx`)

```typescript
const cartApi = {
  viewCart: () => 
    requester.get(STORE_CART.URL_API.VIEW_CART),
    
  addProductToCart: (data: AddProductToCart) =>
    requester.post(STORE_CART.URL_API.ADD_PRODUCT_TO_CART, data),
    
  updateQuantity: (data: AddProductToCart) =>
    requester.put(STORE_CART.URL_API.UPDATE_QUANTITY, data),
    
  removeCartItem: (cartItemId: string) =>
    requester.delete(STORE_CART.URL_API.REMOVE_CART_ITEM, {
      params: { cartItemId }
    }),
    
  getTotalQuantity: () => 
    requester.get(STORE_CART.URL_API.GET_TOTAL_QUANTITY),
};
```

**Cart Data Models:**
```typescript
interface ICartItem {
  cartItemId: string;
  productId: string;
  productVariationId: string;
  productCode: string;
  productName: string;
  productVariationName?: string;
  previewImage?: string;
  price: number;
  quantity: number;
}

interface AddProductToCart {
  productId: string;
  productVariationId?: string | null;
  quantity: number;
}
```

### 5. Order Service (`src/services/api/Order/index.tsx`)

```typescript
const orderApi = {
  // Store operations
  generateOrderCode: () => 
    requester.get(STORE_ORDER.URL_API.GENERATE_ORDER_CODE),
    
  createOrder: (data: ICreateOrder) =>
    requester.post(STORE_ORDER.URL_API.CREATE_ORDER, data),
    
  // Admin operations  
  getPagingOrder: (params: IOrderFilter) =>
    requester.get(ADMIN_ORDER.URL_API.GET_PAGING, { params }),
    
  comfirmOrder: (orderId: string) =>
    requester.post(ADMIN_ORDER.URL_API.COMFIRM_ORDER(orderId)),
    
  changeOrderStatus: (orderId: string, orderStatus: OrderStatus) =>
    requester.put(ADMIN_ORDER.URL_API.CHANGE_ORDER_STATUS(orderId), {
      newStatus: orderStatus
    }),
    
  getById: (orderId: string) =>
    requester.get(ADMIN_ORDER.URL_API.GET_BY_ID(orderId)),
};
```

**Order Status Workflow:**
```typescript
enum OrderStatus {
  Pending = 0,           // Đang chờ xử lý
  Confirmed = 1,         // Đã xác nhận  
  Shipped = 2,           // Đang giao hàng
  Delivered = 3,         // Đã giao hàng thành công
  DeliveryFailed = 4,    // Giao hàng không thành công
  CancelledByShop = 5,   // Shop hủy
  CancelledByUser = 6,   // Khách hàng hủy
  Refunded = 7,          // Đã hoàn tiền
}

enum PaymentMethod {
  COD = 0,              // Thanh toán khi nhận hàng
  BankTransfer = 1,     // Chuyển khoản
}
```

## 🔐 Authentication Flow chi tiết

### 1. Redux Store Setup (`src/views/Auth/store/index.tsx`)

```typescript
// Async thunks cho authentication
export const loginApi = createAsyncThunk(
  "auth/login",
  async (params: { email: string; password: string }, thunkAPI) => {
    try {
      const response = await authApi.login(params);
      const data = response.data;

      if (!data?.success || !data?.token) {
        return thunkAPI.rejectWithValue("Sai tài khoản hoặc mật khẩu!");
      }

      // Lưu tokens và user data
      localStorage.setItem("access_token", data.token);
      localStorage.setItem("refresh_token", data.refreshToken || "");
      localStorage.setItem("user", JSON.stringify(data.data));
      
      // Clear cart khi đăng nhập
      localStorage.removeItem("cart");

      return data;
    } catch (error: any) {
      return thunkAPI.rejectWithValue(error.response?.data?.message);
    }
  }
);
```

### 2. Token Management

**Lưu trữ:**
```typescript
// localStorage keys:
- "access_token": JWT token chính
- "refresh_token": Token để refresh
- "user": User data object
```

**Luồng refresh token:**
```typescript
// Khi API trả 401
if (error.response.status === 401 && !originalRequest._retry) {
  originalRequest._retry = true;
  
  try {
    const refreshToken = getRefreshToken();
    const response = await authApi.refreshToken({ refreshToken });
    
    // Cập nhật tokens mới
    setTokens(response.data.token, response.data.refreshToken);
    
    // Retry request gốc với token mới
    originalRequest.headers.Authorization = `Bearer ${response.data.token}`;
    return axios(originalRequest);
  } catch (refreshError) {
    // Refresh thất bại → Logout + redirect login
    localStorage.clear();
    window.location.href = "/login";
  }
}
```

### 3. Protected Routes

```typescript
const ProtectedRoute = ({ children, requiredRole }: ProtectedRouteProps) => {
  const storedUser = localStorage.getItem("user");
  const user = storedUser ? JSON.parse(storedUser) : null;

  if (!user) {
    return <UnauthorizedPage />;
  }

  if (requiredRole && !user.roles?.includes(requiredRole)) {
    return <UnauthorizedPage message="Bạn không có quyền truy cập" />;
  }

  return <>{children}</>;
};

// Usage:
<Route 
  path="/admin" 
  element={
    <ProtectedRoute requiredRole="Admin">
      <AdminDashboard />
    </ProtectedRoute>
  }
/>
```

## 📊 State Management Strategy

### 1. Authentication State (Redux)
```typescript
// Chỉ auth state dùng Redux
interface AuthState {
  user: UserType | null;
  loading: boolean;
  error: string | null;
}
```

### 2. Server State (React Query)
```typescript
// Tất cả data từ server dùng React Query
const { data: products, isLoading } = useQuery({
  queryKey: ['products', filters],
  queryFn: () => productApi.getPagingProduct(filters)
});
```

### 3. Local State (localStorage)
```typescript
// Cart data lưu local (cho guest users)
const cartData = localStorage.getItem("cart");
```

## 🛠️ Patterns và Best Practices

### 1. Error Handling
```typescript
// Tất cả API calls có try-catch
try {
  const response = await productApi.createProduct(data);
  // Success handling
} catch (error: any) {
  const message = error.response?.data?.message || "Có lỗi xảy ra";
  // Error handling với user-friendly message
}
```

### 2. Loading States
```typescript
// Dùng React Query loading states
const { data, isLoading, error } = useQuery(/*...*/);

if (isLoading) return <CircularProgress />;
if (error) return <ErrorMessage />;
```

### 3. Type Safety
```typescript
// Tất cả API calls có TypeScript interfaces
interface IProductFilter {
  pageNumber?: number;
  pageSize?: number;
  keyword?: string;
  categoryId?: number;
  status?: ProductStatus;
}
```

### 4. File Upload Pattern
```typescript
const uploadImage = async (file: File) => {
  const formData = new FormData();
  formData.append("file", file);
  
  return requester.post(ADMIN_PRODUCT.URL_API.UP_PRODUCT_IMAGE, formData, {
    headers: { "Content-Type": "multipart/form-data" }
  });
};
```

## 🎯 Kết luận

Hệ thống này được thiết kế với:
- ✅ **Separation of Concerns**: Admin vs Store APIs rõ ràng
- ✅ **Automatic Token Management**: Không cần thủ công handle tokens
- ✅ **Type Safety**: Full TypeScript support
- ✅ **Error Resilience**: Auto retry với refresh token
- ✅ **Scalable Architecture**: Dễ extend thêm services

**Để học tốt project này, sinh viên nên:**
1. 📖 Đọc kỹ `axiosInstance.ts` để hiểu token management
2. 🔍 Trace flow từ UI component → Service → API → Backend  
3. 🧪 Test authentication flow: login → token refresh → logout
4. 📝 Practice tạo service mới theo pattern có sẵn
5. 🔧 Hiểu cách Redux + React Query complement nhau
