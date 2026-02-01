# 📋 CODE REVIEW: app/(root) - Đánh Giá Chi Tiết Hiệu Năng & Bug Tiềm Ẩn

**Ngày review:** 2026-01-21  
**Scope:** Toàn bộ code trong `app/(root)` folder  
**Mục tiêu:** Phát hiện bugs, vấn đề hiệu năng, race conditions, và các lỗi tiềm ẩn

---

## 📊 TỔNG QUAN

### Thống kê
- **Tổng số files:** 23 files (TypeScript/TSX)
- **Files có vấn đề:** 15 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 8 issues
  - 🟡 **Warning:** 12 issues
  - 🟢 **Info:** 5 issues

---

## 🔴 CRITICAL ISSUES

### 1. **STATE & DATA FLOW BUGS**

#### 1.1. Stale State - `events/page.tsx`
**File:** `app/(root)/events/page.tsx`  
**Dòng:** 37-54

**Vấn đề:**
```typescript
const fetchEvents = useCallback(async () => {
  setLoading(true);
  setEvents([]); // Clear events to prevent overlap
  const result = await getEvents({
    page: currentPage,
    limit: pageSize,
    search: debouncedSearchText || undefined,
  });
  setEvents(result.events);
  setTotal(result.total);
}, [currentPage, debouncedSearchText, message]);
```

**Bug:** Race condition khi user thay đổi page nhanh:
- Request A (page 1) gửi trước
- Request B (page 2) gửi sau
- B về trước → overwrite A → hiển thị sai data

**Fix:**
```typescript
const fetchEvents = useCallback(async () => {
  setLoading(true);
  const currentPageRef = currentPage; // Capture current page
  const result = await getEvents({
    page: currentPageRef,
    limit: pageSize,
    search: debouncedSearchText || undefined,
  });
  
  // Only update if still on the same page
  if (currentPageRef === currentPage) {
    setEvents(result.events);
    setTotal(result.total);
  }
  setLoading(false);
}, [currentPage, debouncedSearchText, message]);
```

**Hoặc dùng AbortController:**
```typescript
const abortControllerRef = useRef<AbortController | null>(null);

const fetchEvents = useCallback(async () => {
  // Cancel previous request
  if (abortControllerRef.current) {
    abortControllerRef.current.abort();
  }
  
  const controller = new AbortController();
  abortControllerRef.current = controller;
  
  try {
    setLoading(true);
    const result = await getEvents({...}, { signal: controller.signal });
    setEvents(result.events);
    setTotal(result.total);
  } catch (error) {
    if (error.name !== 'AbortError') {
      message.error(error.message);
    }
  } finally {
    setLoading(false);
  }
}, [currentPage, debouncedSearchText, message]);
```

---

#### 1.2. Out-of-sync State - `profile/page.tsx`
**File:** `app/(root)/profile/page.tsx`  
**Dòng:** 28-46

**Vấn đề:**
```typescript
const hasFetched = useRef(false);

useEffect(() => {
  if (hasFetched.current) return;
  
  const fetchUserInfo = async () => {
    hasFetched.current = true;
    const userInfo = await getProfile();
    setUser(userInfo as UserInfoResponse);
  };
  
  fetchUserInfo();
}, []);
```

**Bug:**
- `hasFetched.current` không reset khi component unmount/remount
- Nếu user logout/login lại, data cũ vẫn được giữ
- Không sync với localStorage/Context

**Fix:**
```typescript
useEffect(() => {
  let isMounted = true;
  
  const fetchUserInfo = async () => {
    try {
      const userInfo = await getProfile();
      if (isMounted) {
        setUser(userInfo as UserInfoResponse);
        // Sync với localStorage
        localStorage.setItem('user', JSON.stringify(userInfo));
      }
    } catch (error: any) {
      if (isMounted) {
        message.error(error.message || "Không thể tải thông tin user");
      }
    } finally {
      if (isMounted) {
        setLoading(false);
      }
    }
  };
  
  fetchUserInfo();
  
  return () => {
    isMounted = false;
  };
}, []); // Remove hasFetched ref
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Double Submit - `events/page.tsx`
**File:** `app/(root)/events/page.tsx`  
**Dòng:** 145-148

**Vấn đề:**
```typescript
const handlePageChange = (page: number) => {
  setCurrentPage(page);
  window.scrollTo({ top: 0, behavior: "smooth" });
};
```

**Bug:** User click nhanh pagination → gửi nhiều request cùng lúc

**Fix:**
```typescript
const [isChangingPage, setIsChangingPage] = useState(false);

const handlePageChange = (page: number) => {
  if (isChangingPage) return; // Prevent double click
  
  setIsChangingPage(true);
  setCurrentPage(page);
  window.scrollTo({ top: 0, behavior: "smooth" });
  
  setTimeout(() => setIsChangingPage(false), 500);
};
```

---

#### 2.2. Request về sai thứ tự - `news/page.tsx`
**File:** `app/(root)/news/page.tsx`  
**Dòng:** 129-137

**Vấn đề:**
```typescript
onChange={(page) => {
  setIsScrolling(true);
  setCurrentPage(page);
  window.scrollTo({ top: 0, behavior: 'smooth' });
  setTimeout(() => {
    setIsScrolling(false);
  }, 500);
}}
```

**Bug:** Không có debounce/throttle → nhiều state updates liên tiếp

**Fix:**
```typescript
const handlePageChange = useCallback((page: number) => {
  setIsScrolling(true);
  setCurrentPage(page);
  window.scrollTo({ top: 0, behavior: 'smooth' });
  
  const timer = setTimeout(() => {
    setIsScrolling(false);
  }, 500);
  
  return () => clearTimeout(timer);
}, []);
```

---

### 3. **RENDER / REACT LIFECYCLE BUGS**

#### 3.1. Infinite Re-render Risk - `news/[id]/page.tsx`
**File:** `app/(root)/news/[id]/page.tsx`  
**Dòng:** 51-56

**Vấn đề:**
```typescript
const featuredNewsList = allNews.filter((item) =>
  featuredNews.includes(item.id)
);
const relatedNewsList = allNews.filter((item) =>
  news.relatedNews.includes(item.id)
);
```

**Bug:** Tính toán lại mỗi render, không memoize

**Fix:**
```typescript
const featuredNewsList = useMemo(() => 
  allNews.filter((item) => featuredNews.includes(item.id)),
  [] // allNews và featuredNews là constants
);

const relatedNewsList = useMemo(() => 
  allNews.filter((item) => news?.relatedNews?.includes(item.id) || false),
  [news?.relatedNews]
);
```

---

#### 3.2. Missing Dependency - `events/page.tsx`
**File:** `app/(root)/events/page.tsx`  
**Dòng:** 56-58

**Vấn đề:**
```typescript
useEffect(() => {
  fetchEvents();
}, [fetchEvents]);
```

**Bug:** `fetchEvents` thay đổi mỗi render → infinite loop tiềm ẩn

**Fix:** Đã đúng với `useCallback`, nhưng cần kiểm tra dependencies:
```typescript
const fetchEvents = useCallback(async () => {
  // ... code
}, [currentPage, debouncedSearchText, message]); // ✅ Đã đúng
```

---

#### 3.3. Unnecessary Re-renders - `about/page.tsx`
**File:** `app/(root)/about/page.tsx`  
**Dòng:** 130-138

**Vấn đề:**
```typescript
{stats.map((stat, index) => (
  <div key={index} className="text-center group px-4">
    {/* ... */}
  </div>
))}
```

**Bug:** Dùng `index` làm key → re-render không cần thiết khi array thay đổi

**Fix:**
```typescript
{stats.map((stat) => (
  <div key={stat.title} className="text-center group px-4">
    {/* ... */}
  </div>
))}
```

---

### 4. **HYDRATION / SSR BUGS**

#### 4.1. Client-only Code in SSR - `news/[id]/page.tsx`
**File:** `app/(root)/news/[id]/page.tsx`  
**Dòng:** 18-29

**Vấn đề:**
```typescript
const [mounted, setMounted] = useState(false);

useEffect(() => {
  setMounted(true);
}, []);

if (!mounted) {
  return <NewsDetailSkeleton />;
}
```

**Bug:** 
- Server render skeleton
- Client render content → hydration mismatch
- FOUC (Flash of Unstyled Content)

**Fix:**
```typescript
// Option 1: Make it fully client-side
"use client";

// Option 2: Use dynamic import with ssr: false
import dynamic from 'next/dynamic';
const NewsDetail = dynamic(() => import('./NewsDetailClient'), { 
  ssr: false,
  loading: () => <NewsDetailSkeleton />
});
```

---

#### 4.2. Date Formatting Mismatch - `profile/page.tsx`
**File:** `app/(root)/profile/page.tsx`  
**Dòng:** 250-255

**Vấn đề:**
```typescript
{new Date(user.created_at).toLocaleDateString("vi-VN", {
  year: "numeric",
  month: "long",
  day: "numeric",
})}
```

**Bug:** Server và client có thể format khác nhau (timezone, locale)

**Fix:**
```typescript
// Use a library like date-fns or format on server
import { format } from 'date-fns';
import { vi } from 'date-fns/locale';

{format(new Date(user.created_at), 'dd MMMM yyyy', { locale: vi })}
```

---

### 5. **MEMORY LEAK**

#### 5.1. Interval không cleanup - `features/writing/[id]/page.tsx`
**File:** `app/(root)/features/writing/[id]/page.tsx`  
**Dòng:** 98-103

**Vấn đề:**
```typescript
useEffect(() => {
  if (!loading) {
    intervalRef.current = setInterval(() => {
      setTimeElapsed((prev) => prev + 1);
    }, 1000);
  }
}, [loading]);
```

**Bug:** Không cleanup interval khi component unmount

**Fix:**
```typescript
useEffect(() => {
  if (!loading) {
    intervalRef.current = setInterval(() => {
      setTimeElapsed((prev) => prev + 1);
    }, 1000);
  }
  
  return () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
    }
  };
}, [loading]);
```

---

#### 5.2. Event Listener không cleanup - `news/page.tsx`
**File:** `app/(root)/news/page.tsx`  
**Dòng:** 16 (state `isScrolling`)

**Vấn đề:** Không có event listener nhưng state `isScrolling` có thể leak nếu component unmount giữa timeout

**Fix:**
```typescript
useEffect(() => {
  return () => {
    setIsScrolling(false); // Cleanup on unmount
  };
}, []);
```

---

### 6. **UI / CSS BUGS**

#### 6.1. Layout Shift - `about/page.tsx`
**File:** `app/(root)/about/page.tsx`  
**Dòng:** 114-119

**Vấn đề:**
```typescript
<img
  src="https://lh3.googleusercontent.com/..."
  alt="Modern digital classroom"
  className="w-full h-full object-cover"
/>
```

**Bug:** Image load sau → CLS (Cumulative Layout Shift) cao

**Fix:**
```typescript
<Image
  src="https://lh3.googleusercontent.com/..."
  alt="Modern digital classroom"
  width={800}
  height={600}
  className="w-full h-full object-cover"
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
  priority // If above fold
/>
```

---

#### 6.2. Z-index Hell - `news/[id]/page.tsx`
**File:** `app/(root)/news/[id]/page.tsx`  
**Dòng:** 149

**Vấn đề:**
```typescript
<span className="relative z-20 flex items-center gap-2">
```

**Bug:** Z-index cao (z-20) có thể conflict với modal/dropdown khác

**Fix:** Sử dụng z-index hợp lý:
- Modal: z-50
- Dropdown: z-40
- Sticky header: z-30
- Button: z-10

---

### 7. **SECURITY BUGS**

#### 7.1. XSS Risk - `guide/page.tsx` & `innovation/page.tsx`
**File:** `app/(root)/guide/page.tsx`, `app/(root)/innovation/page.tsx`  
**Dòng:** 77

**Vấn đề:**
```typescript
<GuideContent content={content} />
```

**Nếu `GuideContent` dùng `dangerouslySetInnerHTML`:** XSS risk

**Fix:**
```typescript
// Use a safe markdown parser
import { marked } from 'marked';
import DOMPurify from 'dompurify';

const safeContent = DOMPurify.sanitize(marked.parse(content));
```

---

#### 7.2. Token trong localStorage - `profile/page.tsx`
**File:** `app/(root)/profile/page.tsx`  
**Dòng:** (implicit - localStorage usage)

**Vấn đề:** Nếu lưu token trong localStorage → vulnerable to XSS

**Fix:** 
- Dùng httpOnly cookies (server-side)
- Hoặc encrypt token trước khi lưu localStorage

---

### 8. **ERROR HANDLING BUGS**

#### 8.1. Promise reject không catch - `vocabulary/[folderId]/page.tsx`
**File:** `app/(root)/features/vocabulary/[folderId]/page.tsx`  
**Dòng:** 29-49

**Vấn đề:**
```typescript
const fetchVocabularies = async () => {
  try {
    const data = await getVocabulariesByFolder(folderId);
    setVocabularies(data);
  } catch (error: any) {
    console.error("Error fetching vocabularies:", error);
    message.error(error?.message || "Không thể tải danh sách từ vựng");
  }
};
```

**Bug:** Nếu `getVocabulariesByFolder` throw error không phải Error object → crash

**Fix:**
```typescript
const fetchVocabularies = async () => {
  try {
    const data = await getVocabulariesByFolder(folderId);
    setVocabularies(data);
  } catch (error: unknown) {
    const errorMessage = error instanceof Error 
      ? error.message 
      : 'Không thể tải danh sách từ vựng';
    console.error("Error fetching vocabularies:", error);
    message.error(errorMessage);
    setVocabularies([]);
  } finally {
    setLoading(false);
  }
};
```

---

#### 8.2. Không có ErrorBoundary - Tất cả pages

**Vấn đề:** Nếu 1 component crash → toàn bộ app crash

**Fix:**
```typescript
// app/error-boundary.tsx
'use client';

export default function ErrorBoundary({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h2>Something went wrong!</h2>
        <button onClick={reset}>Try again</button>
      </div>
    </div>
  );
}
```

---

### 9. **PERFORMANCE BUGS**

#### 9.1. Re-render quá nhiều - `news/page.tsx`
**File:** `app/(root)/news/page.tsx`  
**Dòng:** 106-121

**Vấn đề:**
```typescript
{currentNews.map((item, index) => (
  <ScrollAnimation
    key={`${item.id}-${currentPage}`}
    direction="up"
    delay={isScrolling ? 500 + (index * 50) : index * 50}
  >
    <NewsCard {...item} />
  </ScrollAnimation>
))}
```

**Bug:** 
- `ScrollAnimation` re-render mỗi khi `isScrolling` thay đổi
- Key thay đổi → unmount/remount components

**Fix:**
```typescript
const newsCards = useMemo(() => 
  currentNews.map((item, index) => (
    <ScrollAnimation
      key={item.id} // Stable key
      direction="up"
      delay={index * 50}
    >
      <NewsCard {...item} />
    </ScrollAnimation>
  )),
  [currentNews] // Only recalculate when news changes
);
```

---

#### 9.2. Bundle size - `about/page.tsx`
**File:** `app/(root)/about/page.tsx`  
**Dòng:** 4-19

**Vấn đề:**
```typescript
import { 
  RocketOutlined, 
  GlobalOutlined, 
  HeartOutlined, 
  // ... 15 icons
} from "@ant-design/icons";
```

**Bug:** Import tất cả icons → bundle size lớn

**Fix:**
```typescript
// Use dynamic import or tree-shaking
import RocketOutlined from "@ant-design/icons/RocketOutlined";
// Or use a icon registry
```

---

#### 9.3. Large Component - `writing/[id]/page.tsx`
**File:** `app/(root)/features/writing/[id]/page.tsx`  
**Dòng:** 75-592

**Vấn đề:** Component quá lớn (592 dòng) → khó maintain, re-render toàn bộ

**Fix:** Split thành smaller components:
- `WritingPracticeHeader`
- `WritingPracticeContent`
- `WritingPracticeControls`
- `WritingPracticeTimer`

---

## 🟡 WARNING ISSUES

### 10. **Missing Optimizations**

#### 10.1. `useMemo` cho filtered data - `news/page.tsx`
**File:** `app/(root)/news/page.tsx`  
**Dòng:** 20-27

**Status:** ✅ Đã có `useMemo` - Good!

---

#### 10.2. `useCallback` cho handlers - `events/page.tsx`
**File:** `app/(root)/events/page.tsx`  
**Dòng:** 37-54

**Status:** ✅ Đã có `useCallback` - Good!

---

#### 10.3. Debounce search - `events/page.tsx`
**File:** `app/(root)/events/page.tsx`  
**Dòng:** 27-34

**Status:** ✅ Đã có debounce - Good!

---

### 11. **Code Quality Issues**

#### 11.1. Type Safety - `guide/page.tsx`
**File:** `app/(root)/guide/page.tsx`  
**Dòng:** 39-44

**Vấn đề:**
```typescript
export default async function GuidePage(props: any) {
  const searchParams = await props.searchParams;
```

**Fix:**
```typescript
interface GuidePageProps {
  searchParams: Promise<{ doc?: string }>;
}

export default async function GuidePage(props: GuidePageProps) {
  const searchParams = await props.searchParams;
```

---

#### 11.2. Magic Numbers - `news/page.tsx`
**File:** `app/(root)/news/page.tsx`  
**Dòng:** 18, 134

**Vấn đề:**
```typescript
const pageSize = 18;
setTimeout(() => {
  setIsScrolling(false);
}, 500); // Magic number
```

**Fix:**
```typescript
const SCROLL_DELAY_MS = 500;
const DEFAULT_PAGE_SIZE = 18;
```

---

## 🟢 INFO / SUGGESTIONS

### 12. **Best Practices**

#### 12.1. Constants extraction - `about/page.tsx`
**Suggestion:** Extract hardcoded arrays to constants file

---

#### 12.2. Loading states - Tất cả pages
**Suggestion:** Consistent loading skeleton pattern

---

#### 12.3. Error messages - Tất cả pages
**Suggestion:** Centralized error message constants

---

## 📈 PERFORMANCE METRICS

### Re-render Analysis

| Component | Re-renders | Issues |
|-----------|-----------|--------|
| `events/page.tsx` | High | Race condition, double submit |
| `news/page.tsx` | Medium | Unnecessary re-renders on scroll |
| `profile/page.tsx` | Low | ✅ Good |
| `about/page.tsx` | Low | ✅ Good |
| `writing/[id]/page.tsx` | High | Large component, memory leak |

### Bundle Size Impact

| File | Size Impact | Issue |
|------|-------------|-------|
| `about/page.tsx` | High | Import nhiều icons |
| `writing/[id]/page.tsx` | High | Component quá lớn |
| `news/[id]/page.tsx` | Medium | Image không optimize |

---

## ✅ PRIORITY FIX LIST

### 🔴 Critical (Fix ngay)
1. **Race condition** trong `events/page.tsx` - AbortController
2. **Memory leak** trong `writing/[id]/page.tsx` - Cleanup interval
3. **Double submit** trong `events/page.tsx` - Disable button
4. **Hydration mismatch** trong `news/[id]/page.tsx` - Fix SSR

### 🟡 High (Fix sớm)
5. **Stale state** trong `profile/page.tsx` - Sync với localStorage
6. **Re-render** trong `news/page.tsx` - Memoize components
7. **Error handling** trong `vocabulary/[folderId]/page.tsx` - Better catch

### 🟢 Medium (Cải thiện)
8. **Layout shift** trong `about/page.tsx` - Image optimization
9. **Bundle size** trong `about/page.tsx` - Tree-shake icons
10. **Type safety** trong `guide/page.tsx` - Remove `any`

---

## 📝 SUMMARY

### Tổng kết
- **Critical bugs:** 8 issues cần fix ngay
- **Performance issues:** 5 issues ảnh hưởng UX
- **Code quality:** 7 issues cần cải thiện

### Điểm mạnh
- ✅ Sử dụng `useMemo`, `useCallback` đúng chỗ
- ✅ Có debounce cho search
- ✅ Có loading states
- ✅ Code structure tương đối tốt

### Điểm yếu
- ❌ Race conditions không được handle
- ❌ Memory leaks trong timers
- ❌ Hydration mismatches
- ❌ Thiếu ErrorBoundary
- ❌ Bundle size chưa optimize

---

## 🔧 RECOMMENDED ACTIONS

1. **Immediate:**
   - Fix race conditions với AbortController
   - Cleanup intervals/timeouts
   - Fix hydration issues

2. **Short-term:**
   - Add ErrorBoundary
   - Optimize re-renders
   - Improve error handling

3. **Long-term:**
   - Split large components
   - Optimize bundle size
   - Add comprehensive tests

---

## 📁 PHÂN TÍCH: app/actions

### Tổng quan
- **Tổng số files:** 1 file
- **Files có vấn đề:** 1 file
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 3 issues
  - 🟡 **Warning:** 2 issues
  - 🟢 **Info:** 1 issue

---

## 🔴 CRITICAL ISSUES - app/actions

### 1. **SECURITY BUGS**

#### 1.1. Input Validation Missing - `theme.ts`
**File:** `app/actions/theme.ts`  
**Dòng:** 5-14

**Vấn đề:**
```typescript
export async function setThemeCookie(theme: string) {
  const cookieStore = await cookies();
  cookieStore.set("theme", theme, {
    httpOnly: false,
    secure: process.env.NODE_ENV === "production",
    maxAge: 60 * 60 * 24 * 365, // 1 year
    path: "/",
    sameSite: "lax",
  });
}
```

**Bug:** 
- ❌ Không validate input `theme` → có thể inject malicious values
- ❌ `httpOnly: false` → vulnerable to XSS attacks
- ❌ Không sanitize input → có thể chứa special characters

**Fix:**
```typescript
"use server";

import { cookies } from "next/headers";

const VALID_THEMES = ["light", "dark"] as const;
type Theme = typeof VALID_THEMES[number];

export async function setThemeCookie(theme: string) {
  // Validate input
  if (!theme || typeof theme !== "string") {
    throw new Error("Theme must be a string");
  }

  // Sanitize and validate theme value
  const sanitizedTheme = theme.trim().toLowerCase();
  if (!VALID_THEMES.includes(sanitizedTheme as Theme)) {
    throw new Error(`Invalid theme. Must be one of: ${VALID_THEMES.join(", ")}`);
  }

  const cookieStore = await cookies();
  cookieStore.set("theme", sanitizedTheme, {
    httpOnly: false, // Keep false for client-side access, but consider security implications
    secure: process.env.NODE_ENV === "production",
    maxAge: 60 * 60 * 24 * 365, // 1 year
    path: "/",
    sameSite: "lax",
  });

  return { success: true, theme: sanitizedTheme };
}
```

---

#### 1.2. No Error Handling - `theme.ts`
**File:** `app/actions/theme.ts`  
**Dòng:** 5-14

**Vấn đề:**
```typescript
export async function setThemeCookie(theme: string) {
  const cookieStore = await cookies();
  cookieStore.set("theme", theme, {
    // ...
  });
}
```

**Bug:**
- ❌ Không có try-catch → nếu `cookies()` fail → unhandled error
- ❌ Không return success/error status
- ❌ Client không biết action có thành công hay không

**Fix:**
```typescript
"use server";

import { cookies } from "next/headers";

const VALID_THEMES = ["light", "dark"] as const;

export async function setThemeCookie(theme: string): Promise<{ success: boolean; error?: string }> {
  try {
    // Validate input
    if (!theme || typeof theme !== "string") {
      return { success: false, error: "Theme must be a string" };
    }

    const sanitizedTheme = theme.trim().toLowerCase();
    if (!VALID_THEMES.includes(sanitizedTheme as any)) {
      return { success: false, error: `Invalid theme. Must be one of: ${VALID_THEMES.join(", ")}` };
    }

    const cookieStore = await cookies();
    cookieStore.set("theme", sanitizedTheme, {
      httpOnly: false,
      secure: process.env.NODE_ENV === "production",
      maxAge: 60 * 60 * 24 * 365,
      path: "/",
      sameSite: "lax",
    });

    return { success: true };
  } catch (error) {
    console.error("Error setting theme cookie:", error);
    return { 
      success: false, 
      error: error instanceof Error ? error.message : "Unknown error occurred" 
    };
  }
}
```

---

#### 1.3. No Rate Limiting - `theme.ts`
**File:** `app/actions/theme.ts`  
**Dòng:** 5-14

**Vấn đề:**
- ❌ Không có rate limiting → có thể bị spam requests
- ❌ User có thể gọi action liên tục → server overload

**Fix:**
```typescript
"use server";

import { cookies } from "next/headers";
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

// Initialize rate limiter (example using Upstash)
const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "10 s"), // 10 requests per 10 seconds
});

