# 04. 인증·인가 아키텍처 — JWT 오프로딩과 서비스 간 인증

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- API Gateway에서 JWT를 검증하고, 각 마이크로서비스는 검증된 정보를 신뢰하는 이유는?
- JWT의 구조(Header.Payload.Signature)가 왜 위변조 방지 메커니즘을 제공하는가?
- OAuth2 Authorization Code Flow에서 각 단계의 역할은 무엇이고, 왜 Authorization Code가 필요한가?
- 마이크로서비스 간 인증(Service-to-Service)에서 mTLS와 Service Account JWT의 차이는?
- End-User 인증과 Service 인증을 분리하는 것의 중요성은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA에서 **인증**과 **인가**는 API Gateway라는 단일 신뢰 경계(Trust Boundary)에서 처리되어야 합니다. 이유:

1. **중복 검증 방지**: 각 서비스가 JWT를 재검증하면 CPU 낭비 (수십 개의 서비스 × 매 요청마다 HMAC 계산)

2. **신뢰 경계 명확화**: 
   ```
   [외부 클라이언트] → (신뢰 경계) → [API Gateway] ← 신뢰 경계 내부
                                          ↓
                      [user-service] [order-service] [payment-service]
   ```
   - 외부에서 온 요청: Gateway에서 100% 검증
   - 내부 서비스 간 통신: 검증된 헤더 신뢰 (또는 mTLS로 이중 보장)

3. **확장성**: 새로운 마이크로서비스 추가 시 각각 인증 로직 구현 필요 없음

4. **외부 인증 통합**: OAuth2/OIDC(Google, GitHub 로그인)를 Gateway에서 한곳에서 관리

---

## 😱 흔한 실수 (Before)

### 오류 1: 모든 서비스에서 JWT 재검증

```java
// UserService.java - ❌ 비효율적 (JWT 재검증)
@RestController
@RequiredArgsConstructor
public class UserController {
    
    private final JwtValidator jwtValidator;  // 각 서비스가 검증 로직 포함
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@RequestHeader("Authorization") String token) {
        // Step 1: JWT 검증 (HMAC 계산, 서명 확인)
        Claims claims = jwtValidator.validateToken(token);
        
        // Step 2: 사용자 정보 추출
        String userId = (String) claims.get("sub");
        
        // Step 3: 비즈니스 로직
        User user = userRepository.findById(userId);
        return ResponseEntity.ok(user);
    }
}

// OrderService.java - ❌ 동일한 검증 로직 중복
@RestController
@RequiredArgsConstructor
public class OrderController {
    
    private final JwtValidator jwtValidator;  // 중복!
    
    @GetMapping("/orders/{id}")
    public ResponseEntity<Order> getOrder(@RequestHeader("Authorization") String token) {
        Claims claims = jwtValidator.validateToken(token);  // 중복 검증!
        String userId = (String) claims.get("sub");
        // ...
    }
}
```

**문제점:**
- 50개 서비스 × 1,000 req/s = 50,000회/s JWT 검증
- 각 검증에 ~5ms (HMAC 계산) → **250초/s의 CPU 낭비**
- 검증 로직 변경 시 모든 서비스 업데이트 필요

### 오류 2: 평문 Password를 마이크로서비스 간 공유

```java
// OrderService.java - ❌ 위험한 인증
@Component
public class PaymentServiceClient {
    
    private final RestTemplate restTemplate;
    
    public void processPayment(String orderId) {
        // Plain-text 비밀번호를 HTTP 헤더로 전송!
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-Username", "order-service");
        headers.add("X-Password", "order-service-secret-123");  // ❌ 위험!
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        restTemplate.postForObject("http://payment-service/pay", entity, String.class);
    }
}
```

**문제점:**
- 평문 비밀번호가 네트워크로 전송 (TLS 없으면 탈취 위험)
- 비밀번호 변경 시 모든 클라이언트 업데이트
- 감사(Audit) 불가능 (누가 호출했는지 추적 어려움)

### 오류 3: OAuth2 Authorization Code를 클라이언트에 노출

```java
// FrontendAPI.java - ❌ 클라이언트 보안 위반
@RestController
public class OAuthController {
    
    @GetMapping("/oauth/callback")
    public ResponseEntity<String> callback(@RequestParam String code) {
        // Authorization Code를 JSON으로 그대로 클라이언트에 반환!
        return ResponseEntity.ok("{\"code\": \"" + code + "\"}");
    }
}
```

**문제점:**
- Authorization Code는 단회용(One-time)이므로 탈취해도 Access Token을 얻으려면 Client Secret 필요
- 하지만 클라이언트 아래에서 코드를 노출하면 XSS 공격으로 탈취 가능
- Code는 Backend에서만 처리해야 함

---

## ✨ 올바른 접근 (After)

### API Gateway에서 JWT 검증 후 내부 헤더로 전달

