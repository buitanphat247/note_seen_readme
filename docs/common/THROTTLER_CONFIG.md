# Throttler Configuration

## Environment Variables

Thêm các biến sau vào file `.env` của bạn để cấu hình rate limiting:

```env
# Rate Limiting Configuration (Throttler)
# Global rate limit for all endpoints (default: 100 requests per minute)
THROTTLER_TTL=60000
THROTTLER_LIMIT=100

# Signup endpoint rate limit (default: 10 requests per minute)
THROTTLER_SIGNUP_TTL=60000
THROTTLER_SIGNUP_LIMIT=10

# Signin endpoint rate limit (default: 20 requests per minute)
THROTTLER_SIGNIN_TTL=60000
THROTTLER_SIGNIN_LIMIT=20
```

## Giải thích

- **THROTTLER_TTL**: Time window trong milliseconds (mặc định: 60000 = 1 phút)
- **THROTTLER_LIMIT**: Số lượng request tối đa trong time window (mặc định: 100)
- **THROTTLER_SIGNUP_TTL**: Time window cho signup endpoint (mặc định: 60000 = 1 phút)
- **THROTTLER_SIGNUP_LIMIT**: Số lượng request tối đa cho signup trong time window (mặc định: 10)
- **THROTTLER_SIGNIN_TTL**: Time window cho signin endpoint (mặc định: 60000 = 1 phút)
- **THROTTLER_SIGNIN_LIMIT**: Số lượng request tối đa cho signin trong time window (mặc định: 20)

## Ví dụ

Để tăng limit lên 50 requests/phút cho signin:

```env
THROTTLER_SIGNIN_LIMIT=50
THROTTLER_SIGNIN_TTL=60000
```

Để tăng limit lên 20 requests/phút cho signup:

```env
THROTTLER_SIGNUP_LIMIT=20
THROTTLER_SIGNUP_TTL=60000
```