export async function setThemeCookie(theme: string) {
  // Get client IP for rate limiting
  const ip = headers().get("x-forwarded-for") || headers().get("x-real-ip") || "unknown";
  
  // Check rate limit
  const { success, limit, remaining } = await ratelimit.limit(`theme:${ip}`);
  
  if (!success) {
    throw new Error(`Rate limit exceeded. Try again in ${limit} seconds.`);
  }

  // ... rest of the code
}
```

**Hoặc đơn giản hơn với in-memory rate limiting:**
```typescript
const rateLimitMap = new Map<string, { count: number; resetTime: number }>();

function checkRateLimit(identifier: string, maxRequests = 10, windowMs = 10000): boolean {
  const now = Date.now();
  const record = rateLimitMap.get(identifier);

  if (!record || now > record.resetTime) {
    rateLimitMap.set(identifier, { count: 1, resetTime: now + windowMs });
    return true;
  }

  if (record.count >= maxRequests) {
    return false;
  }

  record.count++;
  return true;
}
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Race Condition in ThemeContext - `ThemeContext.tsx`
**File:** `app/context/ThemeContext.tsx`  
**Dòng:** 49-120

**Vấn đề:**
```typescript
const toggleTheme = async (e?: React.MouseEvent) => {
  const newTheme = theme === "light" ? "dark" : "light";
  
  // ...
  await setThemeCookie(newTheme);
  // ...
};
```

**Bug:**
- ❌ User click nhanh → nhiều requests cùng lúc
- ❌ Race condition: Request A (light) và Request B (dark) có thể về sai thứ tự
- ❌ Không có debounce/throttle

**Fix:**
```typescript
const [isToggling, setIsToggling] = useState(false);

const toggleTheme = async (e?: React.MouseEvent) => {
  // Prevent double-click
  if (isToggling) return;
  
  setIsToggling(true);
  
  try {
    const newTheme = theme === "light" ? "dark" : "light";
    
    // ... rest of the code
    
    const result = await setThemeCookie(newTheme);
    if (!result.success) {
      console.error("Failed to set theme cookie:", result.error);
      // Fallback to client-side cookie
      document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
    }
  } catch (error) {
    console.error("Error toggling theme:", error);
    // Fallback handling
  } finally {
    setIsToggling(false);
  }
};
```

---

#### 2.2. Unhandled Promise Rejection - `ThemeContext.tsx`
**File:** `app/context/ThemeContext.tsx`  
**Dòng:** 85-87

**Vấn đề:**
```typescript
setThemeCookie(newTheme).catch(() => {
  document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
});
```

**Bug:**
- ❌ Silent failure → không log error
- ❌ Không notify user nếu server action fail
- ❌ Fallback có thể không hoạt động trong một số trường hợp

**Fix:**
```typescript
try {
  const result = await setThemeCookie(newTheme);
  if (!result.success) {
    console.warn("Server action failed, using client-side fallback:", result.error);
    // Fallback to client-side cookie
    document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
  }
} catch (error) {
  console.error("Error setting theme cookie:", error);
  // Fallback to client-side cookie
  document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
}
```

---

## 🟡 WARNING ISSUES - app/actions

### 3. **CODE QUALITY**

#### 3.1. Missing Type Safety - `theme.ts`
**File:** `app/actions/theme.ts`  
**Dòng:** 5

**Vấn đề:**
```typescript
export async function setThemeCookie(theme: string) {
```

**Fix:**
```typescript
type Theme = "light" | "dark";

export async function setThemeCookie(theme: Theme): Promise<{ success: boolean; error?: string }> {
  // ...
}
```

---

#### 3.2. Magic Numbers - `theme.ts`
**File:** `app/actions/theme.ts`  
**Dòng:** 10

**Vấn đề:**
```typescript
maxAge: 60 * 60 * 24 * 365, // 1 year
```

**Fix:**
```typescript
const COOKIE_MAX_AGE_ONE_YEAR = 60 * 60 * 24 * 365; // 1 year in seconds

cookieStore.set("theme", sanitizedTheme, {
  // ...
  maxAge: COOKIE_MAX_AGE_ONE_YEAR,
  // ...
});
```

---

## 🟢 INFO / SUGGESTIONS - app/actions

### 4. **BEST PRACTICES**

#### 4.1. Logging - `theme.ts`
**Suggestion:** Thêm logging cho debugging và monitoring

```typescript
export async function setThemeCookie(theme: string) {
  try {
    // ... validation and setting
    
    // Log successful theme change (in production, use proper logging service)
    if (process.env.NODE_ENV === "production") {
      // Send to logging service (e.g., Sentry, LogRocket)
    } else {
      console.log(`Theme changed to: ${sanitizedTheme}`);
    }
    
    return { success: true };
  } catch (error) {
    // Log error
    console.error("Error setting theme cookie:", error);
    return { success: false, error: "Failed to set theme" };
  }
}
```

---

#### 4.2. Testing - `theme.ts`
**Suggestion:** Thêm unit tests cho server actions

```typescript
// __tests__/actions/theme.test.ts
import { setThemeCookie } from "@/app/actions/theme";
import { cookies } from "next/headers";

jest.mock("next/headers");

describe("setThemeCookie", () => {
  it("should set theme cookie with valid theme", async () => {
    const mockCookieStore = {
      set: jest.fn(),
    };
    (cookies as jest.Mock).mockResolvedValue(mockCookieStore);

    const result = await setThemeCookie("dark");
    
    expect(result.success).toBe(true);
    expect(mockCookieStore.set).toHaveBeenCalledWith(
      "theme",
      "dark",
      expect.objectContaining({
        maxAge: 31536000,
      })
    );
  });

  it("should reject invalid theme", async () => {
    const result = await setThemeCookie("invalid");
    
    expect(result.success).toBe(false);
    expect(result.error).toContain("Invalid theme");
  });
});
```

---

## 📈 PERFORMANCE METRICS - app/actions

### Server Action Analysis

| Action | Calls | Issues |
|--------|-------|--------|
| `setThemeCookie` | High frequency | No rate limiting, no validation |

### Security Analysis

| Issue | Severity | Impact |
|-------|----------|--------|
| Input validation | 🔴 Critical | XSS risk |
| Error handling | 🔴 Critical | Unhandled errors |
| Rate limiting | 🔴 Critical | DoS risk |
| httpOnly flag | 🟡 Warning | XSS vulnerability |

---

## ✅ PRIORITY FIX LIST - app/actions

### 🔴 Critical (Fix ngay)
1. **Input validation** trong `theme.ts` - Validate và sanitize input
2. **Error handling** trong `theme.ts` - Add try-catch và return status
3. **Rate limiting** trong `theme.ts` - Prevent spam requests

### 🟡 High (Fix sớm)
4. **Race condition** trong `ThemeContext.tsx` - Add debounce/throttle
5. **Type safety** trong `theme.ts` - Use proper types

### 🟢 Medium (Cải thiện)
6. **Logging** trong `theme.ts` - Add proper logging
7. **Testing** - Add unit tests

---

## 📝 SUMMARY - app/actions

### Tổng kết
- **Critical bugs:** 3 issues cần fix ngay
- **Security issues:** 3 issues ảnh hưởng bảo mật
- **Code quality:** 2 issues cần cải thiện

### Điểm mạnh
- ✅ Sử dụng Next.js Server Actions đúng cách
- ✅ Có fallback mechanism trong client
- ✅ Code structure đơn giản và dễ hiểu

### Điểm yếu
- ❌ Không có input validation
- ❌ Không có error handling
- ❌ Không có rate limiting
- ❌ Race conditions trong client-side usage
- ❌ Thiếu type safety

---

## 🔧 RECOMMENDED ACTIONS - app/actions

1. **Immediate:**
   - Add input validation và sanitization
   - Add proper error handling với try-catch
   - Add rate limiting để prevent abuse

2. **Short-term:**
   - Fix race conditions trong ThemeContext
   - Improve type safety
   - Add logging

3. **Long-term:**
   - Add comprehensive tests
   - Consider using a proper rate limiting service
   - Add monitoring và alerting

---

## 📁 PHÂN TÍCH: app/admin

### Tổng quan
- **Tổng số files:** 24 files
- **Files có vấn đề:** 8 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 5 issues
  - 🟡 **Warning:** 6 issues
  - 🟢 **Info:** 3 issues

---

## 🔴 CRITICAL ISSUES - app/admin

### 1. **STATE & DATA FLOW BUGS**

#### 1.1. Race Condition Risk - `admin/page.tsx`
**File:** `app/admin/page.tsx`  
**Dòng:** 165-179

**Vấn đề:**
```typescript
useEffect(() => {
  const fetchStats = async () => {
    try {
      setLoading(true);
      const data = await getStats();
      setStats(data);
    } catch (error: any) {
      message.error(error?.message || "Không thể tải thống kê");
    } finally {
      setLoading(false);
    }
  };

  fetchStats();
}, [message]);
```

**Bug:**
- ❌ Component có thể unmount trước khi request hoàn thành → setState trên unmounted component
- ❌ Không có AbortController để cancel request
- ❌ Dependency `message` có thể thay đổi → re-fetch không cần thiết

**Fix:**
```typescript
useEffect(() => {
  let isMounted = true;
  const abortController = new AbortController();

  const fetchStats = async () => {
    try {
      setLoading(true);
      const data = await getStats();
      if (isMounted) {
        setStats(data);
      }
    } catch (error: any) {
      if (isMounted) {
        message.error(error?.message || "Không thể tải thống kê");
      }
    } finally {
      if (isMounted) {
        setLoading(false);
      }
    }
  };

  fetchStats();

  return () => {
    isMounted = false;
    abortController.abort();
  };
}, []); // Remove message from dependencies
```

---

#### 1.2. Unnecessary Re-renders - `admin/page.tsx`
**File:** `app/admin/page.tsx`  
**Dòng:** 79-103

**Vấn đề:**
```typescript
{stats.map((stat, index) => {
  return (
    <Card key={index} ...>
```

**Bug:**
- ❌ Dùng `index` làm key → re-render không cần thiết khi array thay đổi
- ❌ Tính toán `numericValue` mỗi render

**Fix:**
```typescript
const statsCards = useMemo(() => 
  stats.map((stat) => ({
    ...stat,
    numericValue: parseInt(stat.value.replace(/,/g, "")) || 0,
  })),
  [stats]
);

// In JSX:
{statsCards.map((stat) => (
  <Card key={stat.label} ...>
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Missing Error Handling - `AdminLayoutClient.tsx`
**File:** `app/admin/AdminLayoutClient.tsx`  
**Dòng:** 44-62

**Vấn đề:**
```typescript
const fetchUserInfo = useCallback(async (showError = false) => {
  if (!userId) {
    if (showError) message.error("Không tìm thấy thông tin người dùng");
    return;
  }

  if (showError) setLoadingProfile(true);
  try {
    const user = await getUserInfo(userId);
    setUserInfo(user);
  } catch (error: any) {
    if (showError) {
      message.error(error?.message || "Không thể tải thông tin người dùng");
    }
    console.error("Error fetching user info:", error);
  } finally {
    if (showError) setLoadingProfile(false);
  }
}, [userId, message]);
```

**Bug:**
- ❌ Component có thể unmount trước khi request hoàn thành
- ❌ Không có cleanup
- ❌ Dependency `message` → re-create function mỗi render

**Fix:**
```typescript
const fetchUserInfo = useCallback(async (showError = false) => {
  if (!userId) {
    if (showError) message.error("Không tìm thấy thông tin người dùng");
    return;
  }

  let isMounted = true;
  if (showError) setLoadingProfile(true);
  
  try {
    const user = await getUserInfo(userId);
    if (isMounted) {
      setUserInfo(user);
    }
  } catch (error: unknown) {
    if (isMounted) {
      const errorMessage = error instanceof Error 
        ? error.message 
        : "Không thể tải thông tin người dùng";
      if (showError) {
        message.error(errorMessage);
      }
      console.error("Error fetching user info:", error);
    }
  } finally {
    if (isMounted && showError) {
      setLoadingProfile(false);
    }
  }
  
  return () => {
    isMounted = false;
  };
}, [userId]); // Remove message dependency
```

---

#### 2.2. Date Formatting Mismatch - `AdminLayoutClient.tsx`
**File:** `app/admin/AdminLayoutClient.tsx`  
**Dòng:** 155

**Vấn đề:**
```typescript
{userInfo.created_at ? new Date(userInfo.created_at).toLocaleDateString("vi-VN") : "Chưa có thông tin"}
```

**Bug:** Server và client có thể format khác nhau (timezone, locale)

**Fix:**
```typescript
// Use a consistent date formatting library
import { format } from 'date-fns';
import { vi } from 'date-fns/locale';

{userInfo.created_at 
  ? format(new Date(userInfo.created_at), 'dd/MM/yyyy', { locale: vi })
  : "Chưa có thông tin"}
```

---

### 3. **SECURITY BUGS**

#### 3.1. Cookie Decryption Error Handling - `admin/layout.tsx`
**File:** `app/admin/layout.tsx`  
**Dòng:** 5-30

**Vấn đề:**
```typescript
async function getInitialUserData() {
  try {
    const cookieStore = await cookies();
    const userCookie = cookieStore.get("_u");

    if (userCookie?.value) {
      try {
        const decryptedUser = decryptCookie(userCookie.value);
        const userData = JSON.parse(decryptedUser);
        return {
          username: userData.username || null,
          role_name: userData.role_name || userData.role?.role_name || null,
          avatar: userData.avatar || null,
        };
      } catch (error) {
        console.error("Error decrypting/parsing user cookie:", error);
      }
    }
  } catch (error) {
    console.error("Error reading server cookie:", error);
  }

  return null;
}
```

**Bug:**
- ❌ Silent failure → không log đầy đủ
- ❌ Không validate decrypted data
- ❌ Có thể return partial data nếu JSON.parse fail một phần

**Fix:**
```typescript
async function getInitialUserData() {
  try {
    const cookieStore = await cookies();
    const userCookie = cookieStore.get("_u");

    if (!userCookie?.value) {
      return null;
    }

    try {
      const decryptedUser = decryptCookie(userCookie.value);
      const userData = JSON.parse(decryptedUser);
      
      // Validate required fields
      if (!userData || typeof userData !== 'object') {
        console.error("Invalid user data structure");
        return null;
      }

      return {
        username: userData.username || null,
        role_name: userData.role_name || userData.role?.role_name || null,
        avatar: userData.avatar || null,
      };
    } catch (decryptError) {
      console.error("Error decrypting/parsing user cookie:", decryptError);
      // In production, consider logging to monitoring service
      return null;
    }
  } catch (error) {
    console.error("Error reading server cookie:", error);
    return null;
  }
}
```

---

## 🟡 WARNING ISSUES - app/admin

### 4. **CODE QUALITY**

#### 4.1. Magic Numbers - `admin/page.tsx`
**File:** `app/admin/page.tsx`  
**Dòng:** 60-66

**Vấn đề:**
```typescript
const getGreeting = () => {
  const hour = new Date().getHours();
  if (hour < 12) return "Chào buổi sáng";
  if (hour < 18) return "Chào buổi chiều";
  return "Chào buổi tối";
};
```

**Fix:**
```typescript
const MORNING_HOUR = 12;
const EVENING_HOUR = 18;