```yaml
# application.yml - Spring Cloud Gateway 설정
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            # JWT 검증 필터 (Gateway에서 한 번만)
            - name: AuthorizationHeaderJwtFilter
              args:
                secretKey: ${jwt.secret}
                issuer: ${jwt.issuer}
            
            # 검증된 사용자 정보를 Internal 헤더로 전달
            - AddRequestHeader=X-User-ID, ${jwt.claim.sub}
            - AddRequestHeader=X-User-Roles, ${jwt.claim.roles}
```

```java
// AuthorizationHeaderJwtFilter.java - Gateway 레벨 JWT 검증
@Component
public class AuthorizationHeaderJwtFilter implements GlobalFilter, Ordered {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.issuer}")
    private String jwtIssuer;
    
    private final JwtParser jwtParser = Jwts.parserBuilder()
            .setSigningKey(Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8)))
            .setDefaultClaims(Map.of("iss", jwtIssuer))
            .build();
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return sendUnauthorized(exchange);
        }
        
        try {
            String token = authHeader.substring(7);
            Claims claims = jwtParser.parseClaimsJws(token).getBody();
            
            // ✅ JWT 검증 완료! 검증된 정보를 Internal 헤더로 저장
            ServerWebExchange modifiedExchange = exchange.mutate()
                    .request(req -> {
                        req.header("X-User-ID", (String) claims.get("sub"));
                        req.header("X-User-Email", (String) claims.get("email"));
                        req.header("X-User-Roles", claims.get("roles").toString());
                        
                        // 원본 Authorization 헤더는 제거 (이미 검증됨)
                        req.headers(headers -> headers.remove("Authorization"));
                    })
                    .build();
            
            return chain.filter(modifiedExchange);
            
        } catch (JwtException e) {
            log.warn("JWT 검증 실패: {}", e.getMessage());
            return sendUnauthorized(exchange);
        }
    }
    
    private Mono<Void> sendUnauthorized(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 100;
    }
}
```

### 마이크로서비스에서는 헤더를 신뢰

```java
// UserController.java - 서비스 레벨 (간단!)
@RestController
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(
            @PathVariable String id,
            @RequestHeader("X-User-ID") String userId,  // ← Gateway에서 검증됨
            @RequestHeader("X-User-Roles") String roles) {
        
        // JWT 검증 없음! (이미 Gateway에서 완료)
        // 비즈니스 로직만 처리
        
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
}
```

### OAuth2 Authorization Code Flow (안전한 구현)

```
┌────────────┐                  ┌──────────────┐
│            │                  │ Google/OAuth │
│  클라이언트 │                  │    Provider  │
│ (웹/모바일) │                  │              │
└────────────┘                  └──────────────┘
      ↑                                ↑
      │                                │
      └────────────────────────────────┘
                     ↓
            ┌──────────────────┐
            │ API Gateway      │
            │ (Backend BFF)    │
            └──────────────────┘
```

```java
// OAuth2Controller.java - Gateway에서 처리 (중요!)
@RestController
@RequiredArgsConstructor
public class OAuth2Controller {
    
    private final OAuth2Template oauth2Template;
    private final JwtProvider jwtProvider;
    
    // Step 1: 클라이언트가 로그인 요청
    @GetMapping("/oauth/login")
    public ResponseEntity<String> login() {
        // OAuth Provider로 리다이렉트 (Authorization Code 요청)
        String authUrl = oauth2Template.getAuthorizationUrl();
        return ResponseEntity.ok("{\"redirect_url\": \"" + authUrl + "\"}");
    }
    
    // Step 2: OAuth Provider가 Authorization Code와 함께 Callback
    // (클라이언트는 이 과정에 참여하지 않음)
    @GetMapping("/oauth/callback")
    public ResponseEntity<String> callback(@RequestParam String code) {
        try {
            // Step 3: Authorization Code를 Access Token으로 교환
            // ✅ 이 과정은 Backend-to-Backend (안전!)
            TokenResponse tokenResponse = oauth2Template.exchangeCodeForToken(code);
            
            // Step 4: Access Token으로 사용자 정보 조회
            UserInfo userInfo = oauth2Template.getUserInfo(tokenResponse.getAccessToken());
            
            // Step 5: 우리 서비스 JWT 발급 (클라이언트가 보관할 토큰)
            String ourJwt = jwtProvider.createToken(
                    userInfo.getId(),
                    userInfo.getEmail(),
                    userInfo.getRoles()
            );
            
            // Step 6: 클라이언트에 우리 JWT 반환 (Cookie 또는 SessionStorage)
            // ✅ Authorization Code는 클라이언트에 노출 안 함!
            response.addCookie(
                    ResponseCookie.from("access_token", ourJwt)
                            .httpOnly(true)  // 중요: XSS 방지
                            .secure(true)    // HTTPS only
                            .sameSite("Strict")
                            .maxAge(3600)
                            .build()
            );
            
            return ResponseEntity.ok("{\"success\": true}");
            
        } catch (OAuth2Exception e) {
            log.error("OAuth2 콜백 실패", e);
            return ResponseEntity.status(401).build();
        }
    }
}
```

---

## 🔬 내부 동작 원리 — JWT와 OAuth2 심층 분석

