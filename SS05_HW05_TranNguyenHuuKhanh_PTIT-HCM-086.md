# BÀI 5: Thiết kế Quy trình & Prompt cho Rate Limiter

## 1. Ý đồ thiết kế quy trình 3 bước

Quy trình 3 bước được thiết kế để không ép AI viết code ngay. Đầu tiên, AI phải tư vấn thuật toán và so sánh trade-offs. Sau đó, AI được đặt vào kịch bản phân tán để nhận ra giới hạn của local memory. Cuối cùng, AI mới sinh mã nguồn theo cấu trúc có phân tích từng bước, có Redis, có fallback khi Redis lỗi và có logging chuyên nghiệp.

Mục tiêu của quy trình:

1. Chọn đúng thuật toán rate limiting.
2. Thiết kế phù hợp cho hệ thống nhiều server.
3. Sinh mã nguồn Java Spring Boot có khả năng dùng trong môi trường production.

## 2. Prompt bước 1: Tư vấn thuật toán và Trade-offs

```text
Hãy đóng vai trò là System Engineer chuyên thiết kế API Gateway cho ứng dụng tài chính.

Bối cảnh:
Tôi đang xây dựng API Gateway cho ứng dụng tài chính SafePay. Mỗi API Key chỉ được phép gửi tối đa 100 requests trong vòng 1 phút. Nếu vượt quá giới hạn, hệ thống phải trả về HTTP 429 Too Many Requests.

Nhiệm vụ:
Hãy tư vấn thuật toán Rate Limiting phù hợp trước khi viết code.

Yêu cầu:
1. Đề xuất ít nhất 2 thuật toán phổ biến, bắt buộc có:
   - Fixed Window
   - Token Bucket
2. Có thể bổ sung Sliding Window nếu cần.
3. Lập bảng so sánh ưu điểm, nhược điểm, độ phức tạp, độ chính xác và mức độ phù hợp với Java Spring Boot.
4. Phân tích rủi ro burst traffic ở biên cửa sổ thời gian.
5. Kết luận nên chọn thuật toán nào cho SafePay và giải thích lý do.

Trả lời bằng tiếng Việt, rõ ràng, có bảng so sánh.
```

## 3. Prompt bước 2: What-if Distributed Scenario với Redis

```text
Tiếp tục với bài toán Rate Limiter cho SafePay.

Giả định mới:
Hệ thống API Gateway không chạy trên 1 server nữa mà chạy trên cụm 3 server sau load balancer. Nếu mỗi server lưu biến đếm request trong local memory thì cùng một API Key có thể gửi vượt quá 100 requests/phút vì request bị chia sang nhiều server.

Nhiệm vụ:
Hãy phân tích vì sao local memory bị sai trong distributed system và đề xuất giải pháp dùng Redis làm kho lưu trữ tập trung.

Yêu cầu:
1. Giải thích lỗi sai khi dùng local memory trong cụm 3 server.
2. Đề xuất kiến trúc dùng Redis tập trung.
3. Mô tả luồng xử lý request:
   - Nhận API Key
   - Tạo Redis key theo API Key và time window
   - INCR counter
   - Set TTL 60 giây
   - Nếu counter > 100 thì trả HTTP 429
4. Phân tích rủi ro race condition và cách Redis INCR giúp xử lý nguyên tử.
5. Đề xuất fallback nếu Redis bị lỗi.

Trả lời bằng tiếng Việt, có sơ đồ luồng dạng text.
```

## 4. Prompt bước 3: Sinh mã nguồn CoT có Redis, fallback và log

