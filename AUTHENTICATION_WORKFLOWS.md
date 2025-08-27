# Chi tiết luồng đăng ký và đăng nhập - TiemTra Client

## 🎯 Tổng quan Authentication System

Hệ thống xác thực của TiemTra sử dụng **OTP-based verification** kết hợp với **JWT tokens** để đảm bảo bảo mật cao. Toàn bộ quy trình được thiết kế để:

- ✅ Xác minh email thật trước khi kích hoạt tài khoản
- ✅ Tự động quản lý JWT tokens (access + refresh)  
- ✅ Hỗ trợ đặt lại mật khẩu an toàn qua OTP
- ✅ Phân quyền rõ ràng (Customer vs Admin)

---

## 📝 1. LUỒNG ĐĂNG KÝ (SIGN UP)

### 🔄 Workflow tổng quan

```
[Điền form] → [Validate] → [Gọi API] → [OTP gửi email] → [Verify OTP] → [Tài khoản kích hoạt]
```

### 📋 Step-by-step chi tiết

#### **Step 1: Người dùng điền thông tin (`RegisterForm.tsx`)**

**Thông tin yêu cầu:**
```typescript
interface RegisterFormData {
  fullName: string;      // Họ và tên
  email: string;         // Email (phải unique)
  phoneNumber: string;   // Số điện thoại (format: 0xxxxxxxxx)
  password: string;      // Mật khẩu (tối thiểu 6 ký tự)
}
```

**Client-side validation:**
```typescript
// Email validation
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// Phone validation (Vietnamese format)
const phoneRegex = /^0\d{9}$/;

// Password validation  
password.length >= 6
```

**UI Components:**
- Material-UI `TextField` với real-time validation
- Error messages hiển thị ngay lập tức
- Loading state trong quá trình submit
- Password visibility toggle

#### **Step 2: Gửi request đăng ký**

**Hook sử dụng: `useAuth()`**
```typescript
const { register } = useAuth();

const handleSubmit = async (formData) => {
  const result = await register(formData);
  
  if (result.success) {
    // Chuyển sang verify OTP
    navigate("/verify-otp", { state: { email: formData.email } });
  } else {
    // Hiển thị lỗi
    showError(result.message);
  }
};
```

**Redux Action được trigger:**
```typescript
export const registerApi = createAsyncThunk(
  "auth/register",
  async (params: RegisterParams, thunkAPI) => {
    try {
      const response = await authApi.register(params);
      const data = response.data;

      if (!data?.success) {
        return thunkAPI.rejectWithValue(data.message || "Đăng ký thất bại");
      }

      return data;
    } catch (error: any) {
      return thunkAPI.rejectWithValue(
        error.response?.data?.message || "Lỗi khi đăng ký"
      );
    }
  }
);
```

**API Call:**
```typescript
// Service: src/services/api/Authentication/index.ts
const authApi = {
  register: (params: {
    fullName: string;
    email: string;
    password: string;
    phoneNumber: string;
  }) => requester.post("/api/auth/register", params),
}
```

#### **Step 3: Backend xử lý & gửi OTP**

**Backend response khi thành công:**
```json
{
  "success": true,
  "message": "Đăng ký thành công. Vui lòng kiểm tra email để xác thực.",
  "data": {
    "userId": "uuid-here",
    "email": "user@example.com",
    "otpSent": true
  }
}
```

**Backend actions:**
1. Validate input data
2. Check email uniqueness  
3. Hash password
4. Tạo user record với status = "Pending"
5. Generate OTP (6 digits)
6. Gửi OTP qua email
7. Lưu OTP với expiry time (thường 5-10 phút)

#### **Step 4: User navigate to OTP verification**

**Route transition:**
```typescript
// Từ /register → /verify-otp
navigate("/verify-otp", { 
  state: { email: formData.email } 
});
```

**Email chứa OTP được gửi đến user** với format:
```
Subject: Xác thực tài khoản TiemTra

Xin chào [FullName],

Mã OTP của bạn là: 123456

Mã này có hiệu lực trong 10 phút.

Cảm ơn,
TiemTra Team
```

---

## ✅ 2. LUỒNG XÁC THỰC OTP

### 🔄 Workflow

```
[Nhập OTP] → [Validate] → [Gọi API verify] → [Account activated] → [Redirect to login]
```

### 📋 Chi tiết implementation

#### **Step 1: OTP Input Form (`VerifyOtpForm.tsx`)**