### JWT 구조와 검증 메커니즘

```
JWT 토큰:
┌─────────────────┬─────────────────────┬──────────────────────┐
│ HEADER          │ PAYLOAD             │ SIGNATURE            │
│ (Base64URL)     │ (Base64URL)         │ (Base64URL)          │
└─────────────────┴─────────────────────┴──────────────────────┘

예시:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiJ1c2VyMTIzIiwiZXhwIjoxNjI3OTEyMDAwLCJyb2xlcyI6WyJBRE1JTiJdfQ
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Step 1: Header 디코딩
▼
{"alg":"HS256","typ":"JWT"}

Step 2: Payload 디코딩
▼
{"sub":"user123","exp":1627912000,"roles":["ADMIN"]}

Step 3: Signature 검증
▼
HMAC-SHA256(
    Header + "." + Payload,
    secret_key
) == SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### JWT 검증 흐름 (위변조 방지)

```
시나리오: 공격자가 JWT를 수정하려고 시도

[원본 JWT]
Header:  {"alg":"HS256","typ":"JWT"}
Payload: {"sub":"user123","roles":["USER"]}
Secret:  my-secret-key-12345

검증 성공!

[공격자가 수정]
Header:  {"alg":"HS256","typ":"JWT"}
Payload: {"sub":"user123","roles":["ADMIN"]}  ← 역할을 ADMIN으로 변경!
Secret:  my-secret-key-12345 (알 수 없음)

문제: Secret Key를 모르므로 새로운 Signature를 계산할 수 없음!

공격자의 시도:
1. Signature도 함께 수정 (하지만 Secret을 모르므로 실패)
   → Server가 검증할 때 Signature 불일치
   
2. Signature를 그대로 사용
   → Server가 Payload를 다시 해시 계산했을 때
      ADMIN이 들어간 Payload의 해시는 원본 Signature와 불일치
      → 거부!

결론: 공개된 Secret Key가 없으면 JWT 위변조 불가능
```

### OAuth2 Authorization Code Flow (상세)

```
┌─────────────────────────────────────────────────────────────────┐
│ OAuth2 Authorization Code Flow                                  │
└─────────────────────────────────────────────────────────────────┘

1단계: 클라이언트가 로그인 버튼 클릭
┌─────────────────┐
│  웹 브라우저    │
│ [로그인 버튼]   │
└────────┬────────┘
         │
         ▼
┌────────────────────────────────────────────┐
│ API Gateway (BFF - Backend For Frontend)   │
│ /oauth/login 호출                          │
└────────┬───────────────────────────────────┘
         │
         ▼
2단계: 사용자를 OAuth Provider로 리다이렉트
┌────────────────────────────────────────────────────────────┐
│ Authorization Request:                                     │
│ GET https://google.com/oauth/authorize?                   │
│   client_id=YOUR_CLIENT_ID                                │
│   redirect_uri=https://yourapp.com/oauth/callback         │
│   scope=openid email profile                              │
│   response_type=code                                      │
│   state=random_string_for_csrf_protection                 │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Google (OAuth2 Provider)               │
│                                        │
│ [사용자가 Google 계정으로 로그인]      │
│ [계정 접근 권한 허용]                  │
└────────┬───────────────────────────────┘
         │
         ▼
3단계: Authorization Code와 함께 리다이렉트
┌────────────────────────────────────────────────────────────┐
│ Redirect Response:                                         │
│ HTTP 302                                                   │
│ Location: https://yourapp.com/oauth/callback?             │
│   code=AUTH_CODE_xyz123                                    │
│   state=random_string_for_csrf_protection                 │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────┐
│ API Gateway (Backend만 접근 가능!)                │
│ /oauth/callback 처리                              │
│                                                    │
│ 1. state 검증 (CSRF 공격 방지)                   │
│ 2. Authorization Code 검증                        │
│    (한 번만 사용 가능, 만료 5분)                 │
│ 3. Token Exchange 요청                            │
└────────┬───────────────────────────────────────────┘
         │
         ▼
4단계: Backend-to-Backend 토큰 교환 (클라이언트 모름!)
┌────────────────────────────────────────────────────────────┐
│ Token Request:                                             │
│ POST https://google.com/oauth/token                        │
│ Content-Type: application/x-www-form-urlencoded           │
│                                                            │
│ code=AUTH_CODE_xyz123                                      │
│ client_id=YOUR_CLIENT_ID                                  │
│ client_secret=YOUR_CLIENT_SECRET  ← 절대 노출 금지!      │
│ grant_type=authorization_code                             │
│ redirect_uri=https://yourapp.com/oauth/callback           │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Google (토큰 발급)                    │
└────────┬───────────────────────────────┘
         │
         ▼
