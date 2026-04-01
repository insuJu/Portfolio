# Readfile

> **AI 기반 문서 분석 및 협업 RAG 플랫폼**
> 
> 복잡한 문서(판례, 계약서 등)를 AI가 분석하고, 사용자가 대화형 인터페이스를 통해 필요한 정보를 즉각적으로 추출할 수 있도록 돕는 협업 툴입니다.

[**배포 URL (Readfile)**](https://jiyun.dev)  
[**Team Repository**](https://github.com/ArchitectureReadFile/mainproject)

---

## 프로젝트 소개

문서 데이터를 기반으로 로컬 LLM이 답변을 생성하는 RAG 시스템을 실제 서비스 환경에 통합하고, 사용자가 협업하며 사용할 수 있도록 실시간 알림과 보안 인증 인프라를 구축한 프로젝트입니다.

- **개발 기간**: 2026.02 ~ (진행 중)
- **개발 인원**: 3인 풀스택 (본인 기여: 인증, 실시간 알림, 마이페이지, Ai 채팅, 초기 인프라 및 DB 스키마 설계)

---

## 기술 스택

### Backend & AI Infra
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
│   ├── notification.py           
│   ├── chat.py                   
│   └── ws.py                     
└── services/
    ├── auth_service.py
    ├── chat(chat_processor.py, chat_service.py)        
    └── notification_service.py   
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
```

---

## 주요 기능 및 구현 상세 (개인 기여)

### 1. 보안 중심 인증 시스템 설계 (RTR 기법)
단순한 로그인을 넘어 서비스의 보안 안정성과 사용자 편의성을 동시에 확보하기 위한 인증 아키텍처를 구축했습니다.
- **RTR (Refresh Token Rotation)**: Axios Interceptor를 활용하여 Access Token 만료 시(401 에러) 백그라운드에서 자동으로 Refresh Token을 통해 토큰을 재발급받도록 구현했습니다.
- **쿠키 기반 토큰 관리**: XSS 공격 방지를 위해 `withCredentials` 옵션을 활성화하고 HttpOnly 쿠키로 인증 정보를 안전하게 관리합니다.
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

### 2. 지능형 실시간 알림 시스템
사용자의 알림 피로도를 최소화하고 UX를 극대화하기 위해 세밀한 설정이 가능한 알림 인프라를 구축했습니다.
- **Granular Control (알림 세부 설정)**: 워크스페이스 초대, 문서 업로드/삭제, AI 챗봇 답장 등 이벤트별로 '실시간 토스트 알림 노출'과 '알림 목록 저장' 여부를 개별적으로 껐다 켤 수 있도록 DB 스키마와 API를 설계했습니다.
- **Context API 기반 전역 상태 동기화**: `NotificationContext`를 구현하여 서비스 어느 페이지에 있든 실시간으로 알림 벨 아이콘과 토스트 메시지가 동기화되도록 처리했습니다.
- **Contextual Routing**: 특정 알림 클릭 시 해당 이벤트가 발생한 워크스페이스나 채팅 세션으로 즉시 이동하는 딥링크를 구현했습니다.

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

### 3. 접근성을 극대화한 멀티 채널 AI 인터페이스 구축
팀에서 구축한 핵심 RAG 엔진을 사용자가 가장 편리하게 이용할 수 있도록, 목적에 맞게 이원화된 채팅 인터페이스를 설계하고 통합했습니다.
- **랜딩 페이지 메인 챗봇**: 신규 사용자가 서비스 진입 즉시 AI RAG 기능을 직관적으로 체험할 수 있도록 랜딩 페이지 내에 대형 대화형 섹션을 구축하여 서비스 진입 장벽을 낮췄습니다.
- **전역 플로팅 챗봇 위젯**: 사용자가 워크스페이스 등 서비스 내 어느 페이지에 있더라도 작업 흐름을 끊지 않고 즉각적으로 질문할 수 있도록 우측 하단에 플로팅 챗봇 위젯을 별도로 구현했습니다.
- **실시간 스트리밍 및 세션 관리**: AI의 응답을 사용자가 기다리지 않도록 실시간 스트리밍으로 렌더링하고, 세션별 대화 이력을 연동하여 끊김 없는 UX를 제공했습니다.

### 4. 개발 인프라 및 DB 데이터 모델링
초기 개발 환경의 파편화를 막기 위해 Docker 기반의 통합 인프라를 주도적으로 구성했습니다.
- **Docker Compose 통합**: Frontend, Backend, MariaDB, Redis를 하나의 `docker-compose.yml`로 묶어 팀원들의 초기 개발 환경을 구축했습니다..
- **DB 스키마 초기 설계**: 유저, 문서, 채팅, 알림 설정 등의 핵심 도메인에 대한 RDBMS 스키마를 설계했습니다.

---

## 설계 결정

### Axios Interceptor를 이용한 RTR 기법 도입

| 방식 | 설명 | 단점 및 선택 이유 |
|------|------|------|
| **Manual Refresh** | 만료 시 사용자가 다시 로그인 시도 | **단점**: 사용자 경험(UX) 심각한 저하 및 이탈<br>**선택 안 함** |
| **Interceptor RTR** | 401 에러 시 백그라운드에서 자동 토큰 갱신 | **선택 이유**: Access Token의 수명을 짧게 가져가 보안을 강화하면서도, 사용자는 세션 만료를 인지하지 못하게 하여 편의성 동시 확보 |

### 알림 설정 테이블의 물리적 분리 (Notification vs NotificationSetting)

| 테이블 | 저장 데이터 | 비즈니스 목적 |
|------|------|------|
| **Notification** | 실제 발생한 알림 로그, 읽음 여부 | 과거 알림 히스토리 보존 및 추적 |
| **NotificationSetting** | 유저별 이벤트 수신 여부, 토스트 노출 여부 | 유저 개개인의 피로도를 고려한 맞춤형 UX 제공 |

**선택 이유**: 단일 테이블에서 관리할 경우 유저의 설정 변경 내역이 과거 알림 데이터에 영향을 줄 수 있어, 로그 성격의 테이블과 설정 테이블을 완전히 분리하여 데이터 무결성을 유지했습니다.
