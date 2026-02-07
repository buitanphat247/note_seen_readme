# Fix: req is undefined in refresh endpoint

## Vấn Đề
Khi debug, phát hiện `req` object là `undefined` trong `refresh()` method của `AuthController`.

## Nguyên Nhân Có Thể

### 1. **Next.js API Route không forward request đúng cách**
- Next.js API route (`/api-proxy/auth/refresh`) gọi backend qua `fetch()`
- Request object không được serialize/deserialize đúng cách
- NestJS không nhận được Express request object

### 2. **Cookie Parser không hoạt động**
- Cookies được gửi trong `Cookie` header nhưng không được parse thành `req.cookies`
- Cần parse manually từ `req.headers.cookie`

### 3. **Request Injection Issue**
- `@Req()` decorator không inject request object đúng cách
- Có thể do middleware hoặc interceptor nào đó

## Giải Pháp Đã Áp Dụng

### 1. **Fallback để lấy req từ res.req**
```typescript
if (!req) {
  if (res && (res as any).req) {
    req = (res as any).req;
  } else {
    throw new InvalidRefreshTokenException();
  }
}
```

### 2. **Parse cookies từ Cookie header**
```typescript
if (!req.cookies || Object.keys(req.cookies).length === 0) {
  if (req.headers?.cookie) {
    req.cookies = {};
    const cookieHeader = req.headers.cookie;
    cookieHeader.split(';').forEach((cookie: string) => {
      const [name, ...valueParts] = cookie.trim().split('=');
      if (name && valueParts.length > 0) {
        req.cookies[name] = decodeURIComponent(valueParts.join('='));
      }
    });
  }
}
```

## Debug Steps

1. **Check logs khi refresh được gọi:**
   ```
   [Refresh] req exists: true/false
   [Refresh] req type: object/undefined
   [Refresh] req.cookies: {...}
   [Refresh] req.headers.cookie: exists/missing
   ```

2. **Nếu `req` là undefined:**
   - Check xem `res.req` có tồn tại không
   - Check logs: `[Refresh] Got req from res.req`

3. **Nếu `req.cookies` là empty:**
   - Check xem `req.headers.cookie` có giá trị không
   - Check logs: `[Refresh] Parsed cookies from header: [...]`

## Test

1. Đăng nhập để tạo cookies
2. Đợi access token hết hạn
3. Gọi một API bất kỳ
4. Check logs để xem:
   - `req` có undefined không?
   - Cookies có được parse không?
   - userId có được extract không?

## Nếu Vẫn Không Hoạt Động

Có thể cần kiểm tra:
1. **Next.js API route có forward cookies đúng không?**
   - Check `app/api-proxy/auth/refresh/route.ts`
   - Đảm bảo `Cookie` header được forward

2. **Backend có nhận được Cookie header không?**
   - Check Network tab trong DevTools
   - Check request headers trong backend logs

3. **Cookie parser có hoạt động không?**
   - Check `main.ts` - đảm bảo `cookieParser()` được setup
   - Check middleware order