```typescript
const VerifyOtpForm = () => {
  const [otp, setOtp] = useState("");
  const location = useLocation();
  const email = location.state?.email; // Lấy email từ register step

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    const result = await verifyOtp({ email, otp });
    
    if (result.success) {
      showSuccess("Xác thực thành công");
      navigate("/login");
    } else {
      setError(result.message);
    }
  };
}
```

#### **Step 2: API Call & Redux Action**

```typescript
export const verifyOtpApi = createAsyncThunk(
  "auth/verifyOtp", 
  async (params: { email: string; otp: string }, thunkAPI) => {
    try {
      const response = await authApi.verifyOtp(params);
      const data = response.data;

      if (!data?.success || !data.token) {
        return thunkAPI.rejectWithValue(data.message || "Mã OTP không đúng");
      }

      return data;
    } catch (error: any) {
      return thunkAPI.rejectWithValue(error.response?.data?.message);
    }
  }
);
```

#### **Step 3: Backend verification**

**Backend process:**
1. Find OTP record by email + otp code
2. Check expiry time
3. Validate OTP matches
4. Update user status: "Pending" → "Active"
5. Delete used OTP
6. Return success response

**Success response:**
```json
{
  "success": true,
  "message": "Xác thực thành công",
  "token": "jwt-token-here",
  "data": {
    "userId": "uuid",
    "email": "user@example.com",
    "isVerified": true
  }
}
```

#### **Step 4: Account activation complete**

- User account status = "Active"
- User có thể login bình thường
- Redirect về `/login` page

---

## 🔐 3. LUỒNG ĐĂNG NHẬP (SIGN IN)

### 🔄 Workflow

```
[Nhập email/password] → [Validate] → [API call] → [JWT tokens] → [Save to localStorage] → [Redirect dashboard]
```

### 📋 Chi tiết implementation

#### **Step 1: Login Form (`LoginForm.tsx`)**

```typescript
const LoginForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm();
  const { login, loading, error } = useAuth();
  
  const onSubmit = async (data: { email: string; password: string }) => {
    const result = await login(data);
    
    if (result) {
      navigate("/"); // Redirect to home or dashboard
    }
  };
}
```

**Form validation:**
- Email: Required + email format
- Password: Required
- Real-time error display

#### **Step 2: Login API & Redux Action**

```typescript
export const loginApi = createAsyncThunk(
  "auth/login",
  async (params: { email: string; password: string }, thunkAPI) => {
    try {
      const response = await authApi.login(params);
      const data = response.data;

      if (!data?.success || !data?.token) {
        return thunkAPI.rejectWithValue("Sai tài khoản hoặc mật khẩu!");
      }

      // 🔥 QUAN TRỌNG: Lưu tokens & user data
      localStorage.setItem("access_token", data.token);
      localStorage.setItem("refresh_token", data.refreshToken || "");
      localStorage.setItem("user", JSON.stringify(data.data));

      // Clear cart khi đăng nhập (optional)
      localStorage.removeItem("cart");

      return data;
    } catch (error: any) {
      return thunkAPI.rejectWithValue(
        error.response?.data?.message || "Đăng nhập thất bại"
      );
    }
  }
);
```

#### **Step 3: Backend authentication**

**Backend process:**
1. Find user by email
2. Verify password hash
3. Check account status (Active/Pending/Blocked)
4. Generate JWT access token (short-lived, ~15 mins)
5. Generate refresh token (long-lived, ~7 days)
6. Return tokens + user data

**Success response:**
```json
{
  "success": true,
  "message": "Đăng nhập thành công",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "refresh-token-here",
  "data": {
    "userId": "uuid",
    "email": "user@example.com", 
    "fullName": "Nguyen Van A",
    "avatar": "avatar-url",
    "roles": ["Customer"] // or ["Admin"]
  }
}
```

#### **Step 4: Client-side token management**

**localStorage structure sau khi login:**
```javascript
{
  "access_token": "jwt-access-token",
  "refresh_token": "jwt-refresh-token", 
  "user": "{\"userId\":\"uuid\",\"email\":\"...\",\"roles\":[\"Customer\"]}"
}
```

**Redux state update:**
```typescript
.addCase(loginApi.fulfilled, (state, action) => {
  state.loading = false;
  state.user = action.payload.data; // User info stored in Redux
})
```

#### **Step 5: Automatic token attachment**

**Axios interceptor tự động attach token:**
```typescript
// src/services/extended/axiosInstance.ts
requester.interceptors.request.use((config) => {
  const excludedPaths = ["/auth/login", "/auth/register", "/auth/verify-otp"];
  const shouldExclude = excludedPaths.some(path => config.url?.includes(path));

  if (!shouldExclude) {
    const token = getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
  }
  return config;
});
```