```text
Hãy đóng vai trò là Senior Java Spring Boot Developer.

Bối cảnh:
Tôi cần triển khai Rate Limiter phân tán cho API Gateway của SafePay. Mỗi API Key chỉ được phép gửi tối đa 100 requests trong 1 phút. Nếu vượt quá giới hạn, trả HTTP 429 Too Many Requests.

Ràng buộc kỹ thuật:
- Java 17
- Spring Boot 3
- Spring Data Redis
- Dùng StringRedisTemplate
- Logging bằng Lombok @Slf4j
- Code phải dễ đọc, có thể đưa vào dự án thực tế

Yêu cầu quy trình:
Đừng viết code ngay. Hãy trình bày ngắn gọn các bước thiết kế chính trước, gồm:
1. Cách tạo Redis key theo apiKey và time window.
2. Cách dùng INCR và EXPIRE/TTL để đếm request trong 1 phút.
3. Cách quyết định allow hoặc reject request.
4. Cách fallback nếu Redis đột ngột sập.

Sau phần thiết kế, hãy sinh mã nguồn Java hoàn chỉnh gồm:
1. RateLimiterService
2. ApiKeyRateLimitFilter hoặc HandlerInterceptor
3. Custom exception hoặc response HTTP 429
4. Cơ chế fallback khi Redis lỗi:
   - Ghi log lỗi Redis
   - Dùng local in-memory limiter tạm thời với ConcurrentHashMap
   - Có TTL đơn giản cho local window
5. Log đầy đủ bằng @Slf4j.

Trả lời bằng tiếng Việt và đặt toàn bộ mã nguồn trong khối code markdown.
```

## 5. Minh chứng thực tế: Log chat 3 lượt với AI

### Lượt chat 1: So sánh thuật toán

```markdown
User:
Hãy đóng vai trò là System Engineer chuyên thiết kế API Gateway cho ứng dụng tài chính.
Bối cảnh: Tôi đang xây dựng API Gateway cho ứng dụng tài chính SafePay. Mỗi API Key chỉ được phép gửi tối đa 100 requests trong vòng 1 phút. Nếu vượt quá giới hạn, hệ thống phải trả về HTTP 429 Too Many Requests.
Hãy tư vấn thuật toán Rate Limiting phù hợp, đề xuất Fixed Window, Token Bucket và so sánh trade-offs.

AI:
Đối với SafePay, có thể cân nhắc 3 thuật toán phổ biến:

| Thuật toán | Ưu điểm | Nhược điểm | Độ phức tạp | Phù hợp |
|---|---|---|---|---|
| Fixed Window | Dễ triển khai, dễ hiểu, chỉ cần counter và TTL | Có thể bị burst ở biên cửa sổ. Ví dụ user gửi 100 request ở giây 59 và 100 request ở giây 61 | Thấp | Phù hợp bản đầu, dễ triển khai với Redis |
| Token Bucket | Cho phép kiểm soát tốc độ trung bình và vẫn cho burst nhỏ hợp lệ | Cần lưu số token và thời điểm refill, logic phức tạp hơn | Trung bình | Phù hợp hệ thống production cần mềm dẻo |
| Sliding Window | Chính xác hơn Fixed Window, giảm burst ở biên | Cần lưu nhiều timestamp hoặc tính toán phức tạp hơn | Cao hơn | Phù hợp khi yêu cầu giới hạn rất chính xác |

Với quy tắc 100 requests/phút, Fixed Window là lựa chọn đơn giản và phù hợp nếu hệ thống muốn triển khai nhanh với Redis. Tuy nhiên, với ứng dụng tài chính, nếu cần chống abuse nghiêm ngặt hơn, Token Bucket hoặc Sliding Window sẽ tốt hơn.

Khuyến nghị: chọn Fixed Window + Redis cho phiên bản đầu vì dễ triển khai, dễ kiểm chứng, Redis INCR hỗ trợ tăng counter nguyên tử. Sau đó có thể nâng cấp lên Sliding Window nếu phát hiện burst traffic ở biên cửa sổ gây rủi ro.
```

### Lượt chat 2: Thiết kế phân tán với Redis

```markdown
User:
Tiếp tục với bài toán Rate Limiter cho SafePay. Giả định hệ thống API Gateway chạy trên cụm 3 server sau load balancer. Nếu mỗi server lưu biến đếm request trong local memory thì sẽ bị sai lệch. Hãy đề xuất giải pháp Redis tập trung.

AI:
Khi chạy 3 server, local memory không còn chính xác. Ví dụ API Key A gửi 90 request vào server 1, 90 request vào server 2 và 90 request vào server 3. Mỗi server đều thấy chưa vượt 100 request/phút, nhưng tổng thực tế là 270 request/phút. Vì vậy, local counter phá vỡ giới hạn toàn cục.

Giải pháp là dùng Redis làm kho đếm tập trung:

```text
Client Request
    |
    v