5단계: Access Token, ID Token, Refresh Token 반환
┌────────────────────────────────────────────────────────────┐
│ Token Response:                                            │
│ {                                                          │
│   "access_token": "google_access_token_xyz",              │
│   "id_token": "eyJhbGc...",  (JWT)                        │
│   "refresh_token": "google_refresh_token_abc",            │
│   "expires_in": 3600,                                     │
│   "token_type": "Bearer"                                  │
│ }                                                          │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ API Gateway                                    │
│                                                │
│ 1. ID Token 검증 (Google 공개키로)            │
│ 2. 사용자 정보 추출 (sub, email)             │
│ 3. 우리 서비스 JWT 생성                       │
└────────┬───────────────────────────────────────┘
         │
         ▼
6단계: 클라이언트에 우리 JWT 반환 (안전한 Cookie)
┌────────────────────────────────────────────────────────────┐
│ HTTP 302 Redirect:                                         │
│ Location: https://yourapp.com/dashboard                   │
│                                                            │
│ Set-Cookie: access_token=our_jwt_token;                   │
│   HttpOnly; Secure; SameSite=Strict; Max-Age=3600         │
│                                                            │
│ ✅ 클라이언트는 우리 JWT만 보유                           │
│ ✅ Authorization Code, Google Token은 숨겨짐              │
└────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  웹 브라우저    │
│ (대시보드 표시) │
│                 │
│ Cookie에 JWT    │
│ 자동 포함됨      │
└─────────────────┘
```

### Service-to-Service 인증: mTLS vs Service JWT

```
시나리오: order-service가 payment-service 호출

[방법 1: mTLS (Mutual TLS)]
┌──────────────────┐
│ order-service    │
│                  │
│ client-cert.pem  │
│ client-key.pem   │
└────────┬─────────┘
         │ TLS Handshake
         │ (클라이언트 인증서 제시)
         ▼
┌──────────────────┐
│ payment-service  │
│                  │
│ ca-cert.pem      │
│ (클라이언트 검증)│
└──────────────────┘

장점:
✅ 상호 인증 (누가 누구인지 명확)
✅ 네트워크 레벨에서 암호화
✅ 인증서 기반 (수명 관리)

단점:
❌ 복잡한 인증서 관리
❌ 인증서 갱신 어려움
❌ Kubernetes 없으면 구성 복잡


[방법 2: Service Account JWT]
┌──────────────────┐
│ order-service    │
│                  │
│ Service Account  │
│ Secret: svc-123  │
└────────┬─────────┘
         │ JWT 생성 (Service Account Secret 사용)
         │ Authorization: Bearer eyJhbGc...
         │
         ▼
┌──────────────────┐
│ payment-service  │
│                  │
│ Service Account  │
│ Secret: svc-123  │
│ (JWT 검증)       │
└──────────────────┘

장점:
✅ 간단한 구성 (JWT 생성만 하면 됨)
✅ 런타임 중 비밀 교체 가능
❌ 평문 Secret이 공개되면 위험

주의: 비밀을 최상단(예: ConfigServer)에서 관리!
```

---

## 💻 JWT 생성·검증 및 OAuth2 통합 코드

```java
// JwtProvider.java - JWT 생성
@Component
public class JwtProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private long jwtExpiration;  // 밀리초 (예: 3600000 = 1시간)
    
    private final Key key;
    
    public JwtProvider(@Value("${jwt.secret}") String secret) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
    
    /**
     * JWT 토큰 생성
     */
    public String generateToken(UserPrincipal principal) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("email", principal.getEmail());
        claims.put("roles", principal.getRoles());
        claims.put("iat", System.currentTimeMillis() / 1000);  // 발급 시간
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(principal.getId())  // "sub" claim
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }
    
    /**
     * JWT 토큰 검증 및 파싱
     */
    public Claims validateToken(String token) throws JwtException {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(key)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            log.warn("JWT 만료됨: {}", e.getMessage());
            throw new TokenExpiredException("Token has expired");
        } catch (UnsupportedJwtException e) {
            log.warn("지원하지 않는 JWT 형식: {}", e.getMessage());
            throw new InvalidTokenException("Unsupported JWT format");
        } catch (MalformedJwtException e) {
            log.warn("잘못된 JWT 형식: {}", e.getMessage());
            throw new InvalidTokenException("Invalid JWT format");
        } catch (SignatureException e) {
            log.warn("JWT 서명 검증 실패: {}", e.getMessage());
            throw new InvalidTokenException("JWT signature validation failed");
        }
    }
    
    /**
     * Refresh Token으로 새로운 Access Token 발급
     */
    public String refreshToken(String refreshToken) {
        try {
            Claims claims = validateToken(refreshToken);
            UserPrincipal principal = new UserPrincipal(
                    claims.getSubject(),
                    (String) claims.get("email"),
                    (List<String>) claims.get("roles")
            );
            return generateToken(principal);
        } catch (JwtException e) {
            throw new InvalidTokenException("Cannot refresh token");
        }
    }
}
```

```java
// OAuth2Service.java - OAuth2 통합
@Service
@RequiredArgsConstructor
public class OAuth2Service {
    
    private final RestTemplate restTemplate;
    private final JwtProvider jwtProvider;
    private final UserRepository userRepository;
    
    @Value("${oauth2.google.client-id}")
    private String googleClientId;
    