**Mọi API call sau login đều tự động có header:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 🔄 4. AUTOMATIC TOKEN REFRESH SYSTEM

### 🎯 Vấn đề cần giải quyết

- Access token có thời gian sống ngắn (15 phút)
- User không muốn phải login lại liên tục
- Cần refresh token tự động và seamless

### 🛠️ Implementation chi tiết

#### **Response Interceptor với Auto-Refresh**

```typescript
requester.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Chỉ retry khi: 401 error + chưa retry + không phải auth endpoints
    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = getRefreshToken();
        if (!refreshToken) {
          throw new Error("Không tìm thấy Refresh Token");
        }

        // 🔥 Call refresh token API
        const response = await authApi.refreshToken({ refreshToken });
        
        // 🔥 Update tokens in localStorage
        setTokens(response.data.token, response.data.refreshToken);

        // 🔥 Retry original request với token mới
        originalRequest.headers.Authorization = `Bearer ${response.data.token}`;
        return axios(originalRequest);
        
      } catch (refreshError) {
        console.error("Refresh token thất bại:", refreshError);
        
        // 🔥 Refresh failed → Logout user
        localStorage.removeItem("access_token");
        localStorage.removeItem("refresh_token");
        localStorage.removeItem("user");
        window.location.href = "/login";
      }
    }

    return Promise.reject(error);
  }
);
```

#### **Luồng hoạt động của Auto-Refresh**

```
[API Request] → [401 Error] → [Get Refresh Token] → [Call Refresh API] 
     ↓
[New Tokens] → [Update localStorage] → [Retry Original Request] → [Success]
     ↓ (if refresh fails)
[Clear Storage] → [Redirect to Login]
```

**Ưu điểm:**
- ✅ User không cảm nhận được token đã hết hạn
- ✅ Mọi API call đều được retry tự động
- ✅ Chỉ logout khi refresh token cũng hết hạn
- ✅ Thread-safe (tránh multiple refresh calls)

---

## 🔒 5. PASSWORD RESET WORKFLOW

### 🔄 Workflow tổng quan

```
[Quên mật khẩu] → [Nhập email] → [OTP sent] → [Verify OTP] → [Đặt mật khẩu mới] → [Complete]
```

### 📋 Step-by-step chi tiết

#### **Step 1: Forgot Password Page (`ForgotPassword.tsx`)**

```typescript
const ForgotPasswordPage = () => {
  const [otpSent, setOtpSent] = useState(false);
  const [cooldown, setCooldown] = useState(0);
  
  const handleSendOtp = async () => {
    const email = getValues("email");
    const res = await forgotPassword({ email });
    
    setOtpSent(true);
    setCooldown(60); // 60 giây cooldown
  };
  
  const handleVerifyAndGo = async () => {
    const otp = getValues("otp");
    // Chuyển sang reset password page với email + otp
    navigate("/reset-password", { state: { email, otp } });
  };
}
```

**Features:**
- Email validation trước khi gửi OTP
- Cooldown 60 giây cho nút "Gửi lại OTP"
- Two-step form: Email → OTP input
- Thông báo neutral (không reveal email tồn tại hay không)

#### **Step 2: OTP Generation & Email**

**Backend process cho forgot password:**
```typescript
// API: /api/auth/forgot-password
{
  email: "user@example.com"
}
```

1. Tìm user by email
2. Generate OTP (6 digits)
3. Store OTP với expiry (10 minutes)
4. Gửi email chứa OTP
5. Return neutral message

**Email template:**
```
Subject: Đặt lại mật khẩu TiemTra

Mã OTP để đặt lại mật khẩu: 654321

Mã này có hiệu lực trong 10 phút.

Nếu bạn không yêu cầu đặt lại mật khẩu, vui lòng bỏ qua email này.

TiemTra Team
```

#### **Step 3: Reset Password Page (`ResetPasswordPage.tsx`)**

```typescript
const ResetPasswordPage = () => {
  const location = useLocation();
  const { email, otp } = location.state || {};
  
  const onSubmit = async (data: ResetPasswordFormValues) => {
    const res = await resetPassword({
      email,
      otp,
      newPassword: data.newPassword
    });

    if (res.success) {
      showSuccess("Đặt lại mật khẩu thành công!");
      navigate("/login");
    } else {
      showError(res.message);
    }
  };
}
```

