# Readfile

> **AI 기반 문서 분석 및 협업 RAG 플랫폼**
> 
> 복잡한 문서(판례, 계약서 등)를 AI가 분석하고, 사용자가 대화형 인터페이스를 통해 필요한 정보를 즉각적으로 추출할 수 있도록 돕는 협업 툴입니다.

[**배포 URL (Readfile)**](https://jiyun.dev)  
[**Team Repository**](https://github.com/ArchitectureReadFile/mainproject)

---

## 프로젝트 소개

문서 데이터를 기반으로 로컬 LLM이 답변을 생성하는 RAG 시스템을 실제 서비스 환경에 통합하고, 사용자가 협업하며 사용할 수 있도록 실시간 알림과 보안 인증 인프라를 구축한 프로젝트입니다.

- **개발 기간**: 2026.02 ~ 2026.04
- **개발 인원**: 3인 풀스택 (본인 기여: 인증, 실시간 알림, 마이페이지, AI 채팅, 초기 인프라 및 DB 스키마 설계)

---

## 기술 스택

### Backend & Infra
| 기술 | 용도 |
|------|------|
| Python (FastAPI) | 비동기 기반 REST API 서버 구축 |
| SQLAlchemy | MariaDB ORM 매핑 및 데이터베이스 스키마 설계 |
| Redis | 인증 코드 캐싱 및 실시간 메시지 브로커 |
| Docker Compose | 전체 서비스 컨테이너화 |
| Celery | 채팅 비동기 처리 워커 |

### Frontend
| 기술 | 용도 |
|------|------|
| React | 사용자 인터페이스 구현 |
| Axios | Interceptor를 활용한 RTR(Refresh Token Rotation) 기법 구현 |
| Context API | 전역 알림 상태 및 유저 인증 정보 관리 |

---

## 프로젝트 구조

### Backend - 도메인 및 계층별 구조
```text
backend/
├── models/
│   └── model.py                  
├── routers/
│   ├── auth.py                   
│   ├── oauth.py
│   ├── notification.py           
│   ├── chat.py                   
│   └── ws.py                     
├── services/
│   ├── auth_service.py
│   ├── oauth_service.py
│   ├── chat(chat_processor.py, chat_service.py)        
│   └── notification_service.py   
└── repositories/
    └── oauth_repository.py
```

### Frontend - Feature 기반 구조
```text
src/
├── api/
│   └── client.js                 
├── features/
│   ├── auth/                     
│   │   ├── context/AuthContext.jsx
│   │   └── utils/authCookie.js
│   ├── notification/             
│   │   ├── context/NotificationContext.jsx
│   │   └── components/NotificationToast.jsx
│   └── chat/                     
│       └── components/ChatWidget.jsx
└── pages/
    ├── Landing/                  
    └── Mypage/                   
        └── components/SocialSection.jsx
```

---

## 주요 기능 및 구현 상세 (개인 기여)

### 1. 보안 중심 인증 시스템 설계
단순한 로그인을 넘어 서비스의 보안 안정성과 사용자 편의성을 동시에 확보하기 위한 인증 아키텍처를 구축했습니다.
- **RTR (Refresh Token Rotation)**: Axios Interceptor를 활용하여 Access Token 만료 시(401 에러) 백그라운드에서 자동으로 Refresh Token을 통해 토큰을 재발급받도록 구현했습니다.
- **쿠키 기반 토큰 관리**: XSS 공격 방지를 위해 `withCredentials` 옵션을 활성화하고 HttpOnly 쿠키로 인증 정보를 안전하게 관리하도록 설계했습니다.
- **범용 인증 모듈**: 회원가입, 회원탈퇴, 비밀번호 재설정, 이메일 변경 시 재사용 가능한 6자리 인증 코드(Redis 캐싱) 검증 시스템을 설계했습니다.

#### Axios Interceptor 구현 코드
```javascript
import axios from 'axios'
import { ERROR_CODE } from '../lib/errors'

const client = axios.create({
  baseURL: `/api`,
  withCredentials: true,
})

const tryRefreshToken = async (originalRequest) => {
  originalRequest._retry = true
  try {
    await axios.post(`/api/auth/refresh`, {}, { withCredentials: true })
    return client(originalRequest)
  } catch {
    window.location.href = '/'
    return Promise.reject(new Error('세션이 만료되었습니다. 다시 로그인해주세요.'))
  }
}

client.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config
    const code = error.response?.data?.code ?? null
    
    const isAuthEndpoint =
      originalRequest.url.includes('/auth/login') ||
      originalRequest.url.includes('/auth/refresh') ||
      originalRequest.url.includes('/auth/me')

    if (
      !originalRequest._retry &&
      !isAuthEndpoint &&
      (code === ERROR_CODE.AUTH_TOKEN_MISSING || code === ERROR_CODE.AUTH_TOKEN_INVALID)
    ) {
      return tryRefreshToken(originalRequest)
    }

    return Promise.reject(error)
  }
)

export default client
```

### 2. OAuth 2.0 소셜 로그인 연동 (Google, Github)
다양한 소셜 계정 연동을 통해 사용자 경험을 개선하고 계정 통합 관리 기능을 구현했습니다.
- **이메일 기반 자동 계정 연동**: 소셜 계정의 이메일과 동일한 이메일로 가입된 기존 계정이 존재할 경우, 별도의 절차 없이 자동으로 계정을 연동하도록 설계했습니다.
- **신규 유저 가입 유도**: 미가입 소셜 계정의 경우, 인증 완료된 이메일이라는 안내와 함께 회원가입 폼으로 리다이렉트되며 이메일이 자동 등록되어 가입을 유도하도록 설계했습니다.
- **유연한 연동 관리**: 마이페이지의 `SocialSection`을 통해 실시간으로 소셜 계정 연동 및 해제가 가능하며, 현재 로그인된 계정과 일치하는 소셜 계정만 제어할 수 있도록 보안 로직을 적용했습니다.

#### 소셜 로그인 처리 핵심 로직 (Backend)
```python
async def handle_oauth_login(self, email: str, provider: str, oauth_id: str, db: Session):
    user = await self.user_repo.get_user_by_email(db, email)
    
    if user:
        oauth_account = await self.oauth_repo.get_oauth_account(db, user.id, provider)
        if not oauth_account:
            await self.oauth_repo.create_oauth_account(
                db, user_id=user.id, provider=provider, oauth_id=oauth_id
            )
        return {"status": "success", "user_id": user.id}
    
    return {
        "status": "needs_registration",
        "email": email,
        "message": "인증 완료된 이메일입니다. 회원가입을 완료해주세요."
    }
```

### 3. 지능형 실시간 알림 시스템
사용자의 알림 피로도를 최소화하고 UX를 극대화하기 위해 세밀한 설정이 가능한 알림 인프라를 구축했습니다.
- **Granular Control (알림 세부 설정)**: 워크스페이스 초대, 문서 업로드/삭제, AI 챗봇 답장 등 이벤트별로 '실시간 토스트 알림 노출'과 '알림 목록 저장' 여부를 개별적으로 껐다 켤 수 있도록 DB 스키마와 API를 설계했습니다.
- **Context API 기반 전역 상태 동기화**: `NotificationContext`를 구현하여 서비스 어느 페이지에 있든 실시간으로 알림 벨 아이콘과 토스트 메시지가 동기화되도록 처리했습니다.

### 4. 접근성을 극대화한 멀티 채널 AI 인터페이스 구축
팀에서 구축한 핵심 RAG 엔진을 사용자가 가장 편리하게 이용할 수 있도록, 목적에 맞게 이원화된 채팅 인터페이스를 설계하고 통합했습니다.
- **랜딩 페이지 메인 챗봇**: 신규 사용자가 서비스 진입 즉시 AI RAG 기능을 직관적으로 체험할 수 있도록 랜딩 페이지 내에 대형 대화형 섹션을 구축하여 서비스 진입 장벽을 낮췄습니다.
- **전역 플로팅 챗봇 위젯**: 사용자가 워크스페이스 등 서비스 내 어느 페이지에 있더라도 작업 흐름을 끊지 않고 즉각적으로 질문할 수 있도록 우측 하단에 플로팅 챗봇 위젯을 별도로 구현했습니다.

### 5. 개발 인프라 및 DB 데이터 모델링
- **Docker Compose 통합**: Frontend, Backend, MariaDB, Redis를 하나의 환경으로 통합하여 팀 협업 효율을 증대시켰습니다.
- **데이터 무결성 설계**: 유저, 문서, 채팅, 알림 등 핵심 도메인 간의 관계를 정의하고 인덱싱 최적화를 고려한 스키마를 설계했습니다.

---

## 트러블슈팅 및 성능 최적화 (개인 기여)

### 1. Docker 환경에서 FastAPI와 DB 컨테이너 실행 시점 차이에 따른 테이블 미생성 문제 해결
- **문제 상황**: DB 컨테이너가 준비되기 전 백엔드가 실행되어 테이블 생성에 실패하는 현상 발생.
- **해결 방안**: FastAPI `lifespan` 이벤트를 활용하여 DB 연결 성공 시까지 일정 간격으로 재시도하는 로직을 적용했습니다.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    for i in range(10):
        try:
            Base.metadata.create_all(bind=engine)
            break 
        except Exception as e:
            time.sleep(3)
    yield

app = FastAPI(lifespan=lifespan)
```

### 2. Pytest 환경에서 SQLite 인메모리 DB의 데이터 격리 문제 해결
- **문제 상황**: 비동기 핸들러가 워커 스레드에서 새 커넥션을 맺을 때 기존 테이블을 찾지 못하는 문제 발생.
- **해결 방안**: SQLAlchemy의 `StaticPool`을 사용하여 단일 커넥션을 모든 스레드가 공유하도록 설정했습니다.

```python
from sqlalchemy.pool import StaticPool

engine = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool  
)
```

### 3. 단위 테스트 중 초 단위 타임스탬프 해상도로 인한 JWT 타이밍 이슈 해결
- **문제 상황**: 1초 내에 동일한 토큰이 생성되어 삭제 검증 로직이 실패하는 간헐적 오류 발생.
- **해결 방안**: 내부 상태 검증 대신 클라이언트에게 전달되는 쿠키의 존재 여부를 확인하는 블랙박스 테스트 방식으로 전환하여 안정성을 확보했습니다.

```python
with patch("routers.auth.redis_client", fake_redis):
    response = client.post("/api/auth/refresh")

    assert response.status_code == 200
    assert "access_token" in response.cookies
    assert "refresh_token" in response.cookies
```

---

## 설계 결정

### Axios Interceptor를 이용한 RTR 기법 도입
**선택 이유**: 보안을 위해 Access Token 수명을 단축하면서도, 사용자가 재로그인 없이 서비스를 지속적으로 이용할 수 있는 심리스한 UX를 제공하기 위함입니다.

### 알림 설정 테이블의 물리적 분리
**선택 이유**: 알림 로그(Notification)와 사용자 설정(Setting)을 분리하여 데이터 무결성을 보장하고, 설정 변경이 과거 기록에 영향을 주지 않도록 설계했습니다.

### 소셜 로그인 이메일 기반 자동 연동 및 가입 유도
**선택 이유**: 플랫폼별로 계정이 파편화되는 현상을 방지하여 유저 데이터 관리를 일원화하고, 기존 가입자의 접근성 향상 및 신규 유저의 회원가입 이탈률을 최소화하는 매끄러운 온보딩(Onboarding) 경험을 제공하기 위해 설계했습니다.