    @Value("${oauth2.google.client-secret}")
    private String googleClientSecret;
    
    @Value("${oauth2.google.token-uri}")
    private String googleTokenUri;
    
    @Value("${oauth2.google.user-info-uri}")
    private String googleUserInfoUri;
    
    /**
     * Authorization Code를 Access Token으로 교환
     */
    public String exchangeCodeForToken(String code, String redirectUri) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        
        // 교환 요청
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("code", code);
        params.add("client_id", googleClientId);
        params.add("client_secret", googleClientSecret);
        params.add("redirect_uri", redirectUri);
        params.add("grant_type", "authorization_code");
        
        HttpEntity<MultiValueMap<String, String>> request = 
                new HttpEntity<>(params, headers);
        
        try {
            ResponseEntity<Map> response = restTemplate.postForEntity(
                    googleTokenUri,
                    request,
                    Map.class
            );
            
            return (String) response.getBody().get("access_token");
        } catch (HttpClientErrorException e) {
            throw new OAuth2Exception("Failed to exchange code for token: " + 
                    e.getMessage());
        }
    }
    
    /**
     * Access Token으로 사용자 정보 조회
     */
    public GoogleUserInfo getUserInfo(String accessToken) {
        HttpHeaders headers = new HttpHeaders();
        headers.add("Authorization", "Bearer " + accessToken);
        
        HttpEntity<String> request = new HttpEntity<>(headers);
        
        try {
            ResponseEntity<Map<String, Object>> response = restTemplate.exchange(
                    googleUserInfoUri,
                    HttpMethod.GET,
                    request,
                    new ParameterizedTypeReference<Map<String, Object>>() {}
            );
            
            Map<String, Object> body = response.getBody();
            return GoogleUserInfo.builder()
                    .id((String) body.get("sub"))
                    .email((String) body.get("email"))
                    .name((String) body.get("name"))
                    .picture((String) body.get("picture"))
                    .build();
        } catch (HttpClientErrorException e) {
            throw new OAuth2Exception("Failed to get user info: " + 
                    e.getMessage());
        }
    }
    
    /**
     * Google 정보로 사용자 조회 또는 생성
     */
    public User getUserOrCreate(GoogleUserInfo googleUserInfo) {
        return userRepository.findByEmail(googleUserInfo.getEmail())
                .orElseGet(() -> {
                    User newUser = User.builder()
                            .email(googleUserInfo.getEmail())
                            .name(googleUserInfo.getName())
                            .provider("GOOGLE")
                            .providerId(googleUserInfo.getId())
                            .roles(Collections.singletonList("USER"))
                            .build();
                    return userRepository.save(newUser);
                });
    }
    
    /**
     * 로그인 처리 (코드 → JWT)
     */
    public String authenticateWithGoogle(String code, String redirectUri) {
        // 1. Code를 Google Access Token으로 교환
        String googleAccessToken = exchangeCodeForToken(code, redirectUri);
        
        // 2. Google Access Token으로 사용자 정보 조회
        GoogleUserInfo googleUserInfo = getUserInfo(googleAccessToken);
        
        // 3. 사용자 조회 또는 생성
        User user = getUserOrCreate(googleUserInfo);
        
        // 4. 우리 서비스의 JWT 발급
        UserPrincipal principal = new UserPrincipal(
                user.getId(),
                user.getEmail(),
                user.getRoles()
        );
        
        return jwtProvider.generateToken(principal);
    }
}
```

```yaml
# application.yml - OAuth2 설정
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,email,profile
            redirect-uri: "{baseUrl}/oauth2/callback/{registrationId}"
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://www.googleapis.com/oauth2/v4/token
            user-info-uri: https://www.googleapis.com/oauth2/v1/userinfo
            user-name-attribute: sub

oauth2:
  google:
    client-id: ${GOOGLE_CLIENT_ID}
    client-secret: ${GOOGLE_CLIENT_SECRET}
    token-uri: https://www.googleapis.com/oauth2/v4/token
    user-info-uri: https://www.googleapis.com/oauth2/v1/userinfo

jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key-change-in-production}
  expiration: 3600000  # 1시간
  refresh-expiration: 604800000  # 7일