const getGreeting = () => {
  const hour = new Date().getHours();
  if (hour < MORNING_HOUR) return "Chào buổi sáng";
  if (hour < EVENING_HOUR) return "Chào buổi chiều";
  return "Chào buổi tối";
};
```

---

#### 4.2. Type Safety - `admin/page.tsx`
**File:** `app/admin/page.tsx`  
**Dòng:** 76, 108

**Vấn đề:**
```typescript
function StatisticsCards({ stats }: { stats: any[] }) {
function QuickActionsGrid({ items }: { items: any[] }) {
```

**Fix:**
```typescript
interface StatCard {
  label: string;
  value: string;
  icon: React.ComponentType;
  color: string;
  bgColor: string;
}

function StatisticsCards({ stats }: { stats: StatCard[] }) {
  // ...
}
```

---

## 📁 PHÂN TÍCH: app/api-proxy

### Tổng quan
- **Tổng số files:** 15 files
- **Files có vấn đề:** 12 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 8 issues
  - 🟡 **Warning:** 7 issues
  - 🟢 **Info:** 2 issues

---

## 🔴 CRITICAL ISSUES - app/api-proxy

### 1. **SECURITY BUGS**

#### 1.1. No Input Validation - `[...path]/route.ts`
**File:** `app/api-proxy/[...path]/route.ts`  
**Dòng:** 7-73

**Vấn đề:**
```typescript
async function handleRequest(request: NextRequest, method: string) {
  const backendUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:1611/api';
  const url = new URL(request.url);
  const path = url.pathname.replace('/api-proxy', '');
  const targetUrl = `${backendUrl}${path}${url.search}`;
  
  // ... forward request
}
```

**Bug:**
- ❌ Không validate `path` → có thể bị SSRF (Server-Side Request Forgery)
- ❌ Không whitelist allowed paths
- ❌ Có thể forward request đến internal services

**Fix:**
```typescript
const ALLOWED_PATHS = [
  '/auth',
  '/friends',
  '/writing-chat-bot',
  '/assignment-attachments',
];

function isPathAllowed(path: string): boolean {
  return ALLOWED_PATHS.some(allowed => path.startsWith(allowed));
}

async function handleRequest(request: NextRequest, method: string) {
  const backendUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:1611/api';
  const url = new URL(request.url);
  const path = url.pathname.replace('/api-proxy', '');
  
  // Validate path
  if (!isPathAllowed(path)) {
    return new Response(
      JSON.stringify({ status: false, message: 'Path not allowed', data: null }),
      { status: 403, headers: { 'Content-Type': 'application/json' } }
    );
  }
  
  // Prevent SSRF - validate target URL
  const targetUrl = `${backendUrl}${path}${url.search}`;
  const targetUrlObj = new URL(targetUrl);
  
  // Ensure target is from allowed backend
  if (targetUrlObj.hostname !== new URL(backendUrl).hostname) {
    return new Response(
      JSON.stringify({ status: false, message: 'Invalid target URL', data: null }),
      { status: 403, headers: { 'Content-Type': 'application/json' } }
    );
  }
  
  // ... rest of code
}
```

---

#### 1.2. Cookie Forwarding Security - `[...path]/route.ts`
**File:** `app/api-proxy/[...path]/route.ts`  
**Dòng:** 18-21, 52

**Vấn đề:**
```typescript
const cookie = request.headers.get('cookie');
if (cookie) headers['Cookie'] = cookie;

// ...

setCookies.forEach(c => responseHeaders.append('Set-Cookie', c.replace(/;\s*domain=[^;]*/gi, '')));
```

**Bug:**
- ❌ Forward tất cả cookies → có thể leak sensitive cookies
- ❌ Không validate cookie domain
- ❌ Domain replacement regex có thể không đủ

**Fix:**
```typescript
// Only forward specific cookies
const ALLOWED_COOKIE_NAMES = ['_u', 'access_token', 'refresh_token'];

function filterCookies(cookieHeader: string | null): string {
  if (!cookieHeader) return '';
  
  const cookies = cookieHeader.split(';').map(c => c.trim());
  const filtered = cookies.filter(cookie => {
    const name = cookie.split('=')[0];
    return ALLOWED_COOKIE_NAMES.includes(name);
  });
  
  return filtered.join('; ');
}

// In handleRequest:
const cookie = filterCookies(request.headers.get('cookie'));
if (cookie) headers['Cookie'] = cookie;

// When forwarding Set-Cookie:
setCookies.forEach(c => {
  // Remove domain, secure, httpOnly, sameSite to prevent issues
  const cleaned = c
    .replace(/;\s*domain=[^;]*/gi, '')
    .replace(/;\s*secure/gi, '')
    .replace(/;\s*httponly/gi, '')
    .replace(/;\s*samesite=[^;]*/gi, '');
  responseHeaders.append('Set-Cookie', cleaned);
});
```

---

#### 1.3. No Rate Limiting - Tất cả routes
**File:** Tất cả files trong `app/api-proxy`

**Vấn đề:**
- ❌ Không có rate limiting → có thể bị DDoS
- ❌ Không giới hạn request size
- ❌ Không có request throttling

**Fix:**
```typescript
// Create a rate limiter utility
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, "1 m"), // 100 requests per minute
});

async function handleRequest(request: NextRequest, method: string) {
  // Get client identifier
  const ip = request.headers.get("x-forwarded-for") || 
             request.headers.get("x-real-ip") || 
             "unknown";
  
  // Check rate limit
  const { success, limit, remaining, reset } = await ratelimit.limit(`api-proxy:${ip}`);
  
  if (!success) {
    return new Response(
      JSON.stringify({ 
        status: false, 
        message: `Rate limit exceeded. Try again in ${reset} seconds.`,
        data: null 
      }),
      { 
        status: 429,
        headers: { 
          'Content-Type': 'application/json',
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
        }
      }
    );
  }
  
  // ... rest of code
}
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Timeout Race Condition - `[...path]/route.ts`
**File:** `app/api-proxy/[...path]/route.ts`  
**Dòng:** 28-36

**Vấn đề:**
```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 30000);

const response = await fetch(targetUrl, {
  method,
  headers,
  body: body || undefined,
  signal: controller.signal,
}).finally(() => clearTimeout(timeoutId));
```

**Bug:**
- ❌ Nếu fetch complete trước timeout → `clearTimeout` trong `finally` OK
- ❌ Nhưng nếu timeout xảy ra → `clearTimeout` vẫn chạy nhưng có thể có race condition
- ❌ Không handle timeout error properly

**Fix:**
```typescript
const controller = new AbortController();
let timeoutId: NodeJS.Timeout | null = null;

try {
  timeoutId = setTimeout(() => {
    controller.abort();
  }, 30000);

  const response = await fetch(targetUrl, {
    method,
    headers,
    body: body || undefined,
    signal: controller.signal,
  });
  
  if (timeoutId) {
    clearTimeout(timeoutId);
    timeoutId = null;
  }
  
  // ... process response
} catch (error: any) {
  if (timeoutId) {
    clearTimeout(timeoutId);
  }
  
  if (error.name === 'AbortError') {
    return new Response(
      JSON.stringify({ status: false, message: 'Request timeout', data: null }),
      { status: 504, headers: { 'Content-Type': 'application/json' } }
    );
  }
  throw error;
}
```

---

#### 2.2. Body Size Limit - `writing-chat-bot/generate/route.ts`
**File:** `app/api-proxy/writing-chat-bot/generate/route.ts`  
**Dòng:** 9

**Vấn đề:**
```typescript
const body = await request.json();
```

**Bug:**
- ❌ Không check body size → có thể bị DoS với large payload
- ❌ Không validate body structure

**Fix:**
```typescript
const MAX_BODY_SIZE = 10 * 1024 * 1024; // 10MB

export async function POST(request: NextRequest) {
  try {
    // Check content-length
    const contentLength = request.headers.get('content-length');
    if (contentLength && parseInt(contentLength) > MAX_BODY_SIZE) {
      return NextResponse.json(
        { status: 500, message: 'Request body too large' },
        { status: 413 }
      );
    }
    
    // Read body with size limit
    const bodyText = await request.text();
    if (bodyText.length > MAX_BODY_SIZE) {
      return NextResponse.json(
        { status: 500, message: 'Request body too large' },
        { status: 413 }
      );
    }
    
    const body = JSON.parse(bodyText);
    
    // Validate body structure
    if (!body || typeof body !== 'object') {
      return NextResponse.json(
        { status: 500, message: 'Invalid request body' },
        { status: 400 }
      );
    }
    
    // ... rest of code
  }
}
```

---

### 3. **ERROR HANDLING BUGS**

#### 3.1. Inconsistent Error Responses - Tất cả routes
**File:** Tất cả files trong `app/api-proxy`

**Vấn đề:**
- ❌ Mỗi route có format error response khác nhau
- ❌ Không có consistent error handling
- ❌ Một số route return `status: false`, một số return `status: 500`

**Fix:**
```typescript
// Create a utility for consistent error responses
function createErrorResponse(
  message: string, 
  status: number = 500,
  error?: unknown
): NextResponse {
  const errorResponse = {
    status: false,
    message,
    data: null,
    ...(process.env.NODE_ENV === 'development' && error && { 
      error: error instanceof Error ? error.message : String(error) 
    }),
  };
  
  return NextResponse.json(errorResponse, { status });
}

// Usage:
catch (error: any) {
  if (error.name === 'AbortError') {
    return createErrorResponse('Request timeout', 504, error);
  }
  if (error.message?.includes('ECONNREFUSED')) {
    return createErrorResponse('Backend connection failed', 503, error);
  }
  return createErrorResponse('Internal server error', 500, error);
}
```

---

#### 3.2. Missing Error Logging - Tất cả routes
**Vấn đề:**
- ❌ Chỉ log trong development
- ❌ Không có structured logging
- ❌ Không track errors for monitoring

**Fix:**
```typescript
// Create logging utility
function logError(error: unknown, context: { route: string; method: string }) {
  const errorInfo = {
    route: context.route,
    method: context.method,
    error: error instanceof Error ? {
      name: error.name,
      message: error.message,
      stack: error.stack,
    } : String(error),
    timestamp: new Date().toISOString(),
  };
  
  // In production, send to monitoring service (Sentry, LogRocket, etc.)
  if (process.env.NODE_ENV === 'production') {
    // Send to monitoring service
    // Sentry.captureException(error, { extra: errorInfo });
  } else {
    console.error('API Proxy Error:', errorInfo);
  }
}

// Usage in catch block:
catch (error: any) {
  logError(error, { route: path, method });
  return createErrorResponse('Internal server error', 500, error);
}
```

---

## 🟡 WARNING ISSUES - app/api-proxy

### 4. **CODE QUALITY**

#### 4.1. Code Duplication - Tất cả routes
**Vấn đề:**
- ❌ Mỗi route có code tương tự nhau
- ❌ Không có shared utility functions
- ❌ Hard to maintain

**Fix:**
```typescript
// Create shared proxy utility
// app/api-proxy/utils/proxyHandler.ts

export interface ProxyOptions {
  timeout?: number;
  allowedPaths?: string[];
  maxBodySize?: number;
}

export async function proxyRequest(
  request: NextRequest,
  method: string,
  options: ProxyOptions = {}
) {
  const {
    timeout = 30000,
    allowedPaths = [],
    maxBodySize = 10 * 1024 * 1024,
  } = options;
  
  // Shared logic here
  // ...
}

// Then use in routes:
export const GET = (req: NextRequest) => proxyRequest(req, 'GET', {
  timeout: 10000,
  allowedPaths: ['/auth'],
});
```

---

#### 4.2. Magic Numbers - Tất cả routes
**Vấn đề:**
```typescript
setTimeout(() => controller.abort(), 30000); // Magic number
setTimeout(() => controller.abort(), 10000); // Different in different files
setTimeout(() => controller.abort(), 60000); // Another different value
```

**Fix:**
```typescript
// Create constants file
// app/api-proxy/constants.ts

export const TIMEOUTS = {
  DEFAULT: 30000,      // 30 seconds
  PROFILE: 10000,      // 10 seconds
  AI_GENERATION: 60000, // 60 seconds
} as const;

// Usage:
setTimeout(() => controller.abort(), TIMEOUTS.DEFAULT);
```

---

## 📈 PERFORMANCE METRICS - app/admin & app/api-proxy

### API Proxy Analysis

| Route | Timeout | Issues |
|-------|---------|--------|
| `[...path]` | 30s | No validation, SSRF risk |
| `auth/profile` | 10s | ✅ Good timeout |
| `auth/refresh` | 10s | ✅ Good timeout |
| `writing-chat-bot/generate` | 60s | No body size limit |
| `friends/requests/[id]/accept` | 30s | No input validation |

### Security Analysis

| Issue | Severity | Impact |
|-------|----------|--------|
| SSRF vulnerability | 🔴 Critical | Can access internal services |
| No rate limiting | 🔴 Critical | DDoS risk |
| Cookie forwarding | 🔴 Critical | Cookie leak |
| No input validation | 🔴 Critical | Injection attacks |
| Inconsistent errors | 🟡 Warning | Poor UX |

---

## ✅ PRIORITY FIX LIST - app/admin & app/api-proxy

### 🔴 Critical (Fix ngay)
1. **SSRF vulnerability** trong `[...path]/route.ts` - Add path validation
2. **Rate limiting** trong tất cả routes - Prevent DDoS
3. **Cookie security** trong `[...path]/route.ts` - Filter cookies
4. **Input validation** trong tất cả routes - Prevent injection
5. **Error handling** - Consistent error responses

### 🟡 High (Fix sớm)
6. **Race conditions** trong `admin/page.tsx` - Add cleanup
7. **Code duplication** trong api-proxy - Create shared utilities
8. **Type safety** - Remove `any` types

### 🟢 Medium (Cải thiện)
9. **Magic numbers** - Extract constants
10. **Logging** - Add structured logging
11. **Monitoring** - Add error tracking

---

## 📝 SUMMARY - app/admin & app/api-proxy

### Tổng kết
- **Critical bugs:** 13 issues cần fix ngay
- **Security issues:** 8 issues ảnh hưởng bảo mật
- **Code quality:** 9 issues cần cải thiện

### Điểm mạnh
- ✅ Có timeout handling
- ✅ Có AbortController cho cancellation
- ✅ Có transaction queue trong admin (prevent race conditions)
- ✅ Code structure tương đối tốt

### Điểm yếu
- ❌ Không có input validation
- ❌ Không có rate limiting
- ❌ SSRF vulnerability
- ❌ Cookie forwarding không an toàn
- ❌ Inconsistent error handling
- ❌ Code duplication

---

## 🔧 RECOMMENDED ACTIONS - app/admin & app/api-proxy

1. **Immediate:**
   - Fix SSRF vulnerability với path validation
   - Add rate limiting cho tất cả routes
   - Secure cookie forwarding
   - Add input validation

2. **Short-term:**
   - Create shared proxy utilities
   - Fix race conditions trong admin
   - Improve error handling consistency
   - Add structured logging

3. **Long-term:**
   - Add comprehensive monitoring
   - Implement request throttling
   - Add API documentation
   - Create integration tests

---

## 📁 PHÂN TÍCH: app/auth

### Tổng quan
- **Tổng số files:** 2 files
- **Files có vấn đề:** 2 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 4 issues
  - 🟡 **Warning:** 3 issues
  - 🟢 **Info:** 1 issue

---

## 🔴 CRITICAL ISSUES - app/auth

### 1. **SECURITY BUGS**

#### 1.1. Password in Plain Text - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 43-73, 75-110

**Vấn đề:**
```typescript
const handleSignIn = async (values: any) => {
  const response = await signIn({
    emailOrUsername: values.email,
    password: values.password, // Plain text password
    device_name: deviceName,
  });
};
```

**Bug:**
- ❌ Password được gửi plain text (mặc dù qua HTTPS)
- ❌ Không có client-side hashing (optional but recommended)
- ❌ Password có thể leak trong logs/network tab

**Fix:**
```typescript
// Option 1: Use HTTPS only (current - acceptable)
// Option 2: Add client-side hashing (extra security layer)
import CryptoJS from 'crypto-js';

const handleSignIn = async (values: any) => {
  // Hash password on client (optional extra layer)
  const hashedPassword = CryptoJS.SHA256(values.password).toString();
  
  const response = await signIn({
    emailOrUsername: values.email,
    password: hashedPassword, // Or keep plain if backend handles hashing
    device_name: deviceName,
  });
};
```

**Note:** Backend nên handle password hashing, nhưng client-side hashing là extra layer.

---

#### 1.2. Username Generation Logic - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 79

**Vấn đề:**
```typescript
const username = values.email.split("@")[0] || values.name.toLowerCase().replace(/\s+/g, "_");
```

**Bug:**
- ❌ Username có thể trùng lặp
- ❌ Không validate uniqueness
- ❌ Có thể tạo username không hợp lệ

**Fix:**
```typescript
const generateUsername = (email: string, name: string): string => {
  // Extract from email
  let baseUsername = email.split("@")[0];
  
  // Sanitize
  baseUsername = baseUsername
    .toLowerCase()
    .replace(/[^a-z0-9_]/g, "_")
    .replace(/_+/g, "_")
    .substring(0, 20); // Limit length
  
  // Fallback to name
  if (!baseUsername || baseUsername.length < 3) {
    baseUsername = name
      .toLowerCase()
      .replace(/\s+/g, "_")
      .replace(/[^a-z0-9_]/g, "_")
      .substring(0, 20);
  }
  
  // Add random suffix to reduce collisions
  const suffix = Math.random().toString(36).substring(2, 6);
  return `${baseUsername}_${suffix}`;
};

const username = generateUsername(values.email, values.name);
```

---

#### 1.3. No Rate Limiting on Client - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 43-73

**Vấn đề:**
- ❌ User có thể spam login requests
- ❌ Không có debounce/throttle
- ❌ Có thể bị brute force attack

**Fix:**
```typescript
const [isSubmitting, setIsSubmitting] = useState(false);
const [attemptCount, setAttemptCount] = useState(0);
const lastAttemptRef = useRef<number>(0);

const handleSignIn = async (values: any) => {
  // Rate limiting
  const now = Date.now();
  const timeSinceLastAttempt = now - lastAttemptRef.current;
  
  if (timeSinceLastAttempt < 1000) { // 1 second between attempts
    message.warning("Vui lòng đợi một chút trước khi thử lại");
    return;
  }
  
  // Check attempt count
  if (attemptCount >= 5) {
    message.error("Quá nhiều lần thử. Vui lòng thử lại sau 5 phút.");
    return;
  }
  
  if (isSubmitting) return;
  
  setIsSubmitting(true);
  lastAttemptRef.current = now;
  
  try {
    // ... existing code
  } catch (error) {
    setAttemptCount(prev => prev + 1);
    // ... error handling
  } finally {
    setIsSubmitting(false);
  }
};
```

---

#### 1.4. Force Reload Security - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 60-63, 97-100

**Vấn đề:**
```typescript
setTimeout(() => {
  window.location.href = "/profile";
}, 500);
```

**Bug:**
- ❌ Force reload có thể mất state
- ❌ Không check nếu redirect URL hợp lệ
- ❌ Có thể bị redirect hijacking

**Fix:**
```typescript
// Use Next.js router instead
import { useRouter } from "next/navigation";

const router = useRouter();

// After successful login:
if (response.status && response.data?.user) {
  message.success("Đăng nhập thành công!");
  
  // Clear any sensitive data
  // Use router.push instead of window.location
  router.push("/profile");
  // Or use router.replace for history
  router.replace("/profile");
}
```

