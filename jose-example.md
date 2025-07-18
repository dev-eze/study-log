JWT: “클레임이 담긴 JSON 토큰”
JWS: “위 JSON을 서명으로 감싼 컨테이너” → 서명된 JWT = JWS 객체


아래 예시는 Nimbus JOSE + JWT 라이브러리(최신 10.2 버전)를 이용해 아래 2가지 사항을 Spring Boot 3.x 환경에서 구현한 간단한 예제이다. 
- JWT(=JWS) 서명 발급
- JWT 검증·디코딩



1. 의존성 추가 (Maven / Gradle)

nimbus-jose-jwt 의존성 추가
- 10.2 버전은 2025-04-07 배포본이므로 최신 보안 패치·JDK 21까지 대응합니다. 


2. 키 준비 (예: RSA 4096 비트)

# test용 키쌍 생성
private.pem → 서명(Sign)
public.pem → 검증(Verify)·디코딩

3. JWT 발급 서비스 예제
@Service
@RequiredArgsConstructor
public class JwtIssuer {

    private final RSAPrivateKey privateKey;   // @Bean으로 주입
    private final ObjectMapper objectMapper;

    public String issue(User user) {
        Instant now = Instant.now();

        // ① JOSE Header
        JWSHeader header = new JWSHeader.Builder(JWSAlgorithm.RS256)
                            .type(JOSEObjectType.JWT)       // typ=JWT
                            .keyID("ecomm-rsa-v1")          // 선택
                            .build();

        // ② Claim Set
        JWTClaimsSet claims = new JWTClaimsSet.Builder()
                .subject(user.getId())             // sub
                .issuer("https://api.shop.com")    // iss
                .claim("roles", user.getRoles())   // 커스텀
                .issueTime(Date.from(now))
                .expirationTime(Date.from(now.plus(Duration.ofHours(2))))
                .build();

        // ③ JWS 객체 생성 & 서명
        SignedJWT jwt = new SignedJWT(header, claims);
        jwt.sign(new RSASSASigner(privateKey));

        return jwt.serialize();  // header.payload.signature 문자열
    }
}



스프링 부트용 RSA Key Bean

@Configuration
public class KeyConfig {
    @Bean
    RSAPrivateKey privateKey() throws Exception {
        try (PEMParser p = new PEMParser(new FileReader("private.pem"))) {
            return (RSAPrivateKey) new JcaPEMKeyConverter()
                   .getKeyPair((PEMKeyPair) p.readObject()).getPrivate();
        }
    }
    @Bean
    RSAPublicKey publicKey() throws Exception { /* 비슷하게 public.pem 로드 */ }
}


4. JWT 검증 & 디코딩 유틸

@Component
@RequiredArgsConstructor
public class JwtVerifier {

    private final RSAPublicKey publicKey;

    public JWTClaimsSet verify(String token) {
        try {
            SignedJWT jwt = SignedJWT.parse(token);

            JWSVerifier verifier = new RSASSAVerifier(publicKey);
            if (!jwt.verify(verifier))
                throw new JwtException("서명 불일치");

            if (jwt.getJWTClaimsSet().getExpirationTime()
                    .before(new Date()))
                throw new JwtException("만료 토큰");

            return jwt.getJWTClaimsSet();         // 디코딩 결과
        } catch (ParseException | JOSEException e) {
            throw new JwtException("JWT 파싱 실패", e);
        }
    }
}


Spring Security 필터 연동

@Bean
SecurityFilterChain api(HttpSecurity http, JwtVerifier verifier) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .addFilterBefore(new OncePerRequestFilter() {
            @Override
            protected void doFilterInternal(HttpServletRequest req,
                                            HttpServletResponse res,
                                            FilterChain chain)
                                            throws IOException, ServletException {
                String auth = req.getHeader(HttpHeaders.AUTHORIZATION);
                if (StringUtils.hasText(auth) && auth.startsWith("Bearer ")) {
                    JWTClaimsSet claims = verifier.verify(auth.substring(7));
                    UsernamePasswordAuthenticationToken at =
                        new UsernamePasswordAuthenticationToken(
                            claims.getSubject(), null,
                            AuthorityUtils.createAuthorityList("ROLE_USER"));
                    SecurityContextHolder.getContext().setAuthentication(at);
                }
                chain.doFilter(req, res);
            }
        }, UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(r -> r
            .requestMatchers("/orders/**").authenticated()
            .anyRequest().permitAll())
        .build();
}


5. 주문 API 바디를 추가로 서명(JWS)하는 예시
주문 데이터 자체를 application/jose 타입으로 서명해 보내면, 중간자 변조까지 탐지 가능


public String signOrderPayload(OrderRequest dto) throws Exception {
    JWSHeader h = new JWSHeader.Builder(JWSAlgorithm.RS256)
                    .contentType("application/json")
                    .build();

    Payload p = new Payload(objectMapper.writeValueAsBytes(dto));
    JWSObject jws = new JWSObject(h, p);
    jws.sign(new RSASSASigner(privateKey));
    return jws.serialize();
}

POST /orders
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/jose

eyJhbGciOiJSUzI1NiIsInR5cCI6IiJ9...
서버는 JwtVerifier와 유사하게 **JWSObject.verify(...)**만 호출하면 된다.


JOSE 라이브러리로 SignedJWT 발급 → JwtVerifier로 검증·디코딩.
주문 등 민감 바디는 추가 JWS 서명을 적용하면 변조 위험 이 줄어든다.

BFF를 두면 웹·모바일용 API를 각자 최적화하고, 위 JWT/JWS 로직을 공통으로 사용해 보안·UX 확보 할 수 있다. 