```

---

## 📊 패턴 비교

| 항목 | API Key | OAuth2 | JWT | mTLS | Service Account |
|------|:---:|:---:|:---:|:---:|:---:|
| **구현 복잡도** | 매우 낮음 | 높음 | 중간 | 높음 | 낮음 |
| **보안** | 약함 | 강함 | 중간 | 강함 | 중간 |
| **사용자 친화성** | 나쁨 | 우수 | 중간 | 매우 나쁨 | 중간 |
| **외부 인증 지원** | ❌ | ✅ (Google, GitHub) | 가능 | ❌ | ❌ |
| **권한 관리** | 있음 | 있음 | 있음 | 없음 | 있음 |
| **토큰 갱신** | 수동 | 자동 (Refresh Token) | Refresh Token | 인증서 갱신 | 수동 (Secret 갱신) |
| **서비스 간 통신** | 부적합 | 적합 | 매우 적합 | 매우 적합 | 적합 |
| **클라이언트-서버** | 부적합 | 적합 | 적합 | 부적합 | 부적합 |
| **타임아웃 대처** | 불가능 | 가능 (Refresh) | 가능 (Refresh) | N/A | 불가능 |

---

## ⚖️ 트레이드오프

### JWT의 장점

✅ **Stateless**: 서버가 토큰 상태를 저장할 필요 없음
- 수평 확장 용이 (어떤 Gateway 인스턴스로 가든 검증 가능)
- 세션 저장소(Redis) 불필요

✅ **자가 포함(Self-Contained)**: 토큰이 사용자 정보를 모두 포함
- 데이터베이스 조회 불필요
- 응답 속도 빠름

✅ **크로스 도메인**: 쿠키처럼 Same-Origin Policy 제약 없음
- API와 웹사이트가 다른 도메인에 있어도 사용 가능

### JWT의 단점

❌ **즉시 무효화 불가능**: JWT가 발급되면 만료될 때까지 유효
```
시나리오: 해킹된 계정의 JWT를 긴급히 무효화하고 싶음

결과: 만료될 때까지 계속 유효 (최대 1시간)

해결책:
1. Blacklist 유지 (Redis에 무효 토큰 저장) ← Stateful!
2. 짧은 만료 시간 설정 (5분) + Refresh Token (번거로움)
3. 토큰 재발급 API (클라이언트가 주기적 호출)
```

❌ **토큰 크기**: 많은 정보를 포함하면 토큰이 커짐
- HTTP 헤더 크기 제한 (보통 8KB)
- 매 요청마다 전송 (대역폭 낭비)

❌ **변조 방지만 가능, 암호화 안 됨**: 누구나 Payload 디코딩 가능
```
JWT: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.xyz...

누구나 Payload 읽기 가능:
echo "eyJzdWIiOiJ1c2VyMTIzIn0" | base64 -d
→ {"sub":"user123"}

해결책: 민감 정보(SSN, 비밀번호)는 JWT에 포함하지 말 것!
```

### OAuth2의 장점

✅ **외부 인증 통합**: Google, GitHub, Microsoft 계정으로 로그인
- 사용자 신뢰도 증가
- 비밀번호 관리 부담 감소

✅ **Refresh Token**: 자동으로 새 Access Token 발급
- Access Token은 짧은 수명 (15분)
- Refresh Token은 긴 수명 (7일)
- 해킹당해도 Refresh Token만 변경하면 됨

### OAuth2의 단점

❌ **높은 구현 복잡도**: 여러 단계 (Code → Token → UserInfo)
- 상태 관리 (state 파라미터)
- 타임아웃 처리
- 에러 처리

❌ **외부 의존성**: OAuth Provider 장애 → 전체 서비스 영향
```
Google이 다운됨 → 사용자가 로그인 불가
해결: 여러 OAuth Provider 지원 (Google + GitHub)
```

---

## 📌 핵심 정리

✅ **API Gateway에서 JWT 검증 (단일 책임)**
- 모든 요청에서 JWT 확인
- 검증된 정보를 Internal 헤더로 전달
- 백단 서비스는 헤더 신뢰 (재검증 불필요)

✅ **JWT 구조와 서명의 중요성**
- Header.Payload.Signature 3부분
- Signature는 Secret Key로 계산 → 위변조 방지
- Payload는 Base64 인코딩일 뿐 암호화 아님 (민감 정보 제외)

✅ **OAuth2 Authorization Code Flow**
- 클라이언트는 Authorization Code만 받음 (Access Token 아님)
- Backend에서 Code → Access Token 교환 (Secret Key 사용)
- 사용자 정보 조회 후 우리 서비스 JWT 발급

✅ **End-User 인증 vs Service 인증 분리**
- End-User: OAuth2 + JWT (사람이 사용)
- Service-to-Service: Service Account JWT 또는 mTLS (마이크로서비스 간)

✅ **Best Practices**
```
1. Access Token: 짧은 수명 (15-30분)
2. Refresh Token: 긴 수명 (7일), HttpOnly Cookie
3. JWT Secret: 256bit 이상, 환경변수로 관리
4. 민감 정보: JWT에 포함 금지 (SSN, 비밀번호)
5. HTTPS 필수: 모든 인증 통신
6. 즉시 무효화: Blacklist Redis 또는 짧은 수명
```

---

## 🤔 생각해볼 문제

**Q1.** Gateway에서 JWT를 검증 후 `X-User-ID` 헤더로 전달했는데, 악의적인 클라이언트가 자신의 ID를 다른 사용자 ID로 위조해서 보내면 어떻게 될까요?

<details><summary>해설 보기</summary>

**문제 상황:**

```
정상 흐름:
[클라이언트]
  ↓ Authorization: Bearer eyJ...
[Gateway]
  ✓ JWT 검증 성공
  ✓ X-User-ID: user123 헤더 추가 (제거)