Load Balancer
    |
    +--> API Gateway Server 1
    +--> API Gateway Server 2
    +--> API Gateway Server 3
              |
              v
          Redis Central Counter
```

Luồng xử lý:

1. Gateway nhận request và đọc API Key.
2. Tạo Redis key: `rate_limit:{apiKey}:{yyyyMMddHHmm}`.
3. Gọi Redis `INCR key` để tăng counter.
4. Nếu counter bằng 1 thì set TTL 60 giây.
5. Nếu counter lớn hơn 100 thì reject request với HTTP 429.
6. Nếu counter nhỏ hơn hoặc bằng 100 thì cho request đi tiếp.

Redis `INCR` là thao tác nguyên tử, nên khi nhiều server cùng tăng counter, Redis vẫn đảm bảo không bị race condition ở mức tăng số đếm.

Fallback khi Redis lỗi:
- Fail-closed: chặn request để bảo vệ hệ thống, an toàn hơn nhưng có thể ảnh hưởng user hợp lệ.
- Fail-open: cho request đi qua, đảm bảo trải nghiệm nhưng giảm khả năng chống abuse.
- Hybrid: dùng local in-memory limiter tạm thời và log cảnh báo. Đây là phương án cân bằng.
```

### Lượt chat 3: Mã nguồn Java Redis Rate Limiter

```markdown
User:
Hãy đóng vai trò Senior Java Spring Boot Developer. Viết mã nguồn Java Spring Boot 3 dùng Spring Data Redis để triển khai Rate Limiter phân tán cho SafePay. Mỗi API Key tối đa 100 requests/phút. Nếu Redis sập thì fallback sang local in-memory limiter bằng ConcurrentHashMap. Dùng @Slf4j.

AI:
Thiết kế chính:

1. Redis key được tạo theo API Key và phút hiện tại: `rate_limit:{apiKey}:{yyyyMMddHHmm}`.
2. Mỗi request gọi `INCR` trên Redis key.
3. Nếu counter lần đầu bằng 1, đặt TTL 60 giây.
4. Nếu counter > 100, request bị từ chối với HTTP 429.
5. Nếu Redis lỗi, hệ thống ghi log và dùng local fallback bằng ConcurrentHashMap.

```java
package com.safepay.gateway.ratelimit;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Service
@RequiredArgsConstructor
@Slf4j
public class RateLimiterService {

    private static final int LIMIT = 100;
    private static final Duration WINDOW_TTL = Duration.ofSeconds(60);
    private static final DateTimeFormatter WINDOW_FORMATTER = DateTimeFormatter.ofPattern("yyyyMMddHHmm");

    private final StringRedisTemplate redisTemplate;
    private final LocalFallbackRateLimiter localFallbackRateLimiter = new LocalFallbackRateLimiter();

    public boolean isAllowed(String apiKey) {
        String redisKey = buildRedisKey(apiKey);

        try {
            Long currentCount = redisTemplate.opsForValue().increment(redisKey);

            if (currentCount == null) {
                log.warn("Redis INCR returned null, apiKey={}", mask(apiKey));
                return localFallbackRateLimiter.isAllowed(apiKey);
            }

            if (currentCount == 1L) {
                redisTemplate.expire(redisKey, WINDOW_TTL);
            }

            if (currentCount > LIMIT) {
                log.warn("Rate limit exceeded by Redis, apiKey={}, count={}", mask(apiKey), currentCount);
                return false;
            }

            return true;
        } catch (Exception ex) {
            log.error("Redis rate limiter failed, fallback to local limiter, apiKey={}, reason={}",
                    mask(apiKey), ex.getMessage(), ex);
            return localFallbackRateLimiter.isAllowed(apiKey);
        }
    }

    private String buildRedisKey(String apiKey) {
        String window = LocalDateTime.now().format(WINDOW_FORMATTER);
        return "rate_limit:" + apiKey + ":" + window;
    }

    private String mask(String apiKey) {
        if (apiKey == null || apiKey.length() <= 4) {
            return "****";
        }
        return "****" + apiKey.substring(apiKey.length() - 4);
    }
}

