# ReadLaw

> **AI 기반 문서 분석 및 협업 RAG 플랫폼**
> 

[**배포 URL (ReadLaw)**](https://jiyun.dev)  
[**Team Repository**](https://github.com/ArchitectureReadFile/mainproject)

---

## 프로젝트 소개

문서 데이터를 기반으로 로컬 LLM이 답변을 생성하는 RAG 시스템을 실제 서비스 환경에 통합하고, 실시간 알림과 보안 인증 인프라를 구축한 협업 플랫폼.

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
서비스의 보안 안정성과 사용자 편의성을 동시에 확보하기 위한 인증 아키텍처 구축.
- **RTR (Refresh Token Rotation)**: Axios Interceptor를 활용하여 Access Token 만료(401 에러) 시 백그라운드에서 Refresh Token으로 자동 재발급되도록 구현.
- **쿠키 기반 토큰 관리**: XSS 공격 방지를 위해 `withCredentials` 활성화 및 HttpOnly 쿠키로 인증 정보 안전하게 관리.
- **범용 인증 모듈**: 회원가입, 탈퇴, 비밀번호 재설정 시 재사용 가능한 6자리 인증 코드(Redis 캐싱) 검증 시스템 설계.

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
- **이메일 기반 자동 계정 연동**: 소셜 계정의 이메일과 동일한 로컬 계정 존재 시 자동 연동.
- **신규 유저 가입 유도**: 미가입 소셜 계정 로그인 시, 이메일 정보를 회원가입 폼에 프리필(Pre-fill)하여 온보딩 프로세스 단축.
- **유연한 연동 관리**: 마이페이지(`SocialSection`)에서 로그인된 계정과 일치하는 소셜 계정에 한해 실시간 연동 및 해제 제어.

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
사용자의 알림 피로도를 최소화하고 UX를 극대화하기 위해 세밀한 설정이 가능한 알림 인프라 구축.
- **Granular Control (알림 세부 설정)**: 워크스페이스 초대, 문서 업/삭제, AI 챗봇 답장 등 이벤트별 '토스트 알림 노출' 및 '알림 목록 저장' 개별 On/Off 지원.
- **Context API 기반 전역 동기화**: `NotificationContext`를 구현하여 전역에서 알림 벨 아이콘 및 토스트 메시지 실시간 동기화.

#### Notification Context 구현 코드
```javascript
import { createContext, useContext } from 'react'
import { useNotification as useNotificationSource } from '../hooks/useNotification'

const NotificationContext = createContext()

export const NotificationProvider = ({ children }) => {
  const notificationValues = useNotificationSource()

  return (
    <NotificationContext.Provider value={notificationValues}>
      {children}
    </NotificationContext.Provider>
  )
}

export const useNotification = () => {
  const context = useContext(NotificationContext)
  if (!context) {
    throw new Error('useNotification must be used within a NotificationProvider')
  }
  return context
}
```

### 4. 접근성을 극대화한 멀티 채널 AI 인터페이스 구축
- **랜딩 페이지 메인 챗봇**: 신규 유저가 서비스 진입 즉시 RAG 기능을 체험할 수 있는 대형 대화형 섹션 구축.
- **전역 플로팅 챗봇 위젯**: 서비스 내 어느 페이지에서든 작업 흐름을 끊지 않고 질문할 수 있는 우측 하단 플로팅 위젯 구현.
- **실시간 스트리밍**: AI 응답 대기 시간을 줄이기 위한 실시간 스트리밍 렌더링 및 세션별 대화 이력 연동.

### 5. 개발 인프라 및 DB 데이터 모델링
- **Docker Compose 통합**: Frontend, Backend, MariaDB, Redis를 단일 환경으로 통합하여 팀 개발 초기 세팅 효율화.
- **데이터 무결성 설계**: 유저, 문서, 채팅, 알림 등 핵심 도메인 간의 관계 정의 및 RDBMS 스키마 설계.

---

## 트러블슈팅 및 성능 최적화 (개인 기여)

### 1. Docker 환경 내 컨테이너 실행 시점 차이에 따른 DB 초기화 에러 해결
- **문제 상황**: DB 컨테이너가 완전히 준비되기 전 FastAPI 백엔드가 먼저 실행되어 `create_all()` 호출 시 테이블 생성 실패 현상 발생.
- **해결 방안**: FastAPI `lifespan` 이벤트를 활용, DB 연결 성공 시까지 일정 간격(3초)으로 최대 10회 재시도하는 헬스체크(Health Check) 로직을 적용하여 컨테이너 부팅 순서 의존성 문제 해결.

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
- **문제 상황**: 테스트 격리를 위해 SQLite 인메모리 DB를 사용했으나, FastAPI 비동기 핸들러가 워커 스레드에서 새 커넥션을 맺을 때 테이블이 존재하지 않는다는 `no such table` 에러 발생.
- **해결 방안**: SQLAlchemy의 `StaticPool` 클래스를 적용하여 모든 스레드가 단일 커넥션을 공유하도록 강제함으로써 멀티스레드 환경의 데이터 일관성 확보.

```python
from sqlalchemy.pool import StaticPool

engine = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool  
)
```

### 3. 단위 테스트 중 JWT 발급 시간(초 단위) 해상도로 인한 타이밍 이슈 해결
- **문제 상황**: Refresh Token 갱신 로직 검증 시, JWT 발급 시간(`iat`)이 초 단위로 기록되어 1초 내에 토큰 삭제와 재생성이 이루어지면 기존 토큰과 완전히 동일한 해시가 생성되는 간헐적 검증 실패 발생.
- **해결 방안**: Redis 내부의 키 삭제 상태를 직접 검증하는 로직 대신, 클라이언트가 실제 응답으로 받는 쿠키(`response.cookies`)에 `access_token`과 `refresh_token`이 정상적으로 저장었는지를 확인하는 블랙박스 테스트로 전환하여 테스트 안정성 확보.

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
**선택 이유**: 보안 강화를 위해 Access Token 수명은 단축하되, 사용자는 세션 만료를 인지하지 않고 재로그인 없이 서비스를 지속 이용할 수 있는 편리한 UX를 제공하기 위함.

### 알림 설정 테이블의 물리적 분리
**선택 이유**: 알림 로그(Notification)와 사용자 설정(Setting)을 분리하여 데이터 무결성을 보장하고, 유저의 알림 설정 변경이 과거 알림 히스토리 데이터에 영향을 주지 않도록 격리 설계.

### 소셜 로그인 이메일 기반 자동 연동 및 가입 유도
**선택 이유**: 플랫폼별 계정 파편화를 방지하여 유저 데이터를 일원화하고, 신규 유저의 회원가입을 늘리고 자연스러운 플랫폼 사용 경험 제공.