---

### 2. **STATE & DATA FLOW BUGS**

#### 2.1. Race Condition - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 23-30

**Vấn đề:**
```typescript
useEffect(() => {
  const user = getCurrentUser();
  const token = typeof window !== "undefined" ? localStorage.getItem("accessToken") : null;

  if (user && token) {
    router.push("/profile");
  }
}, [router]);
```

**Bug:**
- ❌ Check localStorage mỗi render
- ❌ Có thể redirect trước khi component mount xong
- ❌ Không cleanup nếu component unmount

**Fix:**
```typescript
useEffect(() => {
  let isMounted = true;
  
  const checkAuth = async () => {
    // Wait a bit to ensure cookies are set
    await new Promise(resolve => setTimeout(resolve, 100));
    
    if (!isMounted) return;
    
    const user = getCurrentUser();
    const token = typeof window !== "undefined" ? localStorage.getItem("accessToken") : null;

    if (user && token) {
      router.push("/profile");
    }
  };
  
  checkAuth();
  
  return () => {
    isMounted = false;
  };
}, [router]);
```

---

## 🟡 WARNING ISSUES - app/auth

### 3. **CODE QUALITY**

#### 3.1. Type Safety - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 43, 75

**Vấn đề:**
```typescript
const handleSignIn = async (values: any) => {
const handleSignUp = async (values: any) => {
```

**Fix:**
```typescript
interface SignInValues {
  email: string;
  password: string;
  remember?: boolean;
}

interface SignUpValues {
  name: string;
  email: string;
  phone: string;
  password: string;
  confirmPassword: string;
  role_id?: number;
  agreement: boolean;
}

const handleSignIn = async (values: SignInValues) => {
  // ...
};

const handleSignUp = async (values: SignUpValues) => {
  // ...
};
```

---

#### 3.2. Magic Numbers - `auth/page.tsx`
**File:** `app/auth/page.tsx`  
**Dòng:** 60, 97

**Vấn đề:**
```typescript
setTimeout(() => {
  window.location.href = "/profile";
}, 500); // Magic number
```

**Fix:**
```typescript
const REDIRECT_DELAY_MS = 500;

setTimeout(() => {
  router.push("/profile");
}, REDIRECT_DELAY_MS);
```

---

## 📁 PHÂN TÍCH: app/config

### Tổng quan
- **Tổng số files:** 1 file
- **Files có vấn đề:** 1 file
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 3 issues
  - 🟡 **Warning:** 2 issues
  - 🟢 **Info:** 1 issue

---

## 🔴 CRITICAL ISSUES - app/config

### 1. **SECURITY BUGS**

#### 1.1. Auth Cache TTL Too Short - `config/api.ts`
**File:** `app/config/api.ts`  
**Dòng:** 28

**Vấn đề:**
```typescript
const AUTH_CACHE_TTL = 500; // 500ms cache
```

**Bug:**
- ❌ Cache TTL quá ngắn → không hiệu quả
- ❌ Có thể gây race condition nếu nhiều requests cùng lúc
- ❌ Token có thể stale trong cache

**Fix:**
```typescript
const AUTH_CACHE_TTL = 5000; // 5 seconds - more reasonable

// Or use a smarter cache invalidation
const getCachedAuthHeader = (): string | null => {
  if (typeof window === "undefined") return null;
  const now = Date.now();
  
  // Check if cache is still valid
  if (cachedAuthHeader && now - cachedAuthTimestamp < AUTH_CACHE_TTL) {
    // Verify token still exists
    const atCookie = getCookie("_at");
    if (atCookie && atCookie === cachedAuthHeader.replace("Bearer ", "")) {
      return cachedAuthHeader;
    }
    // Token changed - invalidate cache
    cachedAuthHeader = null;
  }
  
  const atCookie = getCookie("_at");
  if (atCookie) {
    cachedAuthHeader = `Bearer ${atCookie}`;
    cachedAuthTimestamp = now;
    return cachedAuthHeader;
  }
  
  cachedAuthHeader = null;
  return null;
};
```

---

#### 1.2. Response Cache Memory Leak - `config/api.ts`
**File:** `app/config/api.ts`  
**Dòng:** 83-120

**Vấn đề:**
```typescript
const responseCache = new Map<string, { data: any; ts: number }>();

// In interceptor:
if (responseCache.size > 50) {
  const entries = [...responseCache.entries()].sort((a, b) => a[1].ts - b[1].ts);
  for (let i = 0; i < 10; i++) responseCache.delete(entries[i][0]);
}
```

**Bug:**
- ❌ Cache có thể grow indefinitely nếu không cleanup đúng
- ❌ LRU eviction không đủ aggressive
- ❌ Memory leak trong long-running sessions

**Fix:**
```typescript
const MAX_CACHE_SIZE = 50;
const CACHE_CLEANUP_THRESHOLD = 40; // Start cleanup earlier

// Periodic cleanup
setInterval(() => {
  const now = Date.now();
  const entries = [...responseCache.entries()];
  
  // Remove expired entries
  entries.forEach(([key, value]) => {
    if (now - value.ts > CACHE_TTL) {
      responseCache.delete(key);
    }
  });
  
  // If still too large, remove oldest
  if (responseCache.size > MAX_CACHE_SIZE) {
    const sorted = entries
      .filter(([_, value]) => now - value.ts <= CACHE_TTL)
      .sort((a, b) => a[1].ts - b[1].ts);
    
    const toRemove = sorted.slice(0, responseCache.size - CACHE_CLEANUP_THRESHOLD);
    toRemove.forEach(([key]) => responseCache.delete(key));
  }
}, 30000); // Cleanup every 30 seconds
```

---

#### 1.3. Token Refresh Race Condition - `config/api.ts`
**File:** `app/config/api.ts`  
**Dòng:** 74-80, 160-170

**Vấn đề:**
```typescript
let isRefreshing = false;
let failedQueue: Array<{ resolve: (v?: any) => void; reject: (e?: any) => void }> = [];

// Queue if already refreshing
if (isRefreshing) {
  return new Promise((resolve, reject) => {
    failedQueue.push({ resolve, reject });
  })
```

**Bug:**
- ❌ Queue có thể grow indefinitely
- ❌ Không có timeout cho queued requests
- ❌ Memory leak nếu queue không được process

**Fix:**
```typescript
const MAX_QUEUE_SIZE = 100;
const QUEUE_TIMEOUT = 30000; // 30 seconds

if (isRefreshing) {
  if (failedQueue.length >= MAX_QUEUE_SIZE) {
    return Promise.reject(new Error("Too many queued requests"));
  }
  
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      reject(new Error("Request timeout - token refresh taking too long"));
    }, QUEUE_TIMEOUT);
    
    failedQueue.push({ 
      resolve: (value) => {
        clearTimeout(timeoutId);
        resolve(value);
      }, 
      reject: (error) => {
        clearTimeout(timeoutId);
        reject(error);
      }
    });
  })
}
```

---

## 🟡 WARNING ISSUES - app/config

### 2. **CODE QUALITY**

#### 2.1. Magic Numbers - `config/api.ts`
**File:** `app/config/api.ts`  
**Dòng:** 20, 28, 84

**Vấn đề:**
```typescript
timeout: 30000,
const AUTH_CACHE_TTL = 500;
const CACHE_TTL = 30000;
```

**Fix:**
```typescript
const API_TIMEOUT_MS = 30000;
const AUTH_CACHE_TTL_MS = 5000;
const RESPONSE_CACHE_TTL_MS = 30000;

apiClient = axios.create({
  timeout: API_TIMEOUT_MS,
  // ...
});
```

---

## 📁 PHÂN TÍCH: app/components

### Tổng quan
- **Tổng số files:** ~100+ files
- **Files có vấn đề:** ~30 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 8 issues
  - 🟡 **Warning:** 12 issues
  - 🟢 **Info:** 5 issues

---

## 🔴 CRITICAL ISSUES - app/components

### 1. **SECURITY BUGS**

#### 1.1. XSS Risk in RichTextEditor - `components/common/RichTextEditor.tsx`
**File:** `app/components/common/RichTextEditor.tsx`  
**Dòng:** 46-63, 65-70

**Vấn đề:**
```typescript
const toggleLink = () => {
  const url = window.prompt("URL:", previousUrl);
  // ...
  editor.chain().focus().extendMarkRange("link").setLink({ href: url }).run();
};

const addImage = () => {
  const url = window.prompt("URL hình ảnh:");
  if (url) {
    editor.chain().focus().setImage({ src: url }).run();
  }
};
```

**Bug:**
- ❌ Không validate URL → có thể inject javascript: URLs
- ❌ Không sanitize input
- ❌ XSS risk với malicious URLs

**Fix:**
```typescript
const isValidUrl = (url: string): boolean => {
  try {
    const parsed = new URL(url);
    // Only allow http, https
    return ['http:', 'https:'].includes(parsed.protocol);
  } catch {
    return false;
  }
};

const toggleLink = () => {
  const previousUrl = editor.getAttributes("link").href;
  const url = window.prompt("URL:", previousUrl);

  if (url === null) return;
  if (url === "") {
    editor.chain().focus().extendMarkRange("link").unsetLink().run();
    return;
  }

  // Validate URL
  if (!isValidUrl(url)) {
    message.error("URL không hợp lệ. Chỉ chấp nhận http:// hoặc https://");
    return;
  }

  editor.chain().focus().extendMarkRange("link").setLink({ href: url }).run();
};

const addImage = () => {
  const url = window.prompt("URL hình ảnh:");
  if (!url) return;
  
  // Validate URL
  if (!isValidUrl(url)) {
    message.error("URL không hợp lệ. Chỉ chấp nhận http:// hoặc https://");
    return;
  }
  
  editor.chain().focus().setImage({ src: url }).run();
};
```

---

#### 1.2. No Input Sanitization - `components/common/CustomInput.tsx`
**File:** `app/components/common/CustomInput.tsx`  
**Dòng:** 41-58

**Vấn đề:**
```typescript
<input
  type="text"
  placeholder={placeholder}
  value={value}
  onChange={onChange}
  // No sanitization
/>
```

**Bug:**
- ❌ Không sanitize input
- ❌ Có thể inject HTML/scripts nếu value được render unsafely

**Fix:**
```typescript
import DOMPurify from 'dompurify';

const sanitizeInput = (value: string): string => {
  return DOMPurify.sanitize(value, { 
    ALLOWED_TAGS: [],
    ALLOWED_ATTR: [] 
  });
};

<input
  type="text"
  placeholder={placeholder}
  value={sanitizeInput(value || '')}
  onChange={(e) => {
    const sanitized = sanitizeInput(e.target.value);
    onChange?.({ ...e, target: { ...e.target, value: sanitized } });
  }}
/>
```

---

### 2. **MEMORY LEAK**

#### 2.1. IntersectionObserver Not Cleaned Up - `components/common/ScrollAnimation.tsx`
**File:** `app/components/common/ScrollAnimation.tsx`  
**Dòng:** 21-50

**Vấn đề:**
```typescript
useEffect(() => {
  setIsVisible(false);
  
  const currentRef = ref.current;
  if (!currentRef) return;

  const observer = new IntersectionObserver(
    // ...
  );

  observer.observe(currentRef);

  return () => {
    observer.disconnect();
  };
}, [delay]);
```

**Bug:**
- ❌ Dependency `delay` → observer recreated mỗi khi delay thay đổi
- ❌ Có thể có multiple observers nếu delay changes frequently

**Fix:**
```typescript
useEffect(() => {
  setIsVisible(false);
  
  const currentRef = ref.current;
  if (!currentRef) return;

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          setTimeout(() => {
            setIsVisible(true);
          }, delay);
          observer.unobserve(entry.target);
        }
      });
    },
    {
      threshold: 0.1,
      rootMargin: "0px 0px -50px 0px",
    }
  );

  observer.observe(currentRef);

  return () => {
    observer.disconnect();
  };
}, []); // Remove delay from dependencies, handle it in callback
```

---

#### 2.2. Editor Instance Not Cleaned Up - `components/common/RichTextEditor.tsx`
**File:** `app/components/common/RichTextEditor.tsx`  
**Dòng:** 279-381

**Vấn đề:**
```typescript
const editor = useEditor({
  // ... config
});

useEffect(() => {
  if (editor && onEditorReadyRef.current) {
    onEditorReadyRef.current(editor);
    const timer = setTimeout(() => {
      if (onEditorReadyRef.current) {
        onEditorReadyRef.current(editor);
      }
    }, 100);
    return () => clearTimeout(timer);
  }
}, [editor]);
```

**Bug:**
- ❌ Editor không được destroy khi component unmount
- ❌ Memory leak với TipTap editor instances

**Fix:**
```typescript
useEffect(() => {
  return () => {
    if (editor) {
      editor.destroy();
    }
  };
}, [editor]);
```

---

### 3. **PERFORMANCE BUGS**

#### 3.1. Unnecessary Re-renders - `components/common/ScrollAnimation.tsx`
**File:** `app/components/common/ScrollAnimation.tsx`  
**Dòng:** 18-50

**Vấn đề:**
```typescript
const [isVisible, setIsVisible] = useState(false);

useEffect(() => {
  setIsVisible(false); // Reset on every delay change
  // ...
}, [delay]);
```

**Bug:**
- ❌ Reset visibility mỗi khi delay thay đổi → unnecessary re-render
- ❌ Observer recreated mỗi render

**Fix:**
```typescript
const [isVisible, setIsVisible] = useState(false);
const delayRef = useRef(delay);

useEffect(() => {
  delayRef.current = delay;
}, [delay]);

useEffect(() => {
  const currentRef = ref.current;
  if (!currentRef) return;

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          setTimeout(() => {
            setIsVisible(true);
          }, delayRef.current); // Use ref instead
          observer.unobserve(entry.target);
        }
      });
    },
    {
      threshold: 0.1,
      rootMargin: "0px 0px -50px 0px",
    }
  );

  observer.observe(currentRef);

  return () => {
    observer.disconnect();
  };
}, []); // Only create observer once
```

---

#### 3.2. Large Component - `components/common/RichTextEditor.tsx`
**File:** `app/components/common/RichTextEditor.tsx`  
**Dòng:** 272-467

**Vấn đề:** Component quá lớn (467 dòng) → khó maintain, re-render toàn bộ

**Fix:** Split thành smaller components:
- `RichTextEditorMenuBar.tsx`
- `RichTextEditorContent.tsx`
- `RichTextEditorStyles.tsx`

---

## 🟡 WARNING ISSUES - app/components

### 4. **CODE QUALITY**

#### 4.1. Type Safety - `components/common/PrefetchLink.tsx`
**File:** `app/components/common/PrefetchLink.tsx`  
**Dòng:** 11

**Vấn đề:**
```typescript
[key: string]: any;
```

**Fix:**
```typescript
interface PrefetchLinkProps extends React.ComponentProps<typeof Link> {
  href: string;
  children: ReactNode;
  className?: string;
}
```

---

#### 4.2. Console.log in Production - `components/common/PrefetchLink.tsx`
**File:** `app/components/common/PrefetchLink.tsx`  
**Dòng:** 26-28

**Vấn đề:**
```typescript
if (process.env.NODE_ENV === 'development') {
  console.log(`🚀 [Prefetch] Hovering over: ${href}`);
}
```

**Fix:**
```typescript
// Use a proper logging utility
import { logger } from '@/lib/utils/logger';

onMouseEnter={() => {
  logger.debug('Prefetch', { href });
  router.prefetch(href);
}}
```

---

## 📈 PERFORMANCE METRICS - app/auth, app/config, app/components

### Component Analysis

| Component | Size | Issues |
|-----------|------|--------|
| `auth/page.tsx` | 393 lines | Security, race conditions |
| `config/api.ts` | 250 lines | Memory leaks, cache issues |
| `RichTextEditor.tsx` | 467 lines | XSS risk, memory leak |
| `ScrollAnimation.tsx` | 81 lines | Memory leak, re-renders |

### Security Analysis

| Issue | Severity | Impact |
|-------|----------|--------|
| Password plain text | 🔴 Critical | Password leak risk |
| XSS in RichTextEditor | 🔴 Critical | Code injection |
| No input sanitization | 🔴 Critical | XSS risk |
| Auth cache issues | 🔴 Critical | Token leak |

---

## ✅ PRIORITY FIX LIST - app/auth, app/config, app/components

### 🔴 Critical (Fix ngay)
1. **XSS risk** trong `RichTextEditor.tsx` - Validate URLs
2. **Memory leaks** trong `config/api.ts` - Fix cache cleanup
3. **Memory leaks** trong `ScrollAnimation.tsx` - Fix observer cleanup
4. **Token refresh race** trong `config/api.ts` - Fix queue management

### 🟡 High (Fix sớm)
5. **Password security** trong `auth/page.tsx` - Add rate limiting
6. **Input sanitization** trong `CustomInput.tsx` - Add sanitization
7. **Type safety** - Remove `any` types

### 🟢 Medium (Cải thiện)
8. **Component splitting** - Split large components
9. **Code duplication** - Extract shared utilities
10. **Logging** - Use proper logging utility

---

## 📝 SUMMARY - app/auth, app/config, app/components

### Tổng kết
- **Critical bugs:** 15 issues cần fix ngay
- **Security issues:** 7 issues ảnh hưởng bảo mật
- **Performance issues:** 5 issues ảnh hưởng UX
- **Code quality:** 14 issues cần cải thiện

### Điểm mạnh
- ✅ Có memoization trong một số components
- ✅ Có error handling
- ✅ Code structure tương đối tốt
- ✅ Có loading states

### Điểm yếu
- ❌ XSS vulnerabilities
- ❌ Memory leaks trong observers và caches
- ❌ Không có input sanitization
- ❌ Race conditions trong auth flow
- ❌ Type safety issues

---

## 🔧 RECOMMENDED ACTIONS - app/auth, app/config, app/components

1. **Immediate:**
   - Fix XSS vulnerabilities
   - Fix memory leaks
   - Add input sanitization
   - Fix race conditions

2. **Short-term:**
   - Improve type safety
   - Split large components
   - Add proper logging
   - Improve error handling

3. **Long-term:**
   - Add comprehensive tests
   - Implement proper monitoring
   - Add security audits
   - Optimize bundle size

---

## 📁 PHÂN TÍCH: app/context

### Tổng quan
- **Tổng số files:** 1 file
- **Files có vấn đề:** 1 file
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 2 issues
  - 🟡 **Warning:** 2 issues
  - 🟢 **Info:** 1 issue

---

## 🔴 CRITICAL ISSUES - app/context

### 1. **ASYNC / TIMING BUGS**

#### 1.1. Race Condition in ThemeContext - `context/ThemeContext.tsx`
**File:** `app/context/ThemeContext.tsx`  
**Dòng:** 49-120

**Vấn đề:**
```typescript
const toggleTheme = async (e?: React.MouseEvent) => {
  const newTheme = theme === "light" ? "dark" : "light";
  
  // ...
  await setThemeCookie(newTheme);
  // ...
};
```

**Bug:**
- ❌ User click nhanh → nhiều requests cùng lúc
- ❌ Race condition: Request A (light) và Request B (dark) có thể về sai thứ tự
- ❌ Không có debounce/throttle

**Fix:**
```typescript
const [isToggling, setIsToggling] = useState(false);

const toggleTheme = async (e?: React.MouseEvent) => {
  // Prevent double-click
  if (isToggling) return;
  
  setIsToggling(true);
  
  try {
    const newTheme = theme === "light" ? "dark" : "light";
    
    // ... rest of the code
    
    const result = await setThemeCookie(newTheme);
    if (!result.success) {
      console.error("Failed to set theme cookie:", result.error);
      // Fallback to client-side cookie
      document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
    }
  } catch (error) {
    console.error("Error toggling theme:", error);
    // Fallback handling
  } finally {
    setIsToggling(false);
  }
};
```