@Slf4j
class LocalFallbackRateLimiter {

    private static final int LIMIT = 100;
    private static final long WINDOW_MILLIS = 60_000L;

    private final Map<String, LocalCounter> counters = new ConcurrentHashMap<>();

    public boolean isAllowed(String apiKey) {
        long now = System.currentTimeMillis();

        LocalCounter counter = counters.compute(apiKey, (key, oldCounter) -> {
            if (oldCounter == null || now >= oldCounter.windowExpiresAt()) {
                return new LocalCounter(new AtomicInteger(1), now + WINDOW_MILLIS);
            }
            oldCounter.count().incrementAndGet();
            return oldCounter;
        });

        int currentCount = counter.count().get();

        if (currentCount > LIMIT) {
            log.warn("Rate limit exceeded by local fallback, apiKey={}, count={}", apiKey, currentCount);
            return false;
        }

        return true;
    }

    private record LocalCounter(AtomicInteger count, long windowExpiresAt) {
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
class ApiKeyRateLimitFilter extends OncePerRequestFilter {

    private final RateLimiterService rateLimiterService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String apiKey = request.getHeader("X-API-Key");

        if (apiKey == null || apiKey.isBlank()) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("Missing API Key");
            return;
        }

        boolean allowed = rateLimiterService.isAllowed(apiKey);

        if (!allowed) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Too Many Requests");
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```
```

## 6. Mã nguồn Java hoàn chỉnh rút gọn để nộp

```java
package com.safepay.gateway.ratelimit;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Service
@RequiredArgsConstructor
@Slf4j
class RateLimiterService {

    private static final int LIMIT = 100;
    private static final Duration WINDOW_TTL = Duration.ofSeconds(60);
    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyyMMddHHmm");

    private final StringRedisTemplate redisTemplate;
    private final LocalFallbackRateLimiter fallbackRateLimiter = new LocalFallbackRateLimiter();

    public boolean isAllowed(String apiKey) {
        String key = "rate_limit:" + apiKey + ":" + LocalDateTime.now().format(FORMATTER);

        try {
            Long count = redisTemplate.opsForValue().increment(key);
            if (count == null) {
                return fallbackRateLimiter.isAllowed(apiKey);
            }

            if (count == 1L) {
                redisTemplate.expire(key, WINDOW_TTL);
            }

            if (count > LIMIT) {
                log.warn("API key exceeded rate limit by Redis, apiKey={}, count={}", mask(apiKey), count);
                return false;
            }

            return true;
        } catch (Exception ex) {
            log.error("Redis unavailable, using local fallback limiter, apiKey={}, error={}",
                    mask(apiKey), ex.getMessage(), ex);
            return fallbackRateLimiter.isAllowed(apiKey);
        }
    }

    private String mask(String apiKey) {
        if (apiKey == null || apiKey.length() < 4) {
            return "****";
        }
        return "****" + apiKey.substring(apiKey.length() - 4);
    }
}

@Slf4j
class LocalFallbackRateLimiter {

    private static final int LIMIT = 100;
    private static final long WINDOW_MILLIS = 60_000L;

    private final Map<String, LocalCounter> counters = new ConcurrentHashMap<>();

    public boolean isAllowed(String apiKey) {
        long now = System.currentTimeMillis();

        LocalCounter counter = counters.compute(apiKey, (key, oldValue) -> {
            if (oldValue == null || now > oldValue.expiresAt()) {
                return new LocalCounter(new AtomicInteger(1), now + WINDOW_MILLIS);
            }
            oldValue.count().incrementAndGet();
            return oldValue;
        });

        int count = counter.count().get();
        if (count > LIMIT) {
            log.warn("API key exceeded rate limit by local fallback, apiKey={}, count={}", apiKey, count);
            return false;
        }
        return true;
    }

    private record LocalCounter(AtomicInteger count, long expiresAt) {
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
class ApiKeyRateLimitFilter extends OncePerRequestFilter {

    private final RateLimiterService rateLimiterService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String apiKey = request.getHeader("X-API-Key");

        if (apiKey == null || apiKey.isBlank()) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("Missing API Key");
            return;
        }

        if (!rateLimiterService.isAllowed(apiKey)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Too Many Requests");
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```