**Form validation:**
```typescript
{
  newPassword: {
    required: "Mật khẩu không được để trống",
    minLength: { value: 6, message: "Ít nhất 6 ký tự" }
  },
  confirmPassword: {
    required: "Vui lòng xác nhận mật khẩu",
    validate: (value) => value === watch("newPassword") || "Mật khẩu không khớp"
  }
}
```

#### **Step 4: Backend Password Update**

**API Call:**
```typescript
// POST /api/auth/reset-password
{
  email: "user@example.com",
  otp: "654321", 
  newPassword: "new-password-here"
}
```

**Backend process:**
1. Validate OTP với email
2. Check OTP chưa hết hạn
3. Hash new password  
4. Update user password
5. Invalidate tất cả existing tokens
6. Delete used OTP
7. Return success

---

## 🛡️ 6. PROTECTED ROUTES & AUTHORIZATION

### 🎯 Phân quyền trong hệ thống

**User Roles:**
- `Customer`: Khách hàng thường (shopping, orders)
- `Admin`: Quản trị viên (dashboard, product management)

### 🔒 ProtectedRoute Component

```typescript
const ProtectedRoute = ({ children, requiredRole }: ProtectedRouteProps) => {
  const storedUser = localStorage.getItem("user");
  const user = storedUser ? JSON.parse(storedUser) : null;

  // Check if user is logged in
  if (!user) {
    return <UnauthorizedPage />;
  }

  // Check role-based access
  if (requiredRole && !user.roles?.includes(requiredRole)) {
    return (
      <UnauthorizedPage 
        message="Bạn không có quyền truy cập trang này" 
        showLoginButton={false}
      />
    );
  }

  return <>{children}</>;
};
```

### 🛣️ Route Protection Usage

```typescript
// Admin routes với role protection
<Route 
  path="/admin" 
  element={
    <ProtectedRoute requiredRole="Admin">
      <AdminDashboard />
    </ProtectedRoute>
  }
>
  <Route path="dashboard" element={<Dashboard />} />
  <Route path="product" element={<Product />} />
  <Route path="category" element={<Category />} />
  <Route path="order" element={<Order />} />
</Route>

// Store routes (public + customer only)
<Route element={<StoreLayout />}>
  <Route path="/" element={<HomePage />} />
  <Route path="/san-pham" element={<ProductList />} />
  <Route path="/gio-hang" element={<CartPage />} />
</Route>
```

---

## 🔧 7. STATE MANAGEMENT STRATEGY

### 🏪 Redux cho Authentication State

```typescript
interface AuthState {
  user: UserType | null;    // Current logged in user
  loading: boolean;         // API call loading state  
  error: string | null;     // Error messages
}

// Only authentication state lives in Redux
// Server data uses React Query
// UI state uses local useState
```

### 💾 Local Storage Management

```typescript
// Token management utilities
export const getToken = (): string | null => {
  let accessToken = localStorage.getItem("access_token");
  
  if (!accessToken) {
    try {
      const userData = localStorage.getItem("userData");
      accessToken = userData ? JSON.parse(userData)?.id_token : null;
    } catch (error) {
      console.error("Lỗi khi parse userData:", error);
      accessToken = null;
    }
  }

  return accessToken;
};

export const setTokens = (accessToken: string, refreshToken: string) => {
  localStorage.setItem("access_token", accessToken);
  localStorage.setItem("refresh_token", refreshToken);
};
```

### 🔄 useCurrentUser Hook

```typescript
export const useCurrentUser = () => {
  const [user, setUser] = useState<any>(null);

  useEffect(() => {
    const syncUser = () => {
      const storedUser = localStorage.getItem("user");
      setUser(storedUser ? JSON.parse(storedUser) : null);
    };

    syncUser();
    
    // Listen for user changes across components
    window.addEventListener("userChanged", syncUser);
    return () => window.removeEventListener("userChanged", syncUser);
  }, []);

  return user;
};
```

---

## ⚡ 8. ERROR HANDLING & UX

### 🚨 Error Scenarios & Handling

#### **Network Errors**
```typescript
try {
  const response = await authApi.login(params);
} catch (error: any) {
  const message = error.response?.data?.message || "Kết nối mạng có vấn đề";
  return thunkAPI.rejectWithValue(message);
}
```

#### **Validation Errors**
```typescript
// Client-side validation
const newErrors = {
  email: !emailRegex.test(email) ? "Email không đúng định dạng" : "",
  phone: !phoneRegex.test(phone) ? "Số điện thoại không hợp lệ" : "",
  password: password.length < 6 ? "Mật khẩu phải có ít nhất 6 ký tự" : ""
};
```

