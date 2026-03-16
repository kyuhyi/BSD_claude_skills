---
name: webapp-security
description: 웹앱 보안 체크리스트 및 구현 가이드. 새 프로젝트 생성 시 자동으로 보안 모범 사례를 적용합니다.
---

# 🔒 웹앱 보안 스킬

웹앱/API 개발 시 반드시 적용해야 할 보안 체크리스트와 구현 패턴.

## 적용 시점

- 새 웹앱/API 프로젝트 생성 시
- 사용자 입력을 처리하는 코드 작성 시
- 인증/권한 시스템 구현 시
- 파일 업로드/다운로드 기능 구현 시
- 외부 API 연동 시

---

## 1. 입력 검증 (Input Validation)

### 모든 사용자 입력은 악의적이라고 가정

```typescript
// ❌ 나쁜 예
const userId = req.params.id;
db.query(`SELECT * FROM users WHERE id = ${userId}`);

// ✅ 좋은 예
const userId = parseInt(req.params.id, 10);
if (isNaN(userId) || userId < 0) {
  throw new Error('Invalid user ID');
}
db.query('SELECT * FROM users WHERE id = ?', [userId]);
```

### 검증 체크리스트
- [ ] SQL Injection 방지: Prepared Statements 사용
- [ ] XSS 방지: 출력 시 HTML 이스케이프
- [ ] Command Injection 방지: 쉘 명령어 실행 금지 또는 화이트리스트
- [ ] Path Traversal 방지: `..` 경로 차단, 절대 경로 검증
- [ ] 타입 검증: 숫자, 문자열, 이메일 등 형식 검증
- [ ] 길이 제한: 최대 길이 설정

---

## 2. 인증 & 권한 (Auth)

### JWT 토큰 보안

```typescript
// ✅ 안전한 JWT 설정
const token = jwt.sign(payload, process.env.JWT_SECRET, {
  expiresIn: '1h',        // 짧은 만료 시간
  algorithm: 'HS256',     // 안전한 알고리즘
  issuer: 'your-app',
});

// ✅ 검증 시
jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'],  // 알고리즘 고정
  issuer: 'your-app',
});
```

### 비밀번호 보안

```typescript
import bcrypt from 'bcrypt';

// ✅ 해싱 (저장 시)
const hash = await bcrypt.hash(password, 12);

// ✅ 검증 (로그인 시)
const valid = await bcrypt.compare(password, hash);
```

### 체크리스트
- [ ] 비밀번호: bcrypt/argon2로 해싱 (최소 10 라운드)
- [ ] 세션: HTTP-Only, Secure, SameSite 쿠키
- [ ] JWT: 짧은 만료 시간, Refresh Token 사용
- [ ] API Key: 환경변수로 관리, 절대 코드에 하드코딩 금지
- [ ] 권한 검사: 모든 엔드포인트에서 권한 확인

---

## 3. Rate Limiting

### 구현 예시

```typescript
const rateLimit = new Map<string, { count: number; resetAt: number }>();

function checkRateLimit(userId: string, limit = 100, windowMs = 60000): boolean {
  const now = Date.now();
  const record = rateLimit.get(userId);
  
  if (!record || now > record.resetAt) {
    rateLimit.set(userId, { count: 1, resetAt: now + windowMs });
    return true;
  }
  
  if (record.count >= limit) {
    return false; // 차단
  }
  
  record.count++;
  return true;
}
```

### 권장 설정
| 엔드포인트 | 제한 |
|-----------|------|
| 로그인 | 5회/분 |
| 회원가입 | 3회/시간 |
| API 일반 | 100회/분 |
| 파일 업로드 | 10회/분 |

---

## 4. 파일 업로드 보안

```typescript
// ✅ 안전한 파일 업로드
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateFile(file: File): void {
  // 1. MIME 타입 검증
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new Error('허용되지 않은 파일 형식');
  }
  
  // 2. 크기 검증
  if (file.size > MAX_SIZE) {
    throw new Error('파일 크기 초과');
  }
  
  // 3. 파일명 검증 (Path Traversal 방지)
  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
  if (safeName.includes('..')) {
    throw new Error('잘못된 파일명');
  }
  
  // 4. 확장자 이중 검증
  const ext = path.extname(safeName).toLowerCase();
  if (!['.jpg', '.jpeg', '.png', '.gif'].includes(ext)) {
    throw new Error('허용되지 않은 확장자');
  }
}
```

