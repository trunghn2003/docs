# Giải thích chi tiết về cách thức hoạt động của Eureka và API Gateway trong hệ thống microservices của bạn

## 1. Eureka Server - Service Discovery

### Nguyên lý hoạt động

Eureka Server là một implementation của Service Discovery Pattern trong kiến trúc microservices, được phát triển bởi Netflix và tích hợp vào Spring Cloud.

#### Đăng ký dịch vụ (Service Registration)

1. **Khởi động Eureka Server**:
   - Khi khởi động, Eureka Server mở một API endpoint (thường là `/eureka/apps`) để các service đăng ký.
   - Trong dự án của bạn, Eureka Server chạy trên port 8761 (được cấu hình qua biến môi trường `EUREKA_SERVER_PORT`).

2. **Client đăng ký**:
   - Khi một service khởi động (auth-service, user-service, movie-show-service, v.v.), nó gửi một HTTP POST request đến Eureka Server với thông tin:
     - Service ID/Name (ví dụ: "AUTH-SERVICE")
     - Host/IP
     - Port
     - Health check URL
   - Service được đăng ký với status "UP"

3. **Heartbeat mechanism**:
   - Sau khi đăng ký, service gửi heartbeat đến Eureka Server theo định kỳ (mặc định là 30 giây).
   - Nếu Eureka không nhận được heartbeat sau một khoảng thời gian (mặc định là 90 giây), service được đánh dấu là "DOWN".
   - Nếu không nhận được heartbeat sau thời gian dài hơn, service bị xóa khỏi registry.

#### Tìm kiếm dịch vụ (Service Discovery)

1. **Client lookup**:
   - Khi một service cần gọi service khác (ví dụ: task-service cần gọi movie-show-service), nó gửi request đến Eureka Server để lấy thông tin về service đích.
   - Request này là HTTP GET đến `/eureka/apps/{SERVICE-ID}`.

2. **Client-side cache**:
   - Các service lưu cache registry từ Eureka Server (mặc định refresh 30 giây một lần).
   - Việc này giúp giảm tải cho Eureka Server và cải thiện hiệu suất.

3. **Load balancing**:
   - Nếu có nhiều instances của một service, client-side load balancing được thực hiện (thường dùng Ribbon).
   - Trong dự án của bạn, điều này được cấu hình qua `ribbon.eureka.enabled=true`.

#### Self-preservation mode

- Khi Eureka Server không nhận được heartbeat từ một tỷ lệ lớn các service (cấu hình qua `eureka.server.renewal-percent-threshold`), nó vào chế độ self-preservation.
- Trong chế độ này, Eureka không xóa các service đã đăng ký ngay cả khi chúng không gửi heartbeat.
- Điều này bảo vệ hệ thống trong trường hợp có vấn đề về network partition.

### Cấu hình Eureka trong dự án của bạn

```yaml
server:
  port: ${SERVER_PORT:8761}

eureka:
  instance:
    hostname: ${EUREKA_INSTANCE_HOSTNAME:localhost}
  client:
    register-with-eureka: ${EUREKA_CLIENT_REGISTERWITHEUREKASERVER:false}
    fetch-registry: ${EUREKA_CLIENT_FETCHREGISTRY:false}
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: ${EUREKA_SERVER_ENABLESELFPRESERVATION:true}
    renewal-percent-threshold: ${EUREKA_SERVER_RENEWALPERCENTTHRESHOLD:0.85}
```

- `register-with-eureka: false` - Eureka Server không tự đăng ký chính nó
- `fetch-registry: false` - Eureka Server không fetch registry (vì nó là server)
- `enable-self-preservation: true` - Bật chế độ tự bảo vệ
- `renewal-percent-threshold: 0.85` - Ngưỡng để kích hoạt chế độ tự bảo vệ (85%)

## 2. API Gateway - Spring Cloud Gateway

### Nguyên lý hoạt động

API Gateway là một pattern trong kiến trúc microservices, hoạt động như một điểm vào duy nhất cho tất cả các client (web, mobile, external systems).

#### Định tuyến request (Routing)

1. **Nhận request**:
   - Gateway nhận HTTP request từ client (browser, mobile app, v.v.).
   - Trong dự án của bạn, Gateway chạy trên port 8080 (cấu hình qua `GATEWAY_PORT`).

2. **Route matching**:
   - Gateway sử dụng các "predicates" để xác định route phù hợp cho request.
   - Predicates phổ biến: Path, Method, Header, Query param, v.v.
   - Ví dụ trong dự án của bạn:
     ```yaml
     predicates:
       - Path=/api/auth/**
     ```
     - Route này khớp với bất kỳ request nào có path bắt đầu với `/api/auth/`.

3. **Service discovery integration**:
   - Gateway tích hợp với Eureka để tìm service đích.
   - Khi định tuyến request, Gateway sử dụng service ID để lookup địa chỉ thực của service qua Eureka.
   - Cấu hình trong dự án của bạn:
     ```yaml
     spring:
       cloud:
         gateway:
           discovery:
             locator:
               enabled: true
     ```

4. **Load balancing**:
   - Nếu có nhiều instances của service đích, Gateway tự động load balance giữa chúng.
   - Sử dụng Spring Cloud LoadBalancer (hoặc Netflix Ribbon trong các phiên bản cũ hơn).

#### Filter chain

1. **Pre-filters**:
   - Thực hiện trước khi request được forward đến service đích.
   - Ví dụ: Authentication, logging, rate limiting, v.v.

2. **Post-filters**:
   - Thực hiện sau khi nhận response từ service đích.
   - Ví dụ: Response header modification, logging, metrics collection, v.v.

3. **Global filters**:
   - Áp dụng cho tất cả các routes.
   - Trong dự án của bạn:
     ```yaml
     filters:
       - StripPrefix=1
     ```
     - Filter này loại bỏ segment đầu tiên của URI trước khi gửi đến service đích.

#### Cross-Origin Resource Sharing (CORS)

API Gateway của bạn xử lý CORS để cho phép frontend giao tiếp với các API:

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: ${GATEWAY_CORS_ALLOWED_ORIGINS:http://localhost:3000}
            allowedMethods: ${GATEWAY_CORS_ALLOWED_METHODS:GET,POST,PUT,DELETE,OPTIONS}
            allowedHeaders: ${GATEWAY_CORS_ALLOWED_HEADERS:*}
            allowCredentials: ${GATEWAY_CORS_ALLOW_CREDENTIALS:true}
            maxAge: ${GATEWAY_CORS_MAX_AGE:3600}
```

### Cấu hình Gateway trong dự án của bạn

```yaml
server:
  port: ${SERVER_PORT:8080}

spring:
  application:
    name: gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: auth-service
          uri: lb://AUTH-SERVICE
          predicates:
            - Path=/api/auth/**
          filters:
            - RewritePath=/api/auth/(?<segment>.*), /api/auth/${segment}
            
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<segment>.*), /api/users/${segment}
            
        # Các routes khác tương tự...
```

- `uri: lb://AUTH-SERVICE` - Sử dụng load balancer (lb) để tìm service "AUTH-SERVICE" thông qua Eureka
- `RewritePath` - Filter để rewrite URL trước khi forward request
- `lower-case-service-id: true` - Cho phép sử dụng service ID viết thường

## 3. Luồng hoạt động tổng thể

1. **Service registration**:
   - Các microservices (auth-service, user-service, v.v.) khởi động và đăng ký với Eureka Server.
   - Thông tin đăng ký bao gồm service name, host, port, health URL.

2. **Client request**:
   - Client (browser/mobile app) gửi HTTP request đến API Gateway.
   - URL có dạng: `http://gateway:8080/api/auth/signin`

3. **Gateway routing**:
   - Gateway kiểm tra URL và tìm route phù hợp (ví dụ: `/api/auth/**` khớp với auth-service).
   - Gateway áp dụng các pre-filters (authentication, logging, v.v.).
   - Gateway sử dụng Eureka để tìm địa chỉ thực của service (ví dụ: `auth-service:8081`).

4. **Service discovery**:
   - Gateway queries Eureka: "Đâu là địa chỉ của AUTH-SERVICE?"
   - Eureka trả về: "AUTH-SERVICE đang chạy tại auth-service:8081"

5. **Load balancing** (nếu có nhiều instances):
   - Nếu có nhiều instances của auth-service, Gateway chọn một instance dựa trên thuật toán load balancing.

6. **Request forwarding**:
   - Gateway forward request đến service đích (ví dụ: `http://auth-service:8081/api/auth/signin`).
   - Gateway đợi response từ service.

7. **Response handling**:
   - Gateway nhận response từ service.
   - Gateway áp dụng các post-filters.
   - Gateway trả response về cho client.

8. **Service-to-service communication**:
   - Khi một service cần gọi service khác (ví dụ: task-service gọi movie-show-service):
     - Service queries Eureka để lấy địa chỉ của service đích.
     - Service thực hiện HTTP request trực tiếp đến service đích.
     - Giao tiếp này không đi qua Gateway.

## 4. Khả năng mở rộng và khả năng chịu lỗi (Scalability & Fault Tolerance)

### Khả năng mở rộng

1. **Horizontal scaling của services**:
   - Bạn có thể chạy nhiều instances của cùng một service.
   - Các instances mới tự động đăng ký với Eureka.
   - Gateway và các service khác tự động load balance giữa các instances.

2. **Eureka clustering**:
   - Trong môi trường production, bạn có thể chạy nhiều Eureka Servers để tăng availability.
   - Mỗi Eureka Server biết về các server khác và đồng bộ registry.

### Khả năng chịu lỗi

1. **Service failure**:
   - Nếu một service instance bị lỗi, nó sẽ không gửi heartbeat đến Eureka.
   - Sau thời gian chờ, Eureka đánh dấu instance là "DOWN".
   - Gateway và các service khác không gửi request đến instance bị lỗi.
   - Nếu có instance khác của service đó, request sẽ được gửi đến instance còn lại.

2. **Self-preservation mode**:
   - Bảo vệ registry khi có vấn đề về network.
   - Ngăn việc xóa instance do network partition.

3. **Circuit Breaker pattern**:
   - Trong dự án của bạn, có thể thêm Resilience4J/Hystrix để thực hiện circuit breaking.
   - Khi một service liên tục fail, circuit breaker mở ra và fallback response được trả về.

## 5. Kết luận

Eureka và API Gateway là hai thành phần quan trọng trong kiến trúc microservices của bạn:

- **Eureka Server** cung cấp service discovery, cho phép các services tìm và giao tiếp với nhau một cách động, không cần hardcode địa chỉ.

- **API Gateway** cung cấp single entry point cho client, xử lý cross-cutting concerns (authentication, CORS, logging), và routing requests đến đúng service.

Kiến trúc này cho phép:
- Phát triển, triển khai và scale từng service độc lập
- Tăng khả năng chịu lỗi của hệ thống
- Đơn giản hóa giao tiếp client-service
- Cải thiện bảo mật bằng cách ẩn các internal services

Với việc sử dụng các biến môi trường, bạn cũng có thể dễ dàng cấu hình hệ thống cho các môi trường khác nhau mà không cần thay đổi code.
# Giải thích về cấu hình Spring Cloud Gateway

Đoạn code này là cấu hình cho Spring Cloud Gateway, một thành phần quan trọng trong kiến trúc microservices của bạn, có nhiệm vụ định tuyến các yêu cầu API từ client đến các dịch vụ nội bộ tương ứng. Hãy phân tích chi tiết:

## Tổng quan về lớp `SpringCloudConfig`

Đây là một lớp cấu hình (`@Configuration`) xác định cách API Gateway định tuyến các request HTTP đến các microservices khác nhau trong hệ thống:

```java
@Configuration
public class SpringCloudConfig {
    // ...
}
```

## Cấu hình Authentication Filter

```java
@Autowired
private AuthenticationFilter authFilter;

private AuthenticationFilter.Config configWithAuth(boolean roleValidationRequired,
        java.util.List<String> requiredRoles) {
    AuthenticationFilter.Config config = new AuthenticationFilter.Config();
    config.setRoleValidationRequired(roleValidationRequired);
    config.setRequiredRoles(requiredRoles);
    return config;
}
```

Phương thức `configWithAuth` tạo cấu hình cho bộ lọc xác thực với:
- `roleValidationRequired`: Xác định liệu endpoint có yêu cầu kiểm tra role hay không
- `requiredRoles`: Danh sách các role được phép truy cập endpoint

## Cấu hình định tuyến (Route Configuration)

Bean `customRouteLocator` định nghĩa tất cả các tuyến đường cho API Gateway:

### 1. Các endpoint công khai của Auth Service (không yêu cầu xác thực)

```java
.route("auth-service-public-login",
        r -> r.path("/api/auth/login").or().path("/api/auth/signin")
                .filters(f -> f
                        .rewritePath("/api/auth/(?<path>.*)", "/api/auth/${path}")
                        .filter((exchange, chain) -> {
                            // Logging filter
                        }))
                .uri("lb://auth-service"))
```

- **ID**: `auth-service-public-login`
- **Path**: Khớp với `/api/auth/login` hoặc `/api/auth/signin`
- **Filters**:
  - `rewritePath`: Chuyển đổi đường dẫn trước khi gửi đến service (giữ nguyên trong trường hợp này)
  - Filter logging để ghi lại thông tin request/response
- **URI**: `lb://auth-service` - sử dụng load balancer để gửi đến service "auth-service" đã đăng ký với Eureka

Tương tự cho các endpoint public khác: `register`, `signup`, và `validate-token`.

### 2. Các endpoint bảo vệ của Auth Service (yêu cầu xác thực)

```java
.route("auth-service-protected",
        r -> r.path("/api/auth/**")
                .and()
                .not(r1 -> r1.path("/api/auth/login", "/api/auth/register", "/api/auth/validate-token"))
                .filters(f -> f
                        .rewritePath("/api/auth/(?<path>.*)", "/api/auth/${path}")
                        .filter(authFilter.apply(configWithAuth(false, null)))
                        // Logging filter
                )
                .uri("lb://auth-service"))
```

- **Path**: Bất kỳ đường dẫn nào khớp với `/api/auth/**` nhưng không phải `/api/auth/login`, `/api/auth/register`, hoặc `/api/auth/validate-token`
- **Filters**: 
  - Áp dụng AuthenticationFilter (yêu cầu user đã xác thực nhưng không kiểm tra role)
  - Giá trị `false` cho `roleValidationRequired` và `null` cho `requiredRoles` nghĩa là chỉ cần user đã đăng nhập

### 3. Các endpoint User Service (yêu cầu role USER hoặc ADMIN hoặc STAFF)

```java
.route("user-service-standard", 
       r -> r.path("/api/users/**")
               .filters(f -> f
                       .rewritePath("/api/users/(?<path>.*)", "/api/users/${path}")
                       .filter(authFilter.apply(
                               configWithAuth(true, Arrays.asList("ROLE_USER", "ROLE_ADMIN", "ROLE_STAFF"))))
                       // Logging filter
               )
               .uri("lb://user-service"))
```

- **Path**: Khớp với `/api/users/**`
- **Filters**:
  - Yêu cầu xác thực với kiểm tra role (`roleValidationRequired = true`)
  - Chỉ những user có role "ROLE_USER", "ROLE_ADMIN", hoặc "ROLE_STAFF" mới được phép truy cập

### 4. Các endpoint Movie Show Service

```java
.route("movie-show-service", 
       r -> r.path("/api/movieshows/**", "/api/shows/**", "/api/tickets/**", "/api/snacks/**")
               .filters(f -> f
                       .rewritePath("/api/(?<path>.*)", "/api/${path}")
                       .filter(authFilter.apply(
                               configWithAuth(true, Arrays.asList("ROLE_USER", "ROLE_ADMIN", "ROLE_STAFF")))))
               .uri("lb://movie-show-service"))
```

- **Path**: Khớp với nhiều đường dẫn khác nhau liên quan đến chức năng movie show
- **Filters**: Yêu cầu xác thực với role "ROLE_USER", "ROLE_ADMIN", hoặc "ROLE_STAFF"

### 5. Các endpoint Bill Service

```java
.route("bill-service", 
       r -> r.path("/api/bills/**", "/api/payments/**")
               // Cấu hình filter tương tự
               .uri("lb://bill-service"))
```

### 6. Task Service (Booking Service)

```java
.route("task-service", 
       r -> r.path("/api/booking/**")
               .filters(f -> f
                       .rewritePath("/api/(?<path>.*)", "/api/${path}")
                       .filter(authFilter.apply(
                               configWithAuth(true, Arrays.asList("ROLE_USER")))))
               .uri("lb://task-service"))
```

- **Đặc biệt**: Chỉ cho phép người dùng có role "ROLE_USER" (không cho phép ADMIN hoặc STAFF) 

### 7. Notification Service

```java
.route("notification-service", 
       r -> r.path("/api/notifications/**")
               // Cấu hình filter tương tự
               .uri("lb://notification-service"))
```

## Tóm tắt

Lớp `SpringCloudConfig` định nghĩa cách thức API Gateway:

1. **Phân biệt giữa các endpoint công khai và bảo vệ:**
   - Endpoints công khai: Không yêu cầu xác thực (login, register, validate-token)
   - Endpoints bảo vệ: Yêu cầu xác thực và có thể yêu cầu role cụ thể

2. **Áp dụng các policy bảo mật khác nhau:**
   - Một số endpoint chỉ yêu cầu user đã đăng nhập
   - Một số endpoint yêu cầu user có role cụ thể (USER, ADMIN, STAFF)
   - Endpoint booking chỉ cho phép USER thông thường truy cập

3. **Rewrite path và định tuyến:**
   - Chuyển đổi đường dẫn từ gateway sang định dạng mà microservice mong đợi
   - Sử dụng load balancer để định tuyến request đến các microservices

4. **Logging và giám sát:**
   - Ghi lại thông tin về request path và response status để debug và giám sát

Đây là một thiết kế bảo mật tốt cho gateway, áp dụng mô hình "defense in depth" với nhiều lớp bảo vệ và kiểm soát truy cập chi tiết đến từng endpoint.