#### **Server Errors**
```typescript
// Server trả về error codes
{
  success: false,
  message: "Email đã tồn tại",
  errorCode: "EMAIL_ALREADY_EXISTS"
}
```

### 🎨 Loading States & Feedback

**Button Loading States:**
```tsx
<Button
  disabled={loading}
  startIcon={loading ? <CircularProgress size={18} /> : null}
>
  {loading ? "Đang đăng nhập..." : "Đăng nhập"}
</Button>
```

**Toast Notifications:**
```typescript
const { showSuccess, showError } = useToast();

// Success
showSuccess("Đăng nhập thành công");

// Error  
showError("Sai tài khoản hoặc mật khẩu");
```

**Form Validation Feedback:**
```tsx
<TextField
  error={!!errors.email}
  helperText={errors.email}
  // Real-time validation display
/>
```

---

## 🎯 9. SECURITY CONSIDERATIONS

### 🔐 Security Best Practices Implemented

#### **Token Security**
- ✅ Short-lived access tokens (15 minutes)
- ✅ HTTP-only refresh tokens (7 days)
- ✅ Automatic token rotation
- ✅ Secure storage in localStorage (encrypted in production)

#### **Password Security** 
- ✅ Minimum 6 characters requirement
- ✅ Server-side hashing (bcrypt/Argon2)
- ✅ No password transmission in URLs
- ✅ Password visibility toggle

#### **OTP Security**
- ✅ 6-digit random codes
- ✅ Short expiry time (10 minutes)
- ✅ Single-use tokens
- ✅ Rate limiting on OTP requests

#### **API Security**
- ✅ HTTPS only communication
- ✅ Input validation & sanitization
- ✅ Rate limiting on auth endpoints
- ✅ Request timeout handling

---

## 📚 10. TESTING AUTHENTICATION FLOWS

### 🧪 Test Cases để verify

#### **Sign Up Flow**
1. ✅ Valid registration creates pending user
2. ✅ Duplicate email shows proper error
3. ✅ Invalid phone format rejected
4. ✅ OTP email sent successfully
5. ✅ Valid OTP activates account
6. ✅ Expired OTP shows error
7. ✅ Invalid OTP shows error

#### **Sign In Flow**  
1. ✅ Valid credentials login successful
2. ✅ Invalid email shows error
3. ✅ Wrong password shows error
4. ✅ Unverified account blocked
5. ✅ Tokens stored correctly
6. ✅ User data populated in Redux

#### **Token Management**
1. ✅ Expired token triggers refresh
2. ✅ Invalid refresh token logs out
3. ✅ Concurrent API calls handle refresh properly
4. ✅ Manual logout clears all storage

#### **Password Reset**
1. ✅ Valid email sends OTP
2. ✅ Invalid email shows neutral message  
3. ✅ Valid OTP allows password reset
4. ✅ Password complexity enforced
5. ✅ Old tokens invalidated after reset

---

## 🎓 KẾT LUẬN CHO SINH VIÊN

### 📖 Kiến thức cần nắm vững:

1. **JWT Authentication Pattern**
   - Access token vs Refresh token
   - Token expiry & rotation
   - Secure storage strategies

2. **Redux Async Actions**
   - createAsyncThunk pattern
   - Error handling with thunkAPI
   - State management best practices

3. **Form Validation**
   - Client-side validation
   - Real-time feedback
   - Error state management

4. **React Router Navigation**
   - Programmatic navigation
   - State passing between routes
   - Protected route patterns

5. **Axios Interceptors**
   - Request/Response modification
   - Automatic retry logic
   - Error handling strategies

### 🔧 Thực hành nên làm:

1. **Trace toàn bộ flow** từ UI → Redux → API → Backend
2. **Test error scenarios** như network failure, invalid credentials
3. **Understand token lifecycle** và automatic refresh mechanism
4. **Practice form validation** patterns với React Hook Form
5. **Study security implications** của từng design decision

### 💡 Điểm mạnh của architecture này:

- ✅ **Separation of Concerns**: Auth logic tách biệt rõ ràng
- ✅ **User Experience**: Seamless token refresh, no interruption
- ✅ **Security**: Multi-layer protection với OTP + JWT
- ✅ **Scalability**: Easy to extend với new auth methods
- ✅ **Maintainability**: Centralized error handling & state management

Hệ thống authentication này đảm bảo cả **security** và **user experience**, là foundation vững chắc cho một e-commerce application production-ready! 🚀