# ✅ PRODUCTION COOKIE CONFIGURATION CHECKLIST

## 🔍 Tóm tắt các thay đổi Cookie

### Backend (`auth.controller.ts`)
- **Local Development**: `SameSite=Lax`, `Secure=false` (cho localhost cross-port)
- **Production**: `SameSite=None`, `Secure=true`, `domain=.edulearning.io.vn`

### Frontend (`lib/api/auth.ts` & `app/config/api.ts`)
- **Local Development**: `SameSite=Lax` (khi `NODE_ENV === "development"`)
- **Production**: `SameSite=None; Secure=true` (khi `NODE_ENV === "production"`)

---

## ⚠️ QUAN TRỌNG: Kiểm tra trước khi deploy Production

### 1. Backend `.env` file (`Sever_Linux/Edulearn-Sever/.env`)

**✅ ĐÚNG cho Production:**
```env
NODE_ENV=production
IS_LOCAL=false
# hoặc không set IS_LOCAL (sẽ được detect từ origin)
```

**❌ SAI - KHÔNG được commit:**
```env
NODE_ENV=production
IS_LOCAL=true  # ← NÀY SẼ GÂY LỖI!
```

**Logic hiện tại:**
- Backend check: `if (isProd && !isLocalOrigin)` → dùng production config
- Nếu `IS_LOCAL=true` nhưng origin là production domain → vẫn dùng production config ✅
- **NHƯNG**: Để an toàn, nên set `IS_LOCAL=false` trong production

### 2. Frontend `.env` file (`EduLearn-Client/.env`)

**✅ ĐÚNG cho Production:**
```env
NEXT_PUBLIC_API_URL=https://api.edulearning.io.vn/api
NEXT_PUBLIC_FLASK_API_URL=https://api.edulearning.io.vn/api
```

**Logic Frontend:**
- `process.env.NODE_ENV === "development"` → `SameSite=Lax`
- `process.env.NODE_ENV === "production"` → `SameSite=None; Secure=true` ✅

### 3. Docker/Deployment Config

**✅ Đảm bảo trong `production.yml` hoặc deployment script:**
```yaml
environment:
  - NODE_ENV=production
  - IS_LOCAL=false  # ← QUAN TRỌNG!
```

---

## 🛡️ Bảo vệ Production

### Backend Logic (Đã được bảo vệ):
```typescript
const isProd = process.env.NODE_ENV === 'production';
const isLocalOrigin = origin.includes('localhost') || origin.includes('127.0.0.1');

// Production: Always use production config (ignore IS_LOCAL if set incorrectly)
if (isProd && !isLocalOrigin) {
  return {
    sameSite: 'none',
    secure: true,
    domain: '.edulearning.io.vn',
  };
}
```

**✅ An toàn vì:**
- Kiểm tra cả `NODE_ENV` VÀ `origin`
- Nếu origin là production domain → luôn dùng production config
- `IS_LOCAL` chỉ ảnh hưởng khi `NODE_ENV !== 'production'` hoặc origin là localhost

### Frontend Logic (Đã được bảo vệ):
```typescript
const isDev = process.env.NODE_ENV === "development";
const sameSiteAttr = isDev ? 'SameSite=Lax' : 'SameSite=None; Secure=true';
```

**✅ An toàn vì:**
- Next.js tự động set `NODE_ENV=production` khi build production
- Không thể override từ `.env` file

---

## 📋 CHECKLIST TRƯỚC KHI DEPLOY

- [ ] **Backend `.env`**: `IS_LOCAL=false` hoặc không set (KHÔNG được là `true`)
- [ ] **Backend `.env`**: `NODE_ENV=production`
- [ ] **Frontend build**: Đảm bảo build với `NODE_ENV=production` (Next.js tự động)
- [ ] **Frontend `.env`**: `NEXT_PUBLIC_API_URL=https://api.edulearning.io.vn/api`
- [ ] **Docker/Deploy**: Set `NODE_ENV=production` và `IS_LOCAL=false` trong environment
- [ ] **Test**: Verify cookies có `SameSite=None; Secure=true` trong production

---

## 🔒 KẾT LUẬN

**Code hiện tại ĐÃ AN TOÀN cho production** vì:
1. ✅ Backend check cả `NODE_ENV` VÀ `origin` → không thể sai
2. ✅ Frontend check `NODE_ENV` → Next.js tự động set trong production build
3. ✅ Logic fallback đúng: nếu origin là production domain → luôn dùng production config

**Lưu ý duy nhất:**
- Đảm bảo `IS_LOCAL=false` hoặc không set trong production `.env`
- Nhưng ngay cả khi set sai, logic vẫn an toàn vì check `origin`