[서비스]
  ← X-User-ID: user123 신뢰


공격자 시나리오:
[공격자 클라이언트]
  ↓ Authorization: Bearer eyJ... (자신의 유효한 JWT)
  ↓ X-User-ID: hacker (위조!)
[Gateway]
  ? 어떻게 처리?
```

**답변:**

Gateway의 역할이 **명확히 정의되어야 함**:

1. **옳은 방법 (권장): Gateway가 헤더 덮어씀**
   ```java
   ServerWebExchange modifiedExchange = exchange.mutate()
       .request(req -> {
           req.header("X-User-ID", claims.get("sub"));  // JWT에서 추출
           
           // 클라이언트가 보낸 X-User-ID 제거 (중요!)
           req.headers(headers -> headers.remove("X-User-ID"));
       })
       .build();
   ```
   - Gateway가 Authorization 헤더를 제거하고 새로 설정
   - 서비스는 Gateway에서 온 헤더만 신뢰

2. **잘못된 방법: Gateway가 클라이언트 헤더 신뢰**
   ```java
   String userId = exchange.getRequest()
       .getHeaders()
       .getFirst("X-User-ID");  // ❌ 클라이언트 입력 신뢰!
   ```
   - 공격자가 헤더 위조 가능
   - JWT 검증이 무의미해짐

3. **추가 보안 (Istio/Network Policy)**
   ```yaml
   # Kubernetes NetworkPolicy
   # user-service는 Gateway로부터의 X-User-ID만 수락
   # (다른 서비스나 직접 호출 차단)
   ```

**결론: Gateway가 Authorization 헤더를 제거하고, 검증된 정보를 새로운 Internal 헤더로 설정해야 함.**
</details>

**Q2.** JWT의 만료 시간이 1시간인데, 사용자가 로그인 후 55분 후에 요청하면 어떻게 될까요?

<details><summary>해설 보기</summary>

**시나리오:**

```
T=0분: 사용자 로그인
  ↓ JWT 발급 (exp: T=60분)
  ↓ 클라이언트가 JWT 보관

T=55분: 사용자가 요청
  ↓ Authorization: Bearer eyJ... (남은 시간: 5분)
  ↓ Gateway가 검증
  ✓ 토큰 유효 (exp > current_time)
  ↓ 요청 처리 시작

T=58분: 서비스가 응답 준비 (처리 시간 3분 소요)

T=61분: 토큰 만료!
  ↓ 아직 응답 중이어도 이미 expired
  ↓ 결과: ???
```

**해결 방법:**

1. **Access Token + Refresh Token 조합 (권장)**
   ```java
   // 액세스 토큰: 15분 (짧음)
   // 리프레시 토큰: 7일 (길음)
   
   클라이언트가 T=14분에 자동으로 리프레시:
   POST /oauth/refresh
   {
     "refresh_token": "..." (유효, 7일 남음)
   }
   
   응답:
   {
     "access_token": "새로운 JWT (15분)" ← 교체
   }
   ```

2. **Sliding Window (자동 연장)**
   ```java
   // 요청할 때마다 토큰 만료 시간 갱신
   if (timeUntilExpiry < 5분) {
       새로운 토큰 발급
       Authorization-Refresh 헤더로 응답
   }
   ```

3. **장시간 작업은 별도 Job ID로 관리**
   ```java
   // 오래 걸리는 작업
   POST /api/generate-report
   
   응답:
   {
     "job_id": "job_xyz",
     "status_url": "/api/jobs/job_xyz"
   }
   
   클라이언트는 job_id로 상태 조회 (JWT 필요 없음)
   ```

4. **서비스 내부: Timeout 재검증**
   ```java
   // 서비스가 처리 중 JWT 다시 확인
   @GetMapping("/process")
   public void process(@RequestHeader("X-User-ID") String userId) {
       // T=58분: 처리 시작
       
       // T=60분: 중간에 JWT 검증 재실행
       validateTokenStillValid(userId);
       
       // 만약 토큰 만료면 중단
       // (아니면 계속)
       
       // T=61분: 처리 완료, 응답
   }
   ```

**권장 선택: Access Token + Refresh Token**
- Access Token: 15분
- Refresh Token: 7일 (Secure HttpOnly Cookie에 저장)
- 사용자가 활동 중이면 자동 갱신
- 비활동 시 7일 후 자동 로그아웃
</details>

**Q3.** Service-to-Service 통신에서 mTLS를 사용할 때, 서비스 간에 인증서를 어떻게 배포하고 관리할까요?

<details><summary>해설 보기</summary>

**인증서 관리 전략:**

1. **Kubernetes Secrets를 통한 배포 (자동화)**
   ```bash
   # 1. CA, 서버 인증서, 개인키 생성
   openssl genrsa -out ca-key.pem 2048
   openssl req -new -x509 -days 365 -key ca-key.pem -out ca-cert.pem \
     -subj "/CN=my-ca"
   
   # 2. order-service 인증서
   openssl genrsa -out order-key.pem 2048
   openssl req -new -key order-key.pem -out order.csr \
     -subj "/CN=order-service"
   openssl x509 -req -in order.csr -CA ca-cert.pem \
     -CAkey ca-key.pem -CAcreateserial -out order-cert.pem -days 365
   
   # 3. payment-service 인증서
   openssl genrsa -out payment-key.pem 2048
   openssl req -new -key payment-key.pem -out payment.csr \
     -subj "/CN=payment-service"
   openssl x509 -req -in payment.csr -CA ca-cert.pem \
     -CAkey ca-key.pem -CAcreateserial -out payment-cert.pem -days 365
   
   # 4. Kubernetes Secret으로 배포
   kubectl create secret tls order-service-tls \
     --cert=order-cert.pem --key=order-key.pem -n default
   
   kubectl create secret tls payment-service-tls \
     --cert=payment-cert.pem --key=payment-key.pem -n default
   
   kubectl create secret generic ca-cert \
     --from-file=ca.crt=ca-cert.pem -n default
   ```

2. **Deployment에서 Secret 마운트**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: order-service
   spec:
     template:
       spec:
         containers:
         - name: order-service
           volumeMounts:
           - name: tls-certs
             mountPath: /etc/tls
             readOnly: true
         volumes:
         - name: tls-certs
           secret:
             secretName: order-service-tls
   ```