---

#### 1.2. Unhandled Promise Rejection - `context/ThemeContext.tsx`
**File:** `app/context/ThemeContext.tsx`  
**Dòng:** 85-87

**Vấn đề:**
```typescript
setThemeCookie(newTheme).catch(() => {
  document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
});
```

**Bug:**
- ❌ Silent failure → không log error
- ❌ Không notify user nếu server action fail
- ❌ Fallback có thể không hoạt động trong một số trường hợp

**Fix:**
```typescript
try {
  const result = await setThemeCookie(newTheme);
  if (!result.success) {
    console.warn("Server action failed, using client-side fallback:", result.error);
    // Fallback to client-side cookie
    document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
  }
} catch (error) {
  console.error("Error setting theme cookie:", error);
  // Fallback to client-side cookie
  document.cookie = `theme=${newTheme}; path=/; max-age=31536000; SameSite=Lax`;
}
```

---

## 🟡 WARNING ISSUES - app/context

### 2. **CODE QUALITY**

#### 2.1. View Transition API Compatibility - `context/ThemeContext.tsx`
**File:** `app/context/ThemeContext.tsx`  
**Dòng:** 53

**Vấn đề:**
```typescript
if (!(document as any).startViewTransition) {
```

**Fix:**
```typescript
// Add proper type definition
interface DocumentWithViewTransition extends Document {
  startViewTransition?: (callback: () => void) => {
    ready: Promise<void>;
    finished: Promise<void>;
    updateCallbackDone: Promise<void>;
    skipTransition: () => void;
  };
}

const hasViewTransition = (document as DocumentWithViewTransition).startViewTransition !== undefined;
```

---

## 📁 PHÂN TÍCH: app/hooks

### Tổng quan
- **Tổng số files:** 4 files
- **Files có vấn đề:** 4 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 6 issues
  - 🟡 **Warning:** 5 issues
  - 🟢 **Info:** 2 issues

---

## 🔴 CRITICAL ISSUES - app/hooks

### 1. **MEMORY LEAK**

#### 1.1. Event Listener Not Cleaned Up - `hooks/useUserId.ts`
**File:** `app/hooks/useUserId.ts`  
**Dòng:** 54-77

**Vấn đề:**
```typescript
useEffect(() => {
  // ...
  window.addEventListener("user_id_cached", handleCacheEvent);

  return () => {
    mounted = false;
    window.removeEventListener("user_id_cached", handleCacheEvent);
  };
}, []); // Chỉ chạy 1 lần khi mount
```

**Bug:**
- ❌ `handleCacheEvent` được tạo mới mỗi render → removeEventListener không match
- ❌ Event listener không được cleanup đúng cách

**Fix:**
```typescript
useEffect(() => {
  let mounted = true;

  const fetchUserId = async () => {
    // ... existing code
  };

  const handleCacheEvent = () => {
    if (!mounted) return;
    const sessionId = getUserIdFromSession();
    if (sessionId) {
      setUserId(sessionId);
      setLoading(false);
    } else {
      getUserIdFromCookieAsync().then((id) => {
        if (mounted) {
          setUserId(id);
          setLoading(false);
        }
      });
    }
  };

  fetchUserId();
  window.addEventListener("user_id_cached", handleCacheEvent);

  return () => {
    mounted = false;
    window.removeEventListener("user_id_cached", handleCacheEvent);
  };
}, []); // Stable dependencies
```

---

#### 1.2. Socket Not Disconnected - `hooks/useExamSocket.ts`
**File:** `app/hooks/useExamSocket.ts`  
**Dòng:** 19-68

**Vấn đề:**
```typescript
useEffect(() => {
  // ...
  const socket = io(SOCKET_URL, {
    transports: ["websocket"],
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
  });

  socketRef.current = socket;

  // ... event listeners

  return () => {
    socket.disconnect();
  };
}, [examId, attemptId, studentId]); // Re-connect if IDs change
```

**Bug:**
- ❌ Socket recreated mỗi khi IDs thay đổi → có thể có nhiều connections
- ❌ Event listeners không được cleanup trước khi disconnect
- ❌ Memory leak nếu component unmount giữa chừng

**Fix:**
```typescript
useEffect(() => {
  if ((!examId && !attemptId) || !studentId) return;

  // Disconnect previous socket if exists
  if (socketRef.current) {
    socketRef.current.removeAllListeners();
    socketRef.current.disconnect();
  }

  const socket = io(SOCKET_URL, {
    transports: ["websocket"],
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
  });

  socketRef.current = socket;

  const handleConnect = () => {
    console.log("✅ Socket Connected:", socket.id);
    setIsConnected(true);

    if (attemptId) {
      socket.emit("join_attempt", { attemptId });
    } else {
      socket.emit("join_exam", { examId, studentId });
    }

    if (onConnect) onConnect();
  };

  const handleDisconnect = () => {
    console.log("❌ Socket Disconnected");
    setIsConnected(false);
    if (onDisconnect) onDisconnect();
  };

  socket.on("connect", handleConnect);
  socket.on("disconnect", handleDisconnect);
  socket.on("join_success", (data: any) => {
    console.log("Exam/Attempt Joined:", data);
  });
  socket.on("violation_recorded", (data: any) => {
    console.log("Violation Ack:", data);
  });

  return () => {
    socket.off("connect", handleConnect);
    socket.off("disconnect", handleDisconnect);
    socket.off("join_success");
    socket.off("violation_recorded");
    socket.removeAllListeners();
    socket.disconnect();
    socketRef.current = null;
  };
}, [examId, attemptId, studentId, onConnect, onDisconnect]);
```

---

#### 1.3. Interval Not Cleaned Up - `hooks/useAntiCheat.ts`
**File:** `app/hooks/useAntiCheat.ts`  
**Dòng:** 195-198

**Vấn đề:**
```typescript
const interval = setInterval(checkDevTools, 2000);
window.addEventListener('resize', checkDevTools);
return () => { clearInterval(interval); window.removeEventListener('resize', checkDevTools); };
```

**Bug:**
- ✅ Đã có cleanup - Good!
- ⚠️ Nhưng `checkDevTools` được tạo mới mỗi render → có thể có memory leak nhỏ

**Fix:**
```typescript
const checkDevToolsRef = useRef(checkDevTools);

useEffect(() => {
  checkDevToolsRef.current = checkDevTools;
}, [checkDevTools]);

useEffect(() => {
  if (!enable || paused) return;
  
  const checkDevTools = () => {
    if (paused) return;
    checkDevToolsRef.current();
  };
  
  const interval = setInterval(checkDevTools, 2000);
  window.addEventListener('resize', checkDevTools);
  
  return () => { 
    clearInterval(interval); 
    window.removeEventListener('resize', checkDevTools); 
  };
}, [enable, paused]); // Remove checkDevTools from dependencies
```

---

### 2. **SECURITY BUGS**

#### 2.1. XSS Risk in Overlay HTML - `hooks/useAntiCheat.ts`
**File:** `app/hooks/useAntiCheat.ts`  
**Dòng:** 101-108

**Vấn đề:**
```typescript
div.innerHTML = `
  <div style="font-size: 60px; margin-bottom: 20px;">${icon}</div>
  <h2 style="color: ${titleColor}; font-weight: bold; font-size: 32px; margin-bottom: 15px;">${title}</h2>
  <p id="overlay-msg" style="margin: 10px 0; font-size: 20px; line-height: 1.6;">${msg}</p>
  <button id="resume-btn" style="...">${btnText}</button>
`;
```

**Bug:**
- ❌ Dùng `innerHTML` với user input → XSS risk
- ❌ `msg`, `btnText` có thể chứa malicious code

**Fix:**
```typescript
// Use textContent and createElement instead
const titleEl = document.createElement('h2');
titleEl.textContent = title;
titleEl.style.cssText = `color: ${titleColor}; font-weight: bold; font-size: 32px; margin-bottom: 15px;`;

const msgEl = document.createElement('p');
msgEl.id = 'overlay-msg';
msgEl.textContent = msg; // textContent escapes HTML
msgEl.style.cssText = 'margin: 10px 0; font-size: 20px; line-height: 1.6;';

const btnEl = document.createElement('button');
btnEl.id = 'resume-btn';
btnEl.textContent = btnText;
btnEl.style.cssText = '...';

div.appendChild(titleEl);
div.appendChild(msgEl);
div.appendChild(btnEl);
```

---

#### 2.2. DOM Manipulation Security - `hooks/useAntiCheat.ts`
**File:** `app/hooks/useAntiCheat.ts`  
**Dòng:** 73-110

**Vấn đề:**
- ❌ Tạo DOM elements trực tiếp → có thể bị bypass
- ❌ Không có validation cho overlay content

**Fix:**
```typescript
// Use React Portal instead of direct DOM manipulation
import { createPortal } from 'react-dom';

const Overlay = ({ message, onResume, isHardBlock }: OverlayProps) => {
  return createPortal(
    <div className="fixed inset-0 z-[2147483647] bg-black/95 flex flex-col items-center justify-center text-white">
      {/* Safe React rendering */}
    </div>,
    document.body
  );
};
```

---

### 3. **ASYNC / TIMING BUGS**

#### 3.1. File Upload Race Condition - `hooks/useFileUpload.ts`
**File:** `app/hooks/useFileUpload.ts`  
**Dòng:** 24-88

**Vấn đề:**
```typescript
const uploadFile = useCallback(
  async (file: UploadFile, index: number, total: number): Promise<UploadResult> => {
    // ...
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 300000);
    
    try {
      const response = await fetch(`/api-proxy/assignment-attachments/${attachmentIdStr}?userId=${userId}`, {
        method: "PATCH",
        body: formData,
        signal: controller.signal,
      });
      clearTimeout(timeoutId);
      // ...
    } catch (error: any) {
      clearTimeout(timeoutId);
      // ...
    }
  },
  [assignmentId, userId, existingAttachment, onProgress]
);
```

**Bug:**
- ❌ Nếu component unmount trước khi request hoàn thành → timeout không được clear
- ❌ Multiple uploads có thể race condition

**Fix:**
```typescript
const uploadFile = useCallback(
  async (file: UploadFile, index: number, total: number): Promise<UploadResult> => {
    let timeoutId: NodeJS.Timeout | null = null;
    let isMounted = true;
    
    try {
      // ... existing code
      
      const controller = new AbortController();
      timeoutId = setTimeout(() => {
        controller.abort();
      }, 300000);
      
      const response = await fetch(/* ... */, {
        signal: controller.signal,
      });
      
      if (timeoutId) {
        clearTimeout(timeoutId);
        timeoutId = null;
      }
      
      if (!isMounted) {
        throw new Error("Component unmounted");
      }
      
      // ... rest of code
    } catch (error: any) {
      if (timeoutId) {
        clearTimeout(timeoutId);
      }
      if (error.name === "AbortError") {
        throw new Error(`Request timeout: Không thể cập nhật file ${file.name} sau 5 phút.`);
      }
      throw error;
    } finally {
      if (timeoutId) {
        clearTimeout(timeoutId);
      }
    }
  },
  [assignmentId, userId, existingAttachment, onProgress]
);
```

---

## 🟡 WARNING ISSUES - app/hooks

### 4. **CODE QUALITY**

#### 4.1. Magic Numbers - `hooks/useAntiCheat.ts`
**File:** `app/hooks/useAntiCheat.ts`  
**Dòng:** 54, 185, 195, 216

**Vấn đề:**
```typescript
if (paused || overlayRef.current || (now - lastViolationTimeRef.current < 1000)) return;
const threshold = 200;
const interval = setInterval(checkDevTools, 2000);
if (now - lastIncidentTimeRef.current < 1000) return;
```

**Fix:**
```typescript
const VIOLATION_COOLDOWN_MS = 1000;
const DEVTOOLS_THRESHOLD = 200;
const DEVTOOLS_CHECK_INTERVAL_MS = 2000;
const INCIDENT_COOLDOWN_MS = 1000;
```

---

#### 4.2. Type Safety - `hooks/useExamSocket.ts`
**File:** `app/hooks/useExamSocket.ts`  
**Dòng:** 16, 23-33

**Vấn đề:**
```typescript
const socketRef = useRef<any>(null);
const socket = (io as any).default
  ? (io as any).default(SOCKET_URL, {
```

**Fix:**
```typescript
import { Socket } from 'socket.io-client';

const socketRef = useRef<Socket | null>(null);

const socket = io(SOCKET_URL, {
  transports: ["websocket"],
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
});
```

---

## 📁 PHÂN TÍCH: app/social

### Tổng quan
- **Tổng số files:** 5 files
- **Files có vấn đề:** 3 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 7 issues
  - 🟡 **Warning:** 8 issues
  - 🟢 **Info:** 3 issues

---

## 🔴 CRITICAL ISSUES - app/social

### 1. **STATE & DATA FLOW BUGS**

#### 1.1. Stale Closure in SocialContext - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 1042

**Vấn đề:**
```typescript
}, [fetchContacts, currentUserIdNumber, fetchConversations]);
```

**Bug:**
- ❌ `fetchContacts` và `fetchConversations` là callbacks → có thể stale
- ❌ Dependencies không đầy đủ → có thể miss updates

**Fix:**
```typescript
// Use refs for stable references
const fetchContactsRef = useRef(fetchContacts);
const fetchConversationsRef = useRef(fetchConversations);

useEffect(() => {
  fetchContactsRef.current = fetchContacts;
  fetchConversationsRef.current = fetchConversations;
}, [fetchContacts, fetchConversations]);

// In socket effect:
useEffect(() => {
  // ... socket setup
  
  const handleMessage = (payload) => {
    // Use refs instead of direct calls
    if (!newConv) {
      fetchConversationsRef.current(); // Use ref
      return prev;
    }
  };
  
  // ...
}, []); // Empty deps, use refs inside
```

---

#### 1.2. Race Condition in Message Loading - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 381-436

**Vấn đề:**
```typescript
const loadMessages = useCallback(async (roomId: string) => {
  setLoadingMessages(true);
  try {
    // ...
    const result = await getMessages({...});
    setMessages(mappedMessages);
    // ...
  } catch (error) {
    console.error("Error loading messages:", error);
  } finally {
    setLoadingMessages(false);
  }
}, []);
```

**Bug:**
- ❌ Nếu user switch room nhanh → request A và B có thể về sai thứ tự
- ❌ Không có AbortController để cancel previous request

**Fix:**
```typescript
const loadMessages = useCallback(async (roomId: string) => {
  const currentRoomIdRef = roomId;
  
  setLoadingMessages(true);
  setMessages([]); // Clear immediately
  
  const controller = new AbortController();
  
  try {
    // ... existing code
    
    const result = await getMessages({
      userId: userIdNumber,
      roomId: roomIdNumber,
      limit: 50,
    }, { signal: controller.signal });
    
    // Only update if still on the same room
    if (currentRoomIdRef === roomId) {
      const mappedMessages: Message[] = result.data.map((msg: any) => ({
        // ... mapping
      }));
      setMessages(mappedMessages);
    }
  } catch (error: any) {
    if (error.name !== 'AbortError') {
      console.error("Error loading messages:", error);
    }
  } finally {
    if (currentRoomIdRef === roomId) {
      setLoadingMessages(false);
    }
  }
  
  return () => {
    controller.abort();
  };
}, []);
```

---

#### 1.3. Message Deduplication Logic - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 788-793, 902-927

**Vấn đề:**
```typescript
const processedMessageIdsRef = React.useRef(new Set<string>());

// Hard Deduplication for Socket Events
const msgId = String(msg.message_id || "");
if (msgId && processedMessageIdsRef.current.has(msgId)) {
  return;
}
if (msgId) processedMessageIdsRef.current.add(msgId);
```

**Bug:**
- ❌ Set có thể grow indefinitely → memory leak
- ❌ Không cleanup old message IDs

**Fix:**
```typescript
const MAX_PROCESSED_IDS = 1000;

const processedMessageIdsRef = React.useRef(new Set<string>());

// Cleanup old IDs periodically
useEffect(() => {
  const interval = setInterval(() => {
    if (processedMessageIdsRef.current.size > MAX_PROCESSED_IDS) {
      // Keep only recent 500 IDs
      const ids = Array.from(processedMessageIdsRef.current);
      processedMessageIdsRef.current = new Set(ids.slice(-500));
    }
  }, 60000); // Cleanup every minute
  
  return () => clearInterval(interval);
}, []);

// In message handler:
const msgId = String(msg.message_id || "");
if (msgId && processedMessageIdsRef.current.has(msgId)) {
  return;
}
if (msgId) {
  processedMessageIdsRef.current.add(msgId);
  // Cleanup if too large
  if (processedMessageIdsRef.current.size > MAX_PROCESSED_IDS) {
    const ids = Array.from(processedMessageIdsRef.current);
    processedMessageIdsRef.current = new Set(ids.slice(-500));
  }
}
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Socket Reconnection Race Condition - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 770-781

**Vấn đề:**
```typescript
const unsubscribeConnection = chatSocketClient.onConnectionChange(
  (isConnected) => {
    if (isConnected && activeConversationIdRef.current) {
      console.log("Reconnected to chat socket, re-joining room:", activeConversationIdRef.current);
      const roomIdNum = parseInt(activeConversationIdRef.current, 10);
      if (!isNaN(roomIdNum)) joinChatRoom(roomIdNum);
    }
  }
);
```

**Bug:**
- ❌ Có thể join room nhiều lần nếu reconnection xảy ra nhiều lần
- ❌ Không check nếu đã join rồi

**Fix:**
```typescript
const joinedRoomsRef = useRef<Set<number>>(new Set());

const unsubscribeConnection = chatSocketClient.onConnectionChange(
  (isConnected) => {
    if (isConnected && activeConversationIdRef.current) {
      const roomIdNum = parseInt(activeConversationIdRef.current, 10);
      if (!isNaN(roomIdNum) && !joinedRoomsRef.current.has(roomIdNum)) {
        console.log("Reconnected to chat socket, re-joining room:", roomIdNum);
        joinChatRoom(roomIdNum);
        joinedRoomsRef.current.add(roomIdNum);
      }
    }
  }
);
```

---

#### 2.2. Double Message Send - `social/page.tsx`
**File:** `app/social/page.tsx`  
**Dòng:** 72-91

**Vấn đề:**
```typescript
const handleSendMessage = async () => {
  const content = message.trim();
  if (!content) return;

  // Optimistic Clear to prevent duplicate sends/Enter spam
  setMessage("");

  try {
    await sendMessage(content);
  } catch (e) {
    console.error("Failed to send", e);
  }
};
```

**Bug:**
- ❌ Không có debounce → user có thể click nhanh → gửi nhiều lần
- ❌ Optimistic clear có thể mất message nếu send fail

**Fix:**
```typescript
const [isSending, setIsSending] = useState(false);

const handleSendMessage = async () => {
  const content = message.trim();
  if (!content || isSending) return;

  setIsSending(true);
  const messageToSend = content; // Save before clearing
  
  // Optimistic Clear
  setMessage("");

  try {
    await sendMessage(messageToSend);
  } catch (e) {
    console.error("Failed to send", e);
    // Restore message on error
    setMessage(messageToSend);
  } finally {
    setIsSending(false);
  }
};
```

---

### 3. **MEMORY LEAK**

#### 3.1. Socket Listeners Not Cleaned Up Properly - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 1028-1041

**Vấn đề:**
```typescript
return () => {
  friendSocketClient.disconnect();
  chatSocketClient.disconnect();
  unsubscribeReceived();
  unsubscribeAccepted();
  // ...
};
```

**Bug:**
- ✅ Đã có cleanup - Good!
- ⚠️ Nhưng nếu component re-render nhiều lần → có thể có multiple listeners

**Fix:**
```typescript
// Store unsubscribe functions in refs
const unsubscribeRefs = useRef<Array<() => void>>([]);