### 체크리스트
- [ ] MIME 타입 화이트리스트
- [ ] 파일 크기 제한
- [ ] 파일명 새니타이징
- [ ] 업로드 폴더: 웹 루트 밖에 저장
- [ ] 실행 권한 제거

---

## 5. API 보안

### CORS 설정

```typescript
// ✅ 안전한 CORS
app.use(cors({
  origin: ['https://yourdomain.com'],  // 특정 도메인만
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
}));
```

### 보안 헤더

```typescript
// ✅ Helmet 사용 (Express)
import helmet from 'helmet';
app.use(helmet());

// 또는 수동 설정
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000');
  res.setHeader('Content-Security-Policy', "default-src 'self'");
  next();
});
```

---

## 6. 데이터베이스 보안

### SQL Injection 방지

```typescript
// ❌ 절대 금지
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Prepared Statements
db.query('SELECT * FROM users WHERE email = ?', [email]);

// ✅ ORM 사용 (Prisma, TypeORM 등)
const user = await prisma.user.findUnique({ where: { email } });
```

### 체크리스트
- [ ] Prepared Statements 또는 ORM 사용
- [ ] DB 계정: 최소 권한 원칙
- [ ] 민감 데이터 암호화 (PII, 카드번호 등)
- [ ] DB 접속: SSL/TLS 사용

---

## 7. 환경변수 & 시크릿

### .env 파일 구조

```bash
# .env.example (커밋 O)
DATABASE_URL=
JWT_SECRET=
API_KEY=

# .env (커밋 X - .gitignore에 추가)
DATABASE_URL=postgresql://user:pass@localhost/db
JWT_SECRET=your-super-secret-key-minimum-32-chars
API_KEY=sk-xxxxx
```

### 체크리스트
- [ ] .env 파일 .gitignore에 추가
- [ ] 시크릿: 최소 32자 랜덤 문자열
- [ ] 프로덕션: 환경변수 또는 시크릿 매니저 사용
- [ ] 로그에 시크릿 출력 금지

---

## 8. 프롬프트 인젝션 방지 (AI 앱용)

### 위험 패턴 감지

```typescript
const DANGEROUS_PATTERNS = [
  /ignore\s+(previous|all)\s+instructions/i,
  /disregard\s+.*\s+instructions/i,
  /forget\s+(everything|all)/i,
  /you\s+are\s+now/i,
  /act\s+as\s+(if|a|an)/i,
  /pretend\s+(you|to\s+be)/i,
  /system\s*:\s*/i,
  /\[\s*system\s*\]/i,
  /bypass\s+(security|filter)/i,
  /reveal\s+(your|the)\s+(prompt|instruction)/i,
];

function detectPromptInjection(input: string): boolean {
  return DANGEROUS_PATTERNS.some(p => p.test(input));
}
```

---

## 9. 로깅 & 모니터링

### 안전한 로깅

```typescript
// ❌ 민감 정보 로깅 금지
log.info({ password, creditCard, token });

// ✅ 안전한 로깅
log.info({ 
  userId: user.id,
  action: 'login',
  ip: req.ip,
  // password, token 등 민감 정보 제외
});
```

### 체크리스트
- [ ] 비밀번호, 토큰, 카드번호 로깅 금지
- [ ] 실패한 로그인 시도 기록
- [ ] 의심스러운 활동 알림 설정
- [ ] 로그 보관 기간 설정 (GDPR 등 규정 준수)

---

## 10. 보안 체크리스트 (배포 전)

### 필수
- [ ] HTTPS 강제 (SSL/TLS)
- [ ] 모든 환경변수 설정 완료
- [ ] .env, node_modules 등 민감 파일 .gitignore
- [ ] 의존성 취약점 스캔 (`npm audit`)
- [ ] 에러 메시지에 스택 트레이스 노출 안 함

### 권장
- [ ] WAF (Web Application Firewall) 설정
- [ ] DDoS 보호 활성화
- [ ] 정기 보안 감사 일정
- [ ] 침투 테스트 (년 1회 이상)

---

## 빠른 참조

```bash
# 의존성 취약점 검사
npm audit
npm audit fix

# 시크릿 생성 (32자)
openssl rand -base64 32

# HTTPS 인증서 (Let's Encrypt)
certbot --nginx -d yourdomain.com
```

---

*이 스킬은 OWASP Top 10, CWE/SANS Top 25 기반으로 작성되었습니다.*