3. **Spring Boot 설정 (mTLS)**
   ```yaml
   # application.yml
   server:
     ssl:
       key-store: /etc/tls/tls.crt
       key-store-password: changeit
       key-store-type: PKCS12
       client-auth: need  # 클라이언트 인증서 필수
       trust-store: /etc/tls/ca.crt
       trust-store-password: changeit
   ```

4. **RestTemplate 설정 (mTLS 클라이언트)**
   ```java
   @Configuration
   public class MtlsClientConfig {
       
       @Bean
       public RestTemplate restTemplate() throws Exception {
           // 클라이언트 인증서 로드
           KeyStore keyStore = KeyStore.getInstance("PKCS12");
           keyStore.load(
               new FileInputStream("/etc/tls/tls.crt"),
               "changeit".toCharArray()
           );
           
           // CA 인증서 로드 (서버 검증용)
           KeyStore trustStore = KeyStore.getInstance("JKS");
           trustStore.load(
               new FileInputStream("/etc/tls/ca.crt"),
               "changeit".toCharArray()
           );
           
           // SSLContext 구성
           SSLContext sslContext = SSLContexts.custom()
               .loadKeyMaterial(keyStore, "changeit".toCharArray())
               .loadTrustMaterial(trustStore, new TrustSelfSignedStrategy())
               .build();
           
           // HttpClient 생성
           HttpClient httpClient = HttpClients.custom()
               .setSSLContext(sslContext)
               .build();
           
           return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
       }
   }
   ```

5. **인증서 만료 처리 (자동 갱신)**
   ```yaml
   # Kubernetes CronJob으로 인증서 자동 갱신
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: cert-renewal
   spec:
     schedule: "0 0 * * 0"  # 매주 일요일
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: cert-renewer
               image: cert-renewer:latest
               command:
               - /bin/sh
               - -c
               - |
                 # 인증서 갱신
                 openssl x509 -req -in order.csr \
                   -CA ca-cert.pem -CAkey ca-key.pem \
                   -CAcreateserial -out order-cert.pem -days 365
                 
                 # Secret 업데이트
                 kubectl patch secret order-service-tls \
                   --type merge \
                   -p '{"data":{"tls.crt":"'$(base64 order-cert.pem)'"}}'
             restartPolicy: OnFailure
   ```

6. **Istio를 이용한 자동 관리 (권장)**
   ```yaml
   # Istio가 자동으로 mTLS 인증서 관리
   apiVersion: security.istio.io/v1beta1
   kind: PeerAuthentication
   metadata:
     name: default
   spec:
     mtls:
       mode: STRICT  # 모든 서비스 간 mTLS 필수
   ---
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: allow-order-to-payment
   spec:
     selector:
       matchLabels:
         app: payment-service
     rules:
     - from:
       - source:
           principals: ["cluster.local/ns/default/sa/order-service"]
       to:
       - operation:
           methods: ["POST"]
           paths: ["/api/pay"]
   ```

**권장 선택:**

| 환경 | 권장 방법 |
|------|---------|
| 로컬 개발 | Service Account JWT |
| On-Prem Kubernetes | Istio + mTLS |
| 클라우드 (AWS/GCP) | Istio + Workload Identity |
| 멀티클라우드 | Vault + mTLS |

**Istio 사용 시 이점:**
- 인증서 자동 생성·갱신
- 정책 기반 접근 제어
- 트래픽 암호화 자동 처리
</details>

---

<div align="center">

**[⬅️ 이전: Rate Limiting 구현 — Token Bucket과 분산 카운터](./03-rate-limiting.md)** | **[홈으로 🏠](../README.md)** | **[다음: BFF 패턴 — 클라이언트별 최적화 API 레이어 ➡️](./05-bff-pattern.md)**

</div>