useEffect(() => {
  // ... setup listeners
  
  const unsubscribes = [
    unsubscribeReceived,
    unsubscribeAccepted,
    // ... all unsubscribes
  ];
  
  unsubscribeRefs.current = unsubscribes;
  
  return () => {
    // Cleanup all
    unsubscribeRefs.current.forEach(unsub => unsub());
    unsubscribeRefs.current = [];
    friendSocketClient.disconnect();
    chatSocketClient.disconnect();
  };
}, [/* stable deps */]);
```

---

#### 3.2. Large Context Value - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 1044-1107

**Vấn đề:**
```typescript
const value = React.useMemo(() => ({
  // ... 30+ properties
}), [
  // ... 20+ dependencies
]);
```

**Bug:**
- ❌ Context value quá lớn → re-render tất cả consumers mỗi khi thay đổi
- ❌ Nhiều dependencies → useMemo không hiệu quả

**Fix:**
```typescript
// Split into multiple contexts
const SocialDataContext = createContext<SocialDataContextType>(/* ... */);
const SocialActionsContext = createContext<SocialActionsContextType>(/* ... */);

// Or use separate providers
<SocialDataProvider>
  <SocialActionsProvider>
    {children}
  </SocialActionsProvider>
</SocialDataProvider>
```

---

### 4. **SECURITY BUGS**

#### 4.1. localStorage XSS Risk - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 146-164

**Vấn đề:**
```typescript
useEffect(() => {
  try {
    const userStr = localStorage.getItem("user");
    if (userStr) {
      const user = JSON.parse(userStr);
      setCurrentUser({
        id: user.user_id || user.id,
        username: user.username,
        // ...
      });
    }
  } catch (e) {
    console.error("Error parsing user from local storage", e);
  }
}, []);
```

**Bug:**
- ❌ JSON.parse có thể throw nếu data corrupted
- ❌ Không validate user data structure
- ❌ Có thể inject malicious data

**Fix:**
```typescript
interface UserData {
  user_id?: number | string;
  id?: number | string;
  username?: string;
  fullname?: string;
  email?: string;
  phone?: string;
  avatar?: string;
  role?: { role_name?: string };
}

function isValidUserData(data: any): data is UserData {
  return (
    data &&
    typeof data === 'object' &&
    (typeof data.user_id === 'number' || typeof data.user_id === 'string' ||
     typeof data.id === 'number' || typeof data.id === 'string')
  );
}

useEffect(() => {
  try {
    const userStr = localStorage.getItem("user");
    if (!userStr) return;
    
    const user = JSON.parse(userStr);
    
    if (!isValidUserData(user)) {
      console.error("Invalid user data structure");
      localStorage.removeItem("user"); // Clean corrupted data
      return;
    }
    
    setCurrentUser({
      id: user.user_id || user.id,
      username: user.username || '',
      fullname: user.fullname,
      email: user.email,
      phone: user.phone,
      avatar: user.avatar,
      role_name: user.role?.role_name,
    });
  } catch (e) {
    console.error("Error parsing user from local storage", e);
    localStorage.removeItem("user"); // Clean corrupted data
  }
}, []);
```

---

## 🟡 WARNING ISSUES - app/social

### 5. **CODE QUALITY**

#### 5.1. Complex State Management - `social/SocialContext.tsx`
**Vấn đề:** Context quá phức tạp (1100+ dòng) → khó maintain

**Fix:** Split thành smaller contexts hoặc use state management library (Zustand, Redux)

---

#### 5.2. Type Safety - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 709, 786

**Vấn đề:**
```typescript
status: payload.friend.status as any,
requester: payload.friend.requester as any,
```

**Fix:**
```typescript
// Define proper types
interface FriendRequestPayload {
  friend: {
    id: number;
    requester_id: number;
    addressee_id: number;
    status: 'pending' | 'accepted' | 'rejected';
    created_at: string;
    accepted_at?: string | null;
    requester: User;
    addressee: User;
  };
}

const newRequest: FriendRequestResponse = {
  id: payload.friend.id,
  requester_id: payload.friend.requester_id,
  addressee_id: payload.friend.addressee_id,
  status: payload.friend.status, // No 'as any'
  // ...
};
```

---

#### 5.3. Magic Numbers - `social/SocialContext.tsx`
**File:** `app/social/SocialContext.tsx`  
**Dòng:** 229, 298, 402

**Vấn đề:**
```typescript
limit: 50,
limit: 100,
limit: 50,
```

**Fix:**
```typescript
const FRIEND_REQUESTS_LIMIT = 50;
const CONVERSATIONS_LIMIT = 100;
const MESSAGES_LIMIT = 50;
```

---

## 📈 PERFORMANCE METRICS - app/context, app/hooks, app/social

### Hook Analysis

| Hook | Complexity | Issues |
|------|-----------|--------|
| `useUserId` | Low | Event listener cleanup |
| `useAntiCheat` | High | Memory leak, XSS risk |
| `useExamSocket` | Medium | Socket cleanup |
| `useFileUpload` | Medium | Race condition |

### Context Analysis

| Context | Size | Issues |
|---------|------|--------|
| `ThemeContext` | 137 lines | Race condition |
| `SocialContext` | 1123 lines | Too large, memory leaks |

### Social Feature Analysis

| Component | Issues |
|-----------|--------|
| `SocialContext.tsx` | Memory leaks, race conditions, XSS risk |
| `page.tsx` | Double send, no debounce |

---

## ✅ PRIORITY FIX LIST - app/context, app/hooks, app/social

### 🔴 Critical (Fix ngay)
1. **XSS risk** trong `useAntiCheat.ts` - Fix innerHTML usage
2. **Memory leaks** trong `SocialContext.tsx` - Fix message ID set cleanup
3. **Race conditions** trong `SocialContext.tsx` - Add AbortController
4. **Socket cleanup** trong `useExamSocket.ts` - Fix event listener cleanup
5. **localStorage XSS** trong `SocialContext.tsx` - Validate user data

### 🟡 High (Fix sớm)
6. **Event listener cleanup** trong `useUserId.ts` - Fix handler reference
7. **Double send** trong `social/page.tsx` - Add debounce
8. **Large context** trong `SocialContext.tsx` - Split contexts

### 🟢 Medium (Cải thiện)
9. **Type safety** - Remove `any` types
10. **Magic numbers** - Extract constants
11. **Code complexity** - Refactor large files

---

## 📝 SUMMARY - app/context, app/hooks, app/social

### Tổng kết
- **Critical bugs:** 15 issues cần fix ngay
- **Security issues:** 3 issues ảnh hưởng bảo mật
- **Memory leaks:** 5 issues
- **Performance issues:** 4 issues ảnh hưởng UX
- **Code quality:** 15 issues cần cải thiện

### Điểm mạnh
- ✅ Có cleanup trong một số hooks
- ✅ Có error handling
- ✅ Code structure tương đối tốt
- ✅ Có socket reconnection logic

### Điểm yếu
- ❌ XSS vulnerabilities
- ❌ Memory leaks trong contexts và hooks
- ❌ Race conditions trong async operations
- ❌ Socket cleanup không đầy đủ
- ❌ Context quá lớn và phức tạp
- ❌ Type safety issues

---

## 🔧 RECOMMENDED ACTIONS - app/context, app/hooks, app/social

1. **Immediate:**
   - Fix XSS vulnerabilities
   - Fix memory leaks
   - Fix race conditions
   - Improve socket cleanup

2. **Short-term:**
   - Split large contexts
   - Improve type safety
   - Add proper validation
   - Optimize re-renders

3. **Long-term:**
   - Consider state management library
   - Add comprehensive tests
   - Implement proper monitoring
   - Refactor complex components

---

## 📁 PHÂN TÍCH: app/super-admin

### Tổng quan
- **Tổng số files:** 16 files
- **Files có vấn đề:** 5 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 4 issues
  - 🟡 **Warning:** 4 issues
  - 🟢 **Info:** 2 issues

---

## 🔴 CRITICAL ISSUES - app/super-admin

### 1. **STATE & DATA FLOW BUGS**

#### 1.1. Race Condition Risk - `super-admin/page.tsx`
**File:** `app/super-admin/page.tsx`  
**Dòng:** 159-173

**Vấn đề:**
```typescript
useEffect(() => {
  const fetchStats = async () => {
    try {
      setLoading(true);
      const data = await getStats();
      setStats(data);
    } catch (error: any) {
      message.error(error?.message || "Không thể tải thống kê");
    } finally {
      setLoading(false);
    }
  };

  fetchStats();
}, [message]);
```

**Bug:**
- ❌ Component có thể unmount trước khi request hoàn thành → setState trên unmounted component
- ❌ Dependency `message` có thể thay đổi → re-fetch không cần thiết

**Fix:**
```typescript
useEffect(() => {
  let isMounted = true;

  const fetchStats = async () => {
    try {
      setLoading(true);
      const data = await getStats();
      if (isMounted) {
        setStats(data);
      }
    } catch (error: any) {
      if (isMounted) {
        message.error(error?.message || "Không thể tải thống kê");
      }
    } finally {
      if (isMounted) {
        setLoading(false);
      }
    }
  };

  fetchStats();

  return () => {
    isMounted = false;
  };
}, []); // Remove message dependency
```

---

#### 1.2. Unnecessary Re-renders - `super-admin/page.tsx`
**File:** `app/super-admin/page.tsx`  
**Dòng:** 141-145

**Vấn đề:**
```typescript
{stats.map((stat, index) => (
  <StatCard key={index} {...stat} />
))}
```

**Bug:**
- ❌ Dùng `index` làm key → re-render không cần thiết khi array thay đổi

**Fix:**
```typescript
{stats.map((stat) => (
  <StatCard key={stat.label} {...stat} />
))}
```

---

### 2. **SECURITY BUGS**

#### 2.1. Cookie Decryption Error Handling - `super-admin/layout.tsx`
**File:** `app/super-admin/layout.tsx`  
**Dòng:** 5-30

**Vấn đề:**
```typescript
async function getInitialUserData() {
  try {
    const cookieStore = await cookies();
    const userCookie = cookieStore.get("_u");

    if (userCookie?.value) {
      try {
        const decryptedUser = decryptCookie(userCookie.value);
        const userData = JSON.parse(decryptedUser);
        return {
          username: userData.username || null,
          role_name: userData.role_name || userData.role?.role_name || null,
          avatar: userData.avatar || null,
        };
      } catch (error) {
        console.error("Error decrypting/parsing user cookie:", error);
      }
    }
  } catch (error) {
    console.error("Error reading server cookie:", error);
  }

  return null;
}
```

**Bug:**
- ❌ Silent failure → không log đầy đủ
- ❌ Không validate decrypted data
- ❌ Có thể return partial data nếu JSON.parse fail một phần

**Fix:**
```typescript
async function getInitialUserData() {
  try {
    const cookieStore = await cookies();
    const userCookie = cookieStore.get("_u");

    if (!userCookie?.value) {
      return null;
    }

    try {
      const decryptedUser = decryptCookie(userCookie.value);
      const userData = JSON.parse(decryptedUser);
      
      // Validate required fields
      if (!userData || typeof userData !== 'object') {
        console.error("Invalid user data structure");
        return null;
      }

      return {
        username: userData.username || null,
        role_name: userData.role_name || userData.role?.role_name || null,
        avatar: userData.avatar || null,
      };
    } catch (decryptError) {
      console.error("Error decrypting/parsing user cookie:", decryptError);
      // In production, consider logging to monitoring service
      return null;
    }
  } catch (error) {
    console.error("Error reading server cookie:", error);
    return null;
  }
}
```

---

#### 2.2. Missing Error Handling - `super-admin/SuperAdminLayoutClient.tsx`
**File:** `app/super-admin/SuperAdminLayoutClient.tsx`  
**Dòng:** 53-72

**Vấn đề:**
```typescript
const fetchUserInfo = useCallback(async (showError = false) => {
  const userId = getUserIdFromCookie();
  if (!userId) {
    if (showError) message.error("Không tìm thấy thông tin người dùng");
    return;
  }

  if (showError) setLoadingProfile(true);
  try {
    const user = await getUserInfo(userId);
    setUserInfo(user);
  } catch (error: any) {
    if (showError) {
      message.error(error?.message || "Không thể tải thông tin người dùng");
    }
    console.error("Error fetching user info:", error);
  } finally {
    if (showError) setLoadingProfile(false);
  }
}, []);
```

**Bug:**
- ❌ Component có thể unmount trước khi request hoàn thành
- ❌ Không có cleanup
- ❌ Dependency array rỗng nhưng dùng `message` → stale closure

**Fix:**
```typescript
const fetchUserInfo = useCallback(async (showError = false) => {
  let isMounted = true;
  
  const userId = getUserIdFromCookie();
  if (!userId) {
    if (showError) message.error("Không tìm thấy thông tin người dùng");
    return;
  }

  if (showError) setLoadingProfile(true);
  
  try {
    const user = await getUserInfo(userId);
    if (isMounted) {
      setUserInfo(user);
    }
  } catch (error: unknown) {
    if (isMounted) {
      const errorMessage = error instanceof Error 
        ? error.message 
        : "Không thể tải thông tin người dùng";
      if (showError) {
        message.error(errorMessage);
      }
      console.error("Error fetching user info:", error);
    }
  } finally {
    if (isMounted && showError) {
      setLoadingProfile(false);
    }
  }
  
  return () => {
    isMounted = false;
  };
}, []); // Keep empty, use message from App.useApp() context
```

---

## 🟡 WARNING ISSUES - app/super-admin

### 3. **CODE QUALITY**

#### 3.1. Magic Numbers - `super-admin/page.tsx`
**File:** `app/super-admin/page.tsx`  
**Dòng:** 90-96

**Vấn đề:**
```typescript
const getGreeting = () => {
  const hour = new Date().getHours();
  if (hour < 12) return "Chào buổi sáng";
  if (hour < 18) return "Chào buổi chiều";
  return "Chào buổi tối";
};
```

**Fix:**
```typescript
const MORNING_HOUR = 12;
const EVENING_HOUR = 18;

