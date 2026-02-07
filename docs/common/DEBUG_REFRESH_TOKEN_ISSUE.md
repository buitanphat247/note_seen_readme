# Debug: "Refresh token không hợp lệ hoặc không tồn tại"

## Vấn Đề
Refresh token đang fail liên tục với lỗi "Refresh token không hợp lệ hoặc không tồn tại"

## Các Nguyên Nhân Có Thể

### 1. **Cookies không được gửi từ Frontend**
- Frontend không gửi cookies `_at` hoặc `_u` trong request
- API proxy không forward cookies đúng cách

### 2. **userId không được extract đúng**
- `extractUserIdFromRequest()` không tìm thấy userId từ cookies hoặc accessToken
- AccessToken đã hết hạn và không thể decode

### 3. **authToken không tìm thấy trong DB**
- Không có record trong `auth_tokens` table cho userId
- Record đã bị xóa hoặc chưa được tạo

### 4. **Refresh token trong DB không hợp lệ**
- Refresh token trong DB đã hết hạn
- Refresh token signature không hợp lệ

## Cách Debug

### Bước 1: Kiểm tra Logs Backend

Khi refresh token được gọi, check console logs của backend server:

```bash
# Trong terminal chạy backend
# Bạn sẽ thấy các logs:
[Refresh] Start
[Refresh] req.cookies: { _at: '...', _u: '...' }
[Refresh] req.headers.authorization: exists/missing
[ExtractUserId] _u cookie exists: true/false
[ExtractUserId] _at cookie exists: true/false
[ExtractUserId] No accessToken found  # ← Nếu thấy này, cookies không được gửi
[Auth] Invalid userId: ...  # ← Nếu thấy này, userId không hợp lệ
[Auth] No authToken found for userId: ...  # ← Nếu thấy này, không có token trong DB
```

### Bước 2: Kiểm tra Logs Frontend (API Proxy)

Check console logs của Next.js server:

```bash
# Trong terminal chạy frontend
[API Proxy Refresh] cookieHeader exists: true/false
[API Proxy Refresh] cookieHeader length: ...
[API Proxy Refresh] Cookie names: ['_at', '_u']
[API Proxy Refresh] Backend status: 200/401/500
[API Proxy Refresh] Response has access_token: true/false
```

### Bước 3: Kiểm tra Network Tab

1. Mở Chrome DevTools (F12)
2. Vào tab "Network"
3. Tìm request `/api-proxy/auth/refresh`
4. Check:
   - **Request Headers**: Có `Cookie` header không?
   - **Response**: Status code và response body là gì?

### Bước 4: Kiểm tra Cookies trong Browser

1. Mở Chrome DevTools (F12)
2. Vào tab "Application" > "Cookies"
3. Check xem có cookies `_at` và `_u` không
4. Check xem cookies có giá trị không (không phải empty)

### Bước 5: Kiểm tra Database

```sql
-- Kiểm tra xem có authToken cho userId không
SELECT * FROM auth_tokens WHERE user_id = <userId> ORDER BY created_at DESC;

-- Kiểm tra refresh token có hết hạn không
SELECT 
  token_id,
  user_id,
  refresh_token,
  refresh_expires_at,
  created_at,
  CASE 
    WHEN refresh_expires_at < NOW() THEN 'EXPIRED'
    ELSE 'VALID'
  END as status
FROM auth_tokens 
WHERE user_id = <userId>
ORDER BY created_at DESC;
```

## Các Trường Hợp Cụ Thể

### Case 1: "No accessToken found"
**Nguyên nhân:** Cookies không được gửi từ frontend

**Fix:**
- Check `withCredentials: true` trong axios request
- Check cookies có tồn tại trong browser không
- Check API proxy có forward cookies không

### Case 2: "No authToken found for userId"
**Nguyên nhân:** Không có record trong DB

**Fix:**
- User cần đăng nhập lại để tạo authToken mới
- Hoặc check xem có bug trong signin/signup không tạo authToken

### Case 3: "Invalid userId"
**Nguyên nhân:** userId không hợp lệ (null, 0, negative)

**Fix:**
- Check extractUserIdFromRequest() có extract đúng không
- Check cookies có chứa user_id không

### Case 4: Refresh token expired/invalid
**Nguyên nhân:** Token trong DB đã hết hạn hoặc signature không hợp lệ

**Fix:**
- User cần đăng nhập lại
- Check JWT_SECRET có đúng không

## Quick Fix

Nếu vấn đề là cookies không được gửi, thử:

1. **Clear cookies và đăng nhập lại**
2. **Check CORS settings** - đảm bảo `credentials: 'include'` được set
3. **Check SameSite cookie settings** - có thể cần set `SameSite=None; Secure`

## Test Manual

1. Đăng nhập để tạo cookies
2. Đợi access token hết hạn (hoặc manually expire)
3. Gọi một API bất kỳ
4. Check logs để xem refresh flow có chạy không
5. Check Network tab để xem cookies có được gửi không