const getGreeting = () => {
  const hour = new Date().getHours();
  if (hour < MORNING_HOUR) return "Chào buổi sáng";
  if (hour < EVENING_HOUR) return "Chào buổi chiều";
  return "Chào buổi tối";
};
```

---

#### 3.2. Type Safety - `super-admin/page.tsx`
**File:** `app/super-admin/page.tsx`  
**Dòng:** 138

**Vấn đề:**
```typescript
function StatisticsCards({ stats }: { stats: any[] }) {
```

**Fix:**
```typescript
interface StatCard {
  label: string;
  value: number;
  icon: React.ComponentType;
  color: string;
  bgColor: string;
  darkBgColor?: string;
}

function StatisticsCards({ stats }: { stats: StatCard[] }) {
  // ...
}
```

---

## 📁 PHÂN TÍCH: app/user

### Tổng quan
- **Tổng số files:** 13 files
- **Files có vấn đề:** 6 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 5 issues
  - 🟡 **Warning:** 5 issues
  - 🟢 **Info:** 2 issues

---

## 🔴 CRITICAL ISSUES - app/user

### 1. **STATE & DATA FLOW BUGS**

#### 1.1. Race Condition Risk - `user/page.tsx`
**File:** `app/user/page.tsx`  
**Dòng:** 111-125

**Vấn đề:**
```typescript
useEffect(() => {
  const fetchStats = async () => {
    try {
      setLoading(true);
      const data = await getStats();
      setStats(data);
    } catch (error: any) {
      message.error(error?.message || "Không thể tải thống kê");
    } finally {
      setLoading(false);
    }
  };

  fetchStats();
}, [message]);
```

**Bug:** Tương tự như `super-admin/page.tsx` - component có thể unmount trước khi request hoàn thành

**Fix:** Tương tự như fix ở trên - thêm `isMounted` check và remove `message` dependency

---

#### 1.2. Unnecessary Re-renders - `user/page.tsx`
**File:** `app/user/page.tsx`  
**Dòng:** 58-99

**Vấn đề:**
```typescript
{items.map((item, index) => {
  return (
    <Card key={index} ...>
```

**Bug:**
- ❌ Dùng `index` làm key → re-render không cần thiết

**Fix:**
```typescript
{items.map((item) => (
  <Card key={item.path} ...>
))}
```

---

### 2. **ASYNC / TIMING BUGS**

#### 2.1. Missing Error Handling - `user/UserLayoutClient.tsx`
**File:** `app/user/UserLayoutClient.tsx`  
**Dòng:** 43-62

**Vấn đề:**
```typescript
const fetchUserInfo = useCallback(async (showError = false) => {
  const userId = getUserIdFromCookie();
  if (!userId) {
    if (showError) message.error("Không tìm thấy thông tin người dùng");
    return;
  }

  if (showError) setLoadingProfile(true);
  try {
    const user = await getUserInfo(userId);
    setUserInfo(user);
  } catch (error: any) {
    if (showError) {
      message.error(error?.message || "Không thể tải thông tin người dùng");
    }
    console.error("Error fetching user info:", error);
  } finally {
    if (showError) setLoadingProfile(false);
  }
}, []);
```

**Bug:** Tương tự như `SuperAdminLayoutClient.tsx` - không có cleanup

**Fix:** Tương tự như fix ở trên

---

#### 2.2. Date Formatting Mismatch - `user/UserLayoutClient.tsx`
**File:** `app/user/UserLayoutClient.tsx`  
**Dòng:** 155

**Vấn đề:**
```typescript
{userInfo.created_at ? new Date(userInfo.created_at).toLocaleDateString("vi-VN") : "Chưa có thông tin"}
```

**Bug:** Server và client có thể format khác nhau (timezone, locale)

**Fix:**
```typescript
// Use a consistent date formatting library
import { format } from 'date-fns';
import { vi } from 'date-fns/locale';

{userInfo.created_at 
  ? format(new Date(userInfo.created_at), 'dd/MM/yyyy', { locale: vi })
  : "Chưa có thông tin"}
```

---

### 3. **SECURITY BUGS**

#### 3.1. Cookie Decryption Error Handling - `user/layout.tsx`
**File:** `app/user/layout.tsx`  
**Dòng:** 5-30

**Bug:** Tương tự như `super-admin/layout.tsx` - silent failure, không validate data

**Fix:** Tương tự như fix ở trên

---

## 🟡 WARNING ISSUES - app/user

### 4. **CODE QUALITY**

#### 4.1. Type Safety - `user/page.tsx`
**File:** `app/user/page.tsx`  
**Dòng:** 52

**Vấn đề:**
```typescript
function QuickActionsGrid({ items }: { items: any[] }) {
```

**Fix:**
```typescript
interface DashboardItem {
  icon: React.ComponentType;
  title: string;
  description: string;
  gradient: string;
  iconBg: string;
  iconColor: string;
  path: string;
  isComingSoon?: boolean;
}

function QuickActionsGrid({ items }: { items: DashboardItem[] }) {
  // ...
}
```

---

## 📁 PHÂN TÍCH: app (Global Files)

### Tổng quan
- **Tổng số files:** 3 files (layout.tsx, not-found.tsx, providers.tsx)
- **Files có vấn đề:** 3 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 3 issues
  - 🟡 **Warning:** 4 issues
  - 🟢 **Info:** 2 issues

---

## 🔴 CRITICAL ISSUES - app (Global Files)

### 1. **SECURITY BUGS**

#### 1.1. XSS Risk in Inline Script - `app/layout.tsx`
**File:** `app/layout.tsx`  
**Dòng:** 62-91

**Vấn đề:**
```typescript
<script
  dangerouslySetInnerHTML={{
    __html: `
      (function() {
        try {
          var html = document.documentElement;
          html.classList.add('no-transitions');
          // ...
        } catch (e) {}
      })();
    `,
  }}
/>
```

**Bug:**
- ❌ Dùng `dangerouslySetInnerHTML` → XSS risk nếu script bị inject
- ❌ Không validate script content

**Fix:**
```typescript
// Option 1: Use Next.js Script component
import Script from 'next/script';

<Script id="no-transitions" strategy="beforeInteractive">
  {`
    (function() {
      try {
        var html = document.documentElement;
        html.classList.add('no-transitions');
        // ...
      } catch (e) {}
    })();
  `}
</Script>

// Option 2: Move to external file and load
// Option 3: Use useEffect in client component
```

---

#### 1.2. External Resource Loading - `app/layout.tsx`
**File:** `app/layout.tsx`  
**Dòng:** 50-61

**Vấn đề:**
```typescript
<link
  rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css"
  integrity="sha512-DTOQO9RWCH3ppGqcWaEA1BIZOC6xxalwEsw9c2QQeAIftl+Vegovlnee1c9QX4TctnWMn13TZye+giMm8e2LwA=="
  crossOrigin="anonymous"
  referrerPolicy="no-referrer"
/>
```

**Bug:**
- ❌ Load external resource → có thể bị compromised
- ❌ Integrity hash có thể outdated
- ❌ Không có fallback nếu CDN fail

**Fix:**
```typescript
// Option 1: Self-host Font Awesome
// Option 2: Use Next.js font optimization
// Option 3: Add error handling
<link
  rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css"
  integrity="sha512-DTOQO9RWCH3ppGqcWaEA1BIZOC6xxalwEsw9c2QQeAIftl+Vegovlnee1c9QX4TctnWMn13TZye+giMm8e2LwA=="
  crossOrigin="anonymous"
  referrerPolicy="no-referrer"
  onError={(e) => {
    console.error("Failed to load Font Awesome");
    // Fallback to local version or remove
  }}
/>
```

---

### 2. **PERFORMANCE BUGS**

#### 2.1. Prefetch Routes Logic - `components/common/PrefetchRoutes.tsx`
**File:** `app/components/common/PrefetchRoutes.tsx`  
**Dòng:** 14-36

**Vấn đề:**
```typescript
const prefetchRoutes = useCallback(() => {
  if (pathname?.startsWith("/admin")) {
    router.prefetch("/admin");
    router.prefetch("/admin/classes");
    // ... many prefetches
  } else if (pathname?.startsWith("/user")) {
    router.prefetch("/user");
    // ... many prefetches
  } else {
    // Prefetch cả admin và user
    router.prefetch("/admin");
    router.prefetch("/admin/classes");
    router.prefetch("/user");
    router.prefetch("/user/classes");
  }
}, [pathname, router]);
```

**Bug:**
- ❌ Prefetch quá nhiều routes → waste bandwidth
- ❌ Prefetch cả admin và user khi ở root → không cần thiết
- ❌ Không có debounce → có thể prefetch nhiều lần

**Fix:**
```typescript
const prefetchRoutes = useCallback(() => {
  // Only prefetch likely next routes
  if (pathname?.startsWith("/admin")) {
    // Only prefetch immediate children
    router.prefetch("/admin/classes");
    router.prefetch("/admin/students");
  } else if (pathname?.startsWith("/user")) {
    router.prefetch("/user/classes");
    router.prefetch("/user/documents");
  } else if (pathname === "/") {
    // Only prefetch if user is likely to navigate
    // Don't prefetch both admin and user
  }
}, [pathname, router]);

// Add debounce
useEffect(() => {
  const timer = setTimeout(() => {
    prefetchRoutes();
  }, 1000); // Wait 1 second after navigation
  
  return () => clearTimeout(timer);
}, [prefetchRoutes]);
```

---

#### 2.2. requestIdleCallback Fallback - `components/common/PrefetchRoutes.tsx`
**File:** `app/components/common/PrefetchRoutes.tsx`  
**Dòng:** 40-53

**Vấn đề:**
```typescript
if (typeof window === "undefined" || !("requestIdleCallback" in window)) {
  setTimeout(() => prefetchRoutes(), 2000);
  return;
}

const prefetchOnIdle = () => {
  requestIdleCallback(() => {
    prefetchRoutes();
  }, { timeout: 2000 });
};
```

**Bug:**
- ❌ Fallback timeout quá ngắn → có thể block main thread
- ❌ Không cleanup timeout nếu component unmount

**Fix:**
```typescript
useEffect(() => {
  let timeoutId: NodeJS.Timeout | null = null;
  let idleCallbackId: number | null = null;

  const prefetchOnIdle = () => {
    if (typeof window === "undefined") return;

    if ("requestIdleCallback" in window) {
      idleCallbackId = requestIdleCallback(
        () => {
          prefetchRoutes();
        },
        { timeout: 5000 } // Longer timeout
      );
    } else {
      // Fallback with longer delay
      timeoutId = setTimeout(() => {
        prefetchRoutes();
      }, 3000);
    }
  };

  prefetchOnIdle();

  return () => {
    if (timeoutId) {
      clearTimeout(timeoutId);
    }
    if (idleCallbackId && typeof window !== "undefined" && "cancelIdleCallback" in window) {
      cancelIdleCallback(idleCallbackId);
    }
  };
}, [prefetchRoutes]);
```

---

### 3. **HYDRATION / SSR BUGS**

#### 3.1. Theme Hydration - `app/layout.tsx`
**File:** `app/layout.tsx`  
**Dòng:** 32-37

**Vấn đề:**
```typescript
const cookieStore = await cookies();
const theme = cookieStore.get("theme");
const isDark = theme?.value === "dark";

return (
  <html lang="vi" className={isDark ? "dark" : ""} suppressHydrationWarning>
```

**Bug:**
- ❌ Server và client có thể có theme khác nhau → hydration mismatch
- ❌ `suppressHydrationWarning` chỉ suppress warning, không fix root cause

**Fix:**
```typescript
// Use inline script to set theme before React hydration (already done)
// But ensure consistency
const cookieStore = await cookies();
const theme = cookieStore.get("theme");
const isDark = theme?.value === "dark";

return (
  <html 
    lang="vi" 
    className={isDark ? "dark" : ""} 
    suppressHydrationWarning
    data-theme={isDark ? "dark" : "light"} // Add data attribute for consistency
  >
```

---

## 🟡 WARNING ISSUES - app (Global Files)

### 4. **CODE QUALITY**

#### 4.1. Magic Numbers - `app/layout.tsx`
**File:** `app/layout.tsx`  
**Dòng:** 11, 79

**Vấn đề:**
```typescript
weight: ['400', '500', '700'],
setTimeout(function() {
  requestAnimationFrame(function() {
    html.classList.remove('no-transitions');
  });
}, 100); // Magic number
```

**Fix:**
```typescript
const FONT_WEIGHTS = ['400', '500', '700'] as const;
const HYDRATION_DELAY_MS = 100;

weight: FONT_WEIGHTS,
setTimeout(function() {
  requestAnimationFrame(function() {
    html.classList.remove('no-transitions');
  });
}, HYDRATION_DELAY_MS);
```

---

#### 4.2. Type Safety - `app/providers.tsx`
**File:** `app/providers.tsx`  
**Dòng:** 6-43

**Vấn đề:**
- ✅ Code tương đối tốt
- ⚠️ Có thể improve với proper types cho theme

**Fix:**
```typescript
interface AntdConfigProviderProps {
  children: React.ReactNode;
}

function AntdConfigProvider({ children }: AntdConfigProviderProps) {
  // ...
}
```

---

#### 4.3. Not Found Page - `app/not-found.tsx`
**File:** `app/not-found.tsx`  
**Dòng:** 8-102

**Vấn đề:**
- ✅ Code tương đối tốt
- ⚠️ Có thể improve với error boundary integration

**Suggestion:**
```typescript
// Add error boundary wrapper
// Add analytics tracking for 404s
// Add search functionality
```

---

## 📈 PERFORMANCE METRICS - app/super-admin, app/user, app (Global)

### Layout Analysis

| Layout | Size | Issues |
|--------|------|--------|
| `app/layout.tsx` | 107 lines | XSS risk, external resources |
| `super-admin/layout.tsx` | 37 lines | Cookie validation |
| `user/layout.tsx` | 37 lines | Cookie validation |
| `providers.tsx` | 53 lines | ✅ Good |

### Dashboard Analysis

| Dashboard | Issues |
|-----------|--------|
| `super-admin/page.tsx` | Race condition, re-renders |
| `user/page.tsx` | Race condition, re-renders |

### Global Components Analysis

| Component | Issues |
|-----------|--------|
| `not-found.tsx` | ✅ Good |
| `PrefetchRoutes.tsx` | Too many prefetches, no cleanup |

---

## ✅ PRIORITY FIX LIST - app/super-admin, app/user, app (Global)

### 🔴 Critical (Fix ngay)
1. **XSS risk** trong `app/layout.tsx` - Fix inline script
2. **Race conditions** trong dashboards - Add cleanup
3. **Cookie validation** trong layouts - Validate decrypted data
4. **Prefetch optimization** trong `PrefetchRoutes.tsx` - Reduce prefetches

### 🟡 High (Fix sớm)
5. **Re-renders** trong dashboards - Fix keys
6. **Error handling** trong layouts - Add cleanup
7. **Date formatting** - Use consistent library

### 🟢 Medium (Cải thiện)
8. **Type safety** - Remove `any` types
9. **Magic numbers** - Extract constants
10. **External resources** - Self-host or add fallback

---

## 📝 SUMMARY - app/super-admin, app/user, app (Global)

### Tổng kết
- **Critical bugs:** 12 issues cần fix ngay
- **Security issues:** 4 issues ảnh hưởng bảo mật
- **Performance issues:** 2 issues ảnh hưởng UX
- **Code quality:** 9 issues cần cải thiện

### Điểm mạnh
- ✅ Code structure tương đối tốt
- ✅ Có error handling
- ✅ Có loading states
- ✅ Có memoization trong một số components

### Điểm yếu
- ❌ XSS vulnerabilities
- ❌ Race conditions trong async operations
- ❌ Cookie validation không đầy đủ
- ❌ Prefetch quá nhiều routes
- ❌ Type safety issues

---

## 🔧 RECOMMENDED ACTIONS - app/super-admin, app/user, app (Global)

1. **Immediate:**
   - Fix XSS vulnerabilities
   - Fix race conditions
   - Improve cookie validation
   - Optimize prefetch logic

2. **Short-term:**
   - Improve type safety
   - Fix re-render issues
   - Add proper error handling
   - Self-host external resources

3. **Long-term:**
   - Add comprehensive tests
   - Implement proper monitoring
   - Add analytics for 404s
   - Optimize bundle size

---

## 📊 TỔNG KẾT TOÀN BỘ CODE REVIEW

### Thống kê tổng thể
- **Tổng số files đã review:** ~150+ files
- **Tổng số issues:**
  - 🔴 **Critical:** 60+ issues
  - 🟡 **Warning:** 80+ issues
  - 🟢 **Info:** 30+ issues

### Phân loại issues

| Loại Bug | Số lượng | Mức độ |
|----------|---------|--------|
| Security bugs | 25+ | 🔴 Critical |
| Memory leaks | 15+ | 🔴 Critical |
| Race conditions | 20+ | 🔴 Critical |
| Performance bugs | 15+ | 🟡 Warning |
| Code quality | 80+ | 🟡 Warning |

### Top 10 Critical Issues cần fix ngay

1. **SSRF vulnerability** trong `api-proxy/[...path]/route.ts`
2. **XSS risks** trong `RichTextEditor.tsx`, `useAntiCheat.ts`, `app/layout.tsx`
3. **Memory leaks** trong `SocialContext.tsx`, `config/api.ts`, `ScrollAnimation.tsx`
4. **Race conditions** trong `events/page.tsx`, `news/page.tsx`, `SocialContext.tsx`
5. **No rate limiting** trong `api-proxy` routes và `auth/page.tsx`
6. **Cookie security** trong `api-proxy` và layouts
7. **Input validation** missing trong nhiều components
8. **Socket cleanup** không đầy đủ trong hooks
9. **Token refresh race** trong `config/api.ts`
10. **Hydration mismatches** trong `news/[id]/page.tsx`, `app/layout.tsx`

### Điểm mạnh tổng thể
- ✅ Code structure tương đối tốt
- ✅ Có sử dụng React best practices (memoization, hooks)
- ✅ Có error handling trong nhiều places
- ✅ Có loading states
- ✅ Có transaction queue để prevent race conditions

### Điểm yếu tổng thể
- ❌ Nhiều security vulnerabilities (XSS, SSRF, input validation)
- ❌ Nhiều memory leaks (observers, caches, sockets)
- ❌ Nhiều race conditions (async operations, state updates)
- ❌ Type safety issues (nhiều `any` types)
- ❌ Code duplication
- ❌ Large components và contexts
- ❌ Inconsistent error handling

### Recommended Next Steps

1. **Week 1-2: Critical Security Fixes**
   - Fix SSRF vulnerability
   - Fix XSS risks
   - Add input validation
   - Improve cookie security

2. **Week 3-4: Memory Leaks & Race Conditions**
   - Fix memory leaks
   - Fix race conditions
   - Improve cleanup logic
   - Add AbortController where needed

3. **Week 5-6: Performance & Code Quality**
   - Optimize re-renders
   - Split large components
   - Improve type safety
   - Reduce code duplication

4. **Week 7-8: Testing & Monitoring**
   - Add comprehensive tests
   - Implement monitoring
   - Add error tracking
   - Performance monitoring

---

## 📁 PHÂN TÍCH: interface

### Tổng quan
- **Tổng số files:** 6 files
- **Files có vấn đề:** 3 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 2 issues
  - 🟡 **Warning:** 3 issues
  - 🟢 **Info:** 1 issue

---

## 🔴 CRITICAL ISSUES - interface

### 1. **TYPE SAFETY BUGS**

#### 1.1. Inconsistent Type Definitions - `interface/auth.ts`
**File:** `interface/auth.ts`  
**Dòng:** 8-20

**Vấn đề:**
```typescript
export interface User {
  user_id: string;  // ❌ Should be number | string
  username: string;
  fullname: string;
  email: string;
  phone: string;
  avatar: string;
  created_at: string;
  updated_at: string;
  role: Role;
  access_token: string;
  refresh_token: string;
}

export interface SignUpUser {
  user_id: number;  // ❌ Inconsistent with User interface
  // ...
}
```

**Bug:**
- ❌ `User.user_id` là `string` nhưng `SignUpUser.user_id` là `number`
- ❌ Không consistent với API response (thường là `number`)
- ❌ Có thể gây type errors khi convert

**Fix:**
```typescript
export interface User {
  user_id: number | string;  // ✅ Support both
  username: string;
  fullname: string;
  email: string;
  phone: string;
  avatar: string;
  created_at: string;
  updated_at: string;
  role: Role;
  access_token?: string;  // ✅ Optional (có thể không có trong response)
  refresh_token?: string;  // ✅ Optional
}

export interface SignUpUser {
  user_id: number | string;  // ✅ Consistent
  // ...
}
```

---

#### 1.2. Missing Optional Fields - `interface/students.ts`
**File:** `interface/students.ts`  
**Dòng:** 1-12

**Vấn đề:**
```typescript
export interface StudentItem {
  key: string;
  userId: number | string;
  name: string;
  studentId: string;
  class: string;
  email: string;
  phone: string;  // ❌ Should be optional
  status: "Đang học" | "Tạm nghỉ" | "Đã tốt nghiệp" | "Bị cấm";
  apiStatus?: string;
  classStudentId?: number | string;
}
```

**Bug:**
- ❌ `phone` không phải optional nhưng API có thể không trả về
- ❌ `email` có thể null từ API

**Fix:**
```typescript
export interface StudentItem {
  key: string;
  userId: number | string;
  name: string;
  studentId: string;
  class: string;
  email: string | null;  // ✅ Can be null
  phone: string | null;  // ✅ Can be null
  status: "Đang học" | "Tạm nghỉ" | "Đã tốt nghiệp" | "Bị cấm";
  apiStatus?: string;
  classStudentId?: number | string;
}
```

---

## 🟡 WARNING ISSUES - interface

### 2. **CODE QUALITY**

#### 2.1. Missing Type Exports - `interface/chat.ts`
**File:** `interface/chat.ts`  
**Dòng:** 1-44

**Vấn đề:**
- ✅ Code tương đối tốt
- ⚠️ Có thể thêm utility types

**Suggestion:**
```typescript
// Add utility types
export type ChatMessageType = "text" | "image" | "file";
export type ChatRole = "admin" | "teacher" | "student";
export type ChatStatus = "Hoạt động" | "Tạm dừng";

// Update interfaces to use these types
export interface ChatMessage {
  // ...
  type?: ChatMessageType;  // ✅ Use type alias
}

export interface ChatParticipant {
  // ...
  role: ChatRole;  // ✅ Use type alias
}
```

---

#### 2.2. Missing Validation Types - `interface/common.ts`
**File:** `interface/common.ts`  
**Dòng:** 1-11

**Vấn đề:**
- ✅ Code đơn giản và tốt
- ⚠️ Có thể thêm validation helpers

**Suggestion:**
```typescript
export interface Comment {
  id: string | number;
  author: string;
  avatar: string;
  time: string;
  content: string;
  likes: number;
  replies?: Comment[];
}

// Add validation helper
export const isValidComment = (comment: Partial<Comment>): comment is Comment => {
  return !!(
    comment.id &&
    comment.author &&
    comment.content &&
    typeof comment.likes === 'number'
  );
};
```

---

## 📁 PHÂN TÍCH: lib

### Tổng quan
- **Tổng số files:** 36 files
- **Files có vấn đề:** 12 files
- **Mức độ nghiêm trọng:**
  - 🔴 **Critical:** 8 issues
  - 🟡 **Warning:** 10 issues
  - 🟢 **Info:** 5 issues

---

## 🔴 CRITICAL ISSUES - lib

### 1. **SECURITY BUGS**

#### 1.1. Weak Encryption Key - `lib/utils/server-cookie-decrypt.ts`
**File:** `lib/utils/server-cookie-decrypt.ts`  
**Dòng:** 9

**Vấn đề:**
```typescript
const ENCRYPTION_KEY = process.env.COOKIE_ENCRYPTION_KEY || 'default-32-char-key-for-dev-only!!';
```

**Bug:**
- ❌ Default key trong code → security risk
- ❌ Key có thể không đủ mạnh
- ❌ Không validate key length

**Fix:**
```typescript
const ENCRYPTION_KEY = process.env.COOKIE_ENCRYPTION_KEY;

if (!ENCRYPTION_KEY) {
  throw new Error('COOKIE_ENCRYPTION_KEY environment variable is required');
}

if (ENCRYPTION_KEY.length < 32) {
  throw new Error('COOKIE_ENCRYPTION_KEY must be at least 32 characters');
}

// Use crypto to derive key from ENCRYPTION_KEY
const key = crypto.scryptSync(ENCRYPTION_KEY, 'salt', 32);
```

---

#### 1.2. Token Storage in localStorage - `lib/socket/client.ts`
**File:** `lib/socket/client.ts`  
**Dòng:** 37-56

**Vấn đề:**
```typescript
private getAccessToken(): string | null {
  if (typeof window === "undefined") return null;
  
  // Try localStorage first
  const token = localStorage.getItem("accessToken");
  if (token) return token;

  // Try to get from user cookie
  try {
    const userStr = localStorage.getItem("user");
    if (userStr) {
      const user = JSON.parse(userStr);
      return user.access_token || null;
    }
  } catch (error) {
    console.error("Error parsing user from localStorage:", error);
  }

  return null;
}
```

**Bug:**
- ❌ Lưu token trong localStorage → XSS risk
- ❌ Token có thể bị đọc bởi malicious scripts
- ❌ Không có encryption

**Fix:**
```typescript
private getAccessToken(): string | null {
  if (typeof window === "undefined") return null;
  
  // ✅ Try cookie first (more secure)
  try {
    const cookies = document.cookie.split(';');
    for (const cookie of cookies) {
      const [name, value] = cookie.trim().split('=');
      if (name === '_at' && value) {
        // Decrypt if needed
        return decodeURIComponent(value);
      }
    }
  } catch (error) {
    console.error("Error getting token from cookie:", error);
  }

  // ❌ Remove localStorage fallback (security risk)
  // Only use if absolutely necessary and document the risk
  
  return null;
}
```

---

### 2. **MEMORY LEAKS**

#### 2.1. Cookie Cache Never Cleared - `lib/utils/cookies.ts`
**File:** `lib/utils/cookies.ts`  
**Dòng:** 6-13

**Vấn đề:**
```typescript
let cachedCookieString: string | null = null;
let cachedCookieTimestamp: number = 0;
const COOKIE_CACHE_DURATION = 100; // 100ms cache

let parsedCookiesCache: Map<string, string | null> = new Map();
let parsedCookiesTimestamp: number = 0;
const PARSED_COOKIES_CACHE_DURATION = 50; // 50ms cache
```

**Bug:**
- ❌ `parsedCookiesCache` Map không bao giờ được clear entries cũ
- ❌ Có thể grow indefinitely nếu có nhiều cookie names
- ❌ Không có max size limit

**Fix:**
```typescript
const MAX_CACHE_SIZE = 100; // Max number of cached cookies

// Clear old entries when cache exceeds max size
if (parsedCookiesCache.size > MAX_CACHE_SIZE) {
  const oldestKey = parsedCookiesCache.keys().next().value;
  parsedCookiesCache.delete(oldestKey);
}

// Or use LRU cache
class LRUCache<K, V> {
  private cache: Map<K, V>;
  private maxSize: number;

  constructor(maxSize: number) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key)!;
    // Move to end (most recently used)
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove oldest (first) entry
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  clear(): void {
    this.cache.clear();
  }
}

const parsedCookiesCache = new LRUCache<string, string | null>(MAX_CACHE_SIZE);
```

---

#### 2.2. Promise Cache Never Cleared - `lib/utils/cookies.ts`
**File:** `lib/utils/cookies.ts`  
**Dòng:** 77-79

**Vấn đề:**
```typescript
let isDecrypting = false;
let decryptPromise: Promise<number | string | null> | null = null;
```

**Bug:**
- ❌ `decryptPromise` có thể bị stuck nếu promise reject không được handle
- ❌ Không có timeout cho promise
- ❌ Không clear promise sau một thời gian

**Fix:**
```typescript
let isDecrypting = false;
let decryptPromise: Promise<number | string | null> | null = null;
let decryptPromiseTimestamp = 0;
const DECRYPT_PROMISE_TTL = 5000; // 5 seconds

// Clear stale promise
const clearStalePromise = () => {
  const now = Date.now();
  if (decryptPromise && (now - decryptPromiseTimestamp > DECRYPT_PROMISE_TTL)) {
    decryptPromise = null;
    isDecrypting = false;
  }
};

// Add timeout to promise
const getUserIdFromCookieAsync = async (): Promise<number | string | null> => {
  // ... existing code ...
  
  if (encryptedUserCookie) {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 10000); // 10s timeout
      
      const response = await fetch("/api-proxy/auth/decrypt-user", {
        signal: controller.signal,
      });
      
      clearTimeout(timeoutId);
      // ... rest of code ...
    } catch (error) {
      // Clear promise on error
      decryptPromise = null;
      isDecrypting = false;
      throw error;
    }
  }
};
```

---

### 3. **ASYNC / TIMING BUGS**

#### 3.1. Race Condition in Cookie Decryption - `lib/utils/cookies.ts`
**File:** `lib/utils/cookies.ts`  
**Dòng:** 160-192

**Vấn đề:**
```typescript
export const getUserIdFromCookie = (): number | string | null => {
  // ...
  if (encryptedUserCookie && !isDecrypting && !decryptPromise) {
    isDecrypting = true;
    decryptPromise = getUserIdFromCookieAsync()
      .then((userId) => {
        isDecrypting = false;
        decryptPromise = null;
        // ...
        return userId;
      })
      .catch((error) => {
        isDecrypting = false;
        decryptPromise = null;
        throw error;
      });
  }

  return null;  // ❌ Returns null immediately, doesn't wait for promise
};
```

**Bug:**
- ❌ Function return `null` ngay lập tức, không đợi promise resolve
- ❌ Caller không biết promise đang chạy
- ❌ Có thể gọi nhiều lần trước khi promise resolve

**Fix:**
```typescript
// Option 1: Make it async
export const getUserIdFromCookie = async (): Promise<number | string | null> => {
  // ...
  if (encryptedUserCookie) {
    if (decryptPromise) {
      return decryptPromise; // Return existing promise
    }
    
    isDecrypting = true;
    decryptPromise = getUserIdFromCookieAsync()
      .then((userId) => {
        isDecrypting = false;
        decryptPromise = null;
        return userId;
      })
      .catch((error) => {
        isDecrypting = false;
        decryptPromise = null;
        throw error;
      });
    
    return decryptPromise;
  }

  return null;
};

// Option 2: Use event-based approach
export const getUserIdFromCookie = (): number | string | null => {
  const sessionUserId = getUserIdFromSession();
  if (sessionUserId) {
    return sessionUserId;
  }

  const encryptedUserCookie = getCookie("_u");
  if (encryptedUserCookie && !isDecrypting && !decryptPromise) {
    isDecrypting = true;
    decryptPromise = getUserIdFromCookieAsync()
      .then((userId) => {
        isDecrypting = false;
        decryptPromise = null;
        if (typeof window !== "undefined" && userId) {
          window.dispatchEvent(new CustomEvent("user_id_cached", { detail: userId }));
        }
        return userId;
      })
      .catch((error) => {
        isDecrypting = false;
        decryptPromise = null;
        throw error;
      });
  }

  return null; // Return null, but trigger async decrypt
};
```

---

#### 3.2. Missing Error Handling - `lib/api/auth.ts`
**File:** `lib/api/auth.ts`  
**Dòng:** 39-80

**Vấn đề:**
```typescript
export const signOut = async (): Promise<void> => {
  try {
    await apiClient.post("/auth/signout", {}, {
      withCredentials: true,
    });
  } catch (error: any) {
    // Logout vẫn tiếp tục ngay cả khi API call fail
    // ...
  } finally {
    // ...
    if (typeof window !== "undefined") {
      const theme = localStorage.getItem("theme");
      localStorage.clear();
      if (theme) {
        localStorage.setItem("theme", theme);
      }
      window.location.replace("/auth");
    }
  }
};
```

**Bug:**
- ❌ `window.location.replace` có thể fail nếu không có window
- ❌ Không handle case khi localStorage.clear() fail
- ❌ Force reload có thể interrupt async operations

**Fix:**
```typescript
export const signOut = async (): Promise<void> => {
  try {
    await apiClient.post("/auth/signout", {}, {
      withCredentials: true,
    });
  } catch (error: any) {
    // Logout vẫn tiếp tục ngay cả khi API call fail
    if (process.env.NODE_ENV === 'development') {
      const status = error?.response?.status;
      if (status === 400) {
        console.error("Dữ liệu không hợp lệ khi đăng xuất");
      } else if (status === 401) {
        console.error("Token không hợp lệ khi đăng xuất");
      } else if (error?.message) {
        console.error("Error signing out:", error.message);
      }
    }
  } finally {
    // Always clear tokens and cache
    clearTokens();
    clearUserCache();
    clearCookieCache();
    clearAuthCache();
    clearResponseCache();
    
    // Clear localStorage safely
    if (typeof window !== "undefined") {
      try {
        const theme = localStorage.getItem("theme");
        localStorage.clear();
        if (theme) {
          localStorage.setItem("theme", theme);
        }
      } catch (error) {
        console.error("Error clearing localStorage:", error);
      }
      
      // Use router instead of window.location for better control
      // Or add delay to allow async operations to complete
      setTimeout(() => {
        window.location.replace("/auth");
      }, 100);
    }
  }
};
```

---

### 4. **PERFORMANCE BUGS**

#### 4.1. Inefficient Cookie Parsing - `lib/utils/cookies.ts`
**File:** `lib/utils/cookies.ts`  
**Dòng:** 39-53

**Vấn đề:**
```typescript
const value = `; ${cachedCookieString}`;
const parts = value.split(`; ${name}=`);

let result: string | null = null;
if (parts.length === 2) {
  const cookieValue = parts.pop()?.split(";").shift() || null;
  if (cookieValue) {
    try {
      result = decodeURIComponent(cookieValue);
    } catch {
      result = cookieValue;
    }
  }
}
```

**Bug:**
- ❌ String manipulation không efficient cho nhiều cookies
- ❌ `split` và `pop` có thể slow với large cookie strings
- ❌ Không dùng regex hoặc more efficient parsing

**Fix:**
```typescript
// Use regex for better performance
const getCookie = (name: string): string | null => {
  if (typeof document === "undefined") return null;
  
  const now = Date.now();
  
  // Check parsed cookies cache first
  if (now - parsedCookiesTimestamp < PARSED_COOKIES_CACHE_DURATION) {
    const cached = parsedCookiesCache.get(name);
    if (cached !== undefined) {
      return cached;
    }
  }
  
  // Cache cookie string
  if (!cachedCookieString || now - cachedCookieTimestamp > COOKIE_CACHE_DURATION) {
    cachedCookieString = document.cookie;
    cachedCookieTimestamp = now;
    parsedCookiesCache.clear();
  }
  
  // Use regex for efficient parsing
  const match = cachedCookieString.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
  if (match) {
    try {
      const result = decodeURIComponent(match[1]);
      parsedCookiesCache.set(name, result);
      parsedCookiesTimestamp = now;
      return result;
    } catch {
      const result = match[1];
      parsedCookiesCache.set(name, result);
      parsedCookiesTimestamp = now;
      return result;
    }
  }
  
  parsedCookiesCache.set(name, null);
  parsedCookiesTimestamp = now;
  return null;
};
```

---

## 🟡 WARNING ISSUES - lib

### 5. **CODE QUALITY**

#### 5.1. Code Duplication - `lib/socket/*.ts`
**File:** `lib/socket/client.ts`, `lib/socket/friend-client.ts`, `lib/socket/chat-client.ts`

**Vấn đề:**
- ❌ 3 socket clients có code tương tự nhau
- ❌ Duplicate logic cho `getAccessToken()`, `getUserId()`, `connect()`, etc.

**Fix:**
```typescript
// Create base class
abstract class BaseSocketClient {
  protected socket: SocketInstance | null = null;
  protected isConnecting = false;
  protected connectionListeners: Set<(connected: boolean) => void> = new Set();

  protected abstract getNamespace(): string;
  protected abstract getSocketUrl(): string;
  protected abstract getAccessToken(): string | null;
  protected abstract getUserId(): number | string | null;

  connect(): SocketInstance | null {
    // Shared connection logic
  }

  disconnect(): void {
    // Shared disconnect logic
  }

  // ... other shared methods
}

// Extend base class
class FriendSocketClient extends BaseSocketClient {
  protected getNamespace(): string {
    return '/friends';
  }
  
  // Override specific methods if needed
}
```

---

#### 5.2. Type Safety - `lib/api/users.ts`
**File:** `lib/api/users.ts`  
**Dòng:** 93-120

**Vấn đề:**
```typescript
const extractArrayFromResponse = (data: any): GetUsersResponse[] | null => {
  // ... complex extraction logic with any type
};
```

**Bug:**
- ❌ Dùng `any` type
- ❌ Complex extraction logic khó maintain
- ❌ Không có type guards

**Fix:**
```typescript
interface ApiResponseStructure {
  data?: GetUsersResponse[] | {
    users?: GetUsersResponse[];
    items?: GetUsersResponse[];
    list?: GetUsersResponse[];
    results?: GetUsersResponse[];
    data?: GetUsersResponse[];
  };
  users?: GetUsersResponse[];
  items?: GetUsersResponse[];
  list?: GetUsersResponse[];
  results?: GetUsersResponse[];
}

const extractArrayFromResponse = (data: unknown): GetUsersResponse[] | null => {
  if (!data || typeof data !== 'object') {
    return null;
  }

  const response = data as ApiResponseStructure;

  // Direct array
  if (Array.isArray(response)) {
    return response;
  }

  // Nested data structure
  if (response.data) {
    if (Array.isArray(response.data)) {
      return response.data;
    }
    if (typeof response.data === 'object') {
      const nestedKeys = ['users', 'items', 'list', 'results', 'data'] as const;
      for (const key of nestedKeys) {
        if (Array.isArray(response.data[key])) {
          return response.data[key];
        }
      }
    }
  }

  // Direct keys
  const directKeys = ['users', 'items', 'list', 'results'] as const;
  for (const key of directKeys) {
    if (Array.isArray(response[key])) {
      return response[key];
    }
  }

  return null;
};
```

---

## 📈 PERFORMANCE METRICS - interface & lib

### Interface Analysis

| File | Size | Issues |
|------|------|--------|
| `auth.ts` | 78 lines | Type inconsistency |
| `students.ts` | 14 lines | Missing optional fields |
| `chat.ts` | 44 lines | ✅ Good |
| `classes.ts` | 28 lines | ✅ Good |
| `common.ts` | 11 lines | ✅ Good |
| `exercises.ts` | 21 lines | ✅ Good |

### Lib Analysis

| Module | Files | Issues |
|--------|-------|--------|
| `lib/utils` | 7 files | Cookie cache, encryption |
| `lib/api` | 20 files | Type safety, error handling |
| `lib/socket` | 9 files | Code duplication, token storage |

---

## ✅ PRIORITY FIX LIST - interface & lib

### 🔴 Critical (Fix ngay)
1. **Weak encryption key** trong `server-cookie-decrypt.ts` - Add validation
2. **Token storage** trong `socket/client.ts` - Remove localStorage
3. **Cookie cache leak** trong `cookies.ts` - Add LRU cache
4. **Promise cache** trong `cookies.ts` - Add timeout & cleanup
5. **Race condition** trong `cookies.ts` - Fix async flow

### 🟡 High (Fix sớm)
6. **Type inconsistency** trong `interface/auth.ts` - Standardize types
7. **Code duplication** trong `socket/*.ts` - Create base class
8. **Type safety** trong `api/users.ts` - Remove `any` types
9. **Error handling** trong `api/auth.ts` - Improve cleanup

### 🟢 Medium (Cải thiện)
10. **Cookie parsing** - Use regex for better performance
11. **Missing optional fields** - Add null checks
12. **Utility types** - Add type aliases

---

## 📝 SUMMARY - interface & lib

### Tổng kết
- **Critical bugs:** 8 issues cần fix ngay
- **Security issues:** 2 issues ảnh hưởng bảo mật
- **Memory leaks:** 2 issues ảnh hưởng performance
- **Code quality:** 10 issues cần cải thiện

### Điểm mạnh
- ✅ Interface definitions tương đối tốt
- ✅ Có error handling trong nhiều places
- ✅ Có caching mechanism
- ✅ Socket clients có singleton pattern

### Điểm yếu
- ❌ Security vulnerabilities (encryption key, token storage)
- ❌ Memory leaks (cookie cache, promise cache)
- ❌ Type safety issues (nhiều `any` types)
- ❌ Code duplication (socket clients)
- ❌ Race conditions (async operations)

---

## 🔧 RECOMMENDED ACTIONS - interface & lib

1. **Immediate:**
   - Fix encryption key validation
   - Remove token storage from localStorage
   - Fix cookie cache memory leak
   - Fix promise cache timeout

2. **Short-term:**
   - Standardize types across interfaces
   - Create base class for socket clients
   - Improve type safety (remove `any`)
   - Add proper error handling

3. **Long-term:**
   - Add comprehensive tests
   - Implement proper monitoring
   - Add performance metrics
   - Refactor duplicated code

---

## 📊 FINAL TỔNG KẾT TOÀN BỘ CODE REVIEW

### Thống kê tổng thể
- **Tổng số files đã review:** ~200+ files
- **Tổng số issues:**
  - 🔴 **Critical:** 70+ issues
  - 🟡 **Warning:** 100+ issues
  - 🟢 **Info:** 40+ issues

### Phân loại issues

| Loại Bug | Số lượng | Mức độ |
|----------|---------|--------|
| Security bugs | 30+ | 🔴 Critical |
| Memory leaks | 20+ | 🔴 Critical |
| Race conditions | 25+ | 🔴 Critical |
| Type safety | 50+ | 🟡 Warning |
| Performance bugs | 20+ | 🟡 Warning |
| Code quality | 80+ | 🟡 Warning |

### Top 15 Critical Issues cần fix ngay

1. **SSRF vulnerability** trong `api-proxy/[...path]/route.ts`
2. **XSS risks** trong `RichTextEditor.tsx`, `useAntiCheat.ts`, `app/layout.tsx`
3. **Memory leaks** trong `SocialContext.tsx`, `config/api.ts`, `ScrollAnimation.tsx`, `lib/utils/cookies.ts`
4. **Race conditions** trong `events/page.tsx`, `news/page.tsx`, `SocialContext.tsx`, `lib/utils/cookies.ts`
5. **No rate limiting** trong `api-proxy` routes và `auth/page.tsx`
6. **Cookie security** trong `api-proxy` và layouts
7. **Input validation** missing trong nhiều components
8. **Socket cleanup** không đầy đủ trong hooks
9. **Token refresh race** trong `config/api.ts`
10. **Hydration mismatches** trong `news/[id]/page.tsx`, `app/layout.tsx`
11. **Weak encryption key** trong `lib/utils/server-cookie-decrypt.ts`
12. **Token storage** trong `lib/socket/client.ts` (localStorage)
13. **Cookie cache leak** trong `lib/utils/cookies.ts`
14. **Promise cache** không có timeout trong `lib/utils/cookies.ts`
15. **Type inconsistency** trong `interface/auth.ts`

### Điểm mạnh tổng thể
- ✅ Code structure tương đối tốt
-  Có sử dụng React best practices (memoization, hooks)
-  Có error handling trong nhiều places
-  Có loading states
-  Có transaction queue để prevent race conditions
-  Interface definitions tương đối tốt
-  Có caching mechanism

### Điểm yếu tổng thể
-  Nhiều security vulnerabilities (XSS, SSRF, input validation, encryption)
-  Nhiều memory leaks (observers, caches, sockets, promises)
-  Nhiều race conditions (async operations, state updates)
-  Type safety issues (nhiều `any` types, inconsistent types)
-  Code duplication (socket clients, API calls)
-  Large components và contexts
-  Inconsistent error handling

### Recommended Next Steps

1. **Week 1-2: Critical Security Fixes**
   - Fix SSRF vulnerability
   - Fix XSS risks
   - Fix encryption key validation
   - Remove token storage from localStorage
   - Add input validation
   - Improve cookie security

2. **Week 3-4: Memory Leaks & Race Conditions**
   - Fix memory leaks (observers, caches, sockets)
   - Fix race conditions (async operations, state updates)
   - Fix cookie cache leak
   - Fix promise cache timeout
   - Improve cleanup logic
   - Add AbortController where needed

3. **Week 5-6: Performance & Code Quality**
   - Optimize re-renders
   - Split large components
   - Improve type safety (remove `any`, standardize types)
   - Reduce code duplication (base classes, shared utilities)
   - Fix type inconsistencies

4. **Week 7-8: Testing & Monitoring**
   - Add comprehensive tests
   - Implement monitoring
   - Add error tracking
   - Performance monitoring
   - Add analytics

---

**Reviewer:** AI Code Reviewer  
**Review Date:** 2026-01-21  
**Total Files Reviewed:** ~200+ files  
**Total Issues Found:** 210+ issues  
**Next Review:** Sau khi fix critical issues (estimated 2-3 months)
