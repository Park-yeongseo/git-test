# FastAPI + React OAuth 로그인 가이드 (리다이렉트 방식)

프론트엔드로 리다이렉트하는 방식으로 수정된 가이드입니다.

---

## 📋 목차

1. [프로젝트 준비](#1-프로젝트-준비)
2. [OAuth 키 발급받기](#2-oauth-키-발급받기)
3. [백엔드 코드 작성](#3-백엔드-코드-작성)
4. [프론트엔드 코드 작성](#4-프론트엔드-코드-작성)
5. [실행 및 테스트](#5-실행-및-테스트)

---

## 1. 프로젝트 준비

### 1-1. 백엔드 폴더 구조

```bash
mkdir my-oauth-backend
cd my-oauth-backend

python -m venv venv
# Windows: venv\Scripts\activate
# Mac/Linux: source venv/bin/activate

pip install fastapi uvicorn[standard] python-jose[cryptography] authlib httpx sqlalchemy python-multipart itsdangerous python-dotenv
```

**폴더 구조:**
```
my-oauth-backend/
├── .env
├── main.py
├── config.py
├── oauth_config.py
├── database.py
├── models.py
├── crud.py
└── schemas.py
```

### 1-2. 프론트엔드 폴더 구조

```bash
npm create vite@latest my-oauth-frontend -- --template react
cd my-oauth-frontend
npm install react-router-dom axios
```

**폴더 구조:**
```
my-oauth-frontend/
├── src/
│   ├── components/
│   │   ├── Login.jsx
│   │   ├── Callback.jsx
│   │   ├── Profile.jsx
│   │   └── ProtectedRoute.jsx
│   ├── services/
│   │   └── api.js
│   ├── utils/
│   │   └── auth.js
│   ├── App.jsx
│   └── main.jsx
└── package.json
```

---

## 2. OAuth 키 발급받기

### 🔵 Google OAuth

1. https://console.cloud.google.com/ 접속
2. 프로젝트 생성
3. OAuth 동의 화면 설정 (외부 선택)
4. 사용자 인증 정보 만들기 → OAuth 클라이언트 ID
5. 승인된 리디렉션 URI: `http://localhost:8000/auth/google`

### 🟡 Kakao OAuth

1. https://developers.kakao.com/ 접속
2. 애플리케이션 추가
3. 플랫폼 설정 → Web: `http://localhost:8000`
4. 카카오 로그인 활성화
5. Redirect URI: `http://localhost:8000/auth/kakao`

### 🟣 GitHub OAuth

1. GitHub Settings → Developer settings → OAuth Apps
2. New OAuth App
3. Homepage URL: `http://localhost:8000`
4. Authorization callback URL: `http://localhost:8000/auth/github`

---

## 3. 백엔드 코드 작성

### 3-1. .env

```env
# OAuth 키
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

KAKAO_CLIENT_ID=your-kakao-client-id
KAKAO_CLIENT_SECRET=your-kakao-client-secret

GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret

# JWT Secret
SECRET_KEY=your-super-secret-key-change-this

# 프론트엔드 URL
FRONTEND_URL=http://localhost:5173
```

### 3-2. models.py

```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True, nullable=True)
    name = Column(String)
    profile_image = Column(String, nullable=True)
    provider = Column(String)
    provider_id = Column(String, nullable=True)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    last_login = Column(DateTime, default=datetime.utcnow)
```

### 3-3. database.py

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import Base

DATABASE_URL = "sqlite:///./app.db"

engine = create_engine(
    DATABASE_URL, 
    connect_args={"check_same_thread": False}
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base.metadata.create_all(bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 3-4. config.py

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
import secrets
import os

SECRET_KEY = os.getenv("SECRET_KEY", "fallback-secret-key")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 24 * 7

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None

def generate_unique_user_id():
    return secrets.token_urlsafe(16)
```

### 3-5. crud.py

```python
from sqlalchemy.orm import Session
from models import User
from datetime import datetime
from config import generate_unique_user_id

def get_user_by_id(db: Session, user_id: str):
    return db.query(User).filter(User.user_id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

def get_user_by_provider(db: Session, provider: str, provider_id: str):
    return db.query(User).filter(
        User.provider == provider,
        User.provider_id == provider_id
    ).first()

def create_user(db: Session, email: str = None, name: str = None, 
                provider: str = "local", provider_id: str = None, 
                profile_image: str = None):
    user_id = generate_unique_user_id()
    db_user = User(
        user_id=user_id,
        email=email,
        name=name,
        provider=provider,
        provider_id=provider_id,
        profile_image=profile_image
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def update_last_login(db: Session, user_id: str):
    user = get_user_by_id(db, user_id)
    if user:
        user.last_login = datetime.utcnow()
        db.commit()
```

### 3-6. schemas.py

```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class UserResponse(BaseModel):
    user_id: str
    email: Optional[str]
    name: str
    profile_image: Optional[str]
    provider: str
    created_at: datetime
    last_login: datetime
    
    class Config:
        from_attributes = True

class TokenResponse(BaseModel):
    access_token: str
    token_type: str
    user: UserResponse
```

### 3-7. oauth_config.py

```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

config = Config('.env')
oauth = OAuth(config)

oauth.register(
    name='google',
    client_id=config('GOOGLE_CLIENT_ID'),
    client_secret=config('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

oauth.register(
    name='kakao',
    client_id=config('KAKAO_CLIENT_ID'),
    client_secret=config('KAKAO_CLIENT_SECRET', default=None),
    authorize_url='https://kauth.kakao.com/oauth/authorize',
    access_token_url='https://kauth.kakao.com/oauth/token',
    client_kwargs={'scope': 'profile_nickname profile_image account_email'}
)

oauth.register(
    name='github',
    client_id=config('GITHUB_CLIENT_ID'),
    client_secret=config('GITHUB_CLIENT_SECRET'),
    authorize_url='https://github.com/login/oauth/authorize',
    access_token_url='https://github.com/login/oauth/access_token',
    api_base_url='https://api.github.com/',
    client_kwargs={'scope': 'user:email'}
)
```

### 3-8. main.py

```python
from fastapi import FastAPI, Depends, HTTPException, status, Request
from fastapi.security import OAuth2PasswordBearer
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import RedirectResponse
from sqlalchemy.orm import Session
import httpx
import os
from dotenv import load_dotenv

from oauth_config import oauth
from config import create_access_token, verify_token
from database import get_db
from models import User
from schemas import UserResponse
import crud

load_dotenv()

app = FastAPI(
    title="OAuth 간편 로그인 API",
    description="Google, Kakao, GitHub 로그인 지원",
    version="1.0.0"
)

# 환경변수에서 프론트엔드 URL 가져오기
FRONTEND_URL = os.getenv("FRONTEND_URL", "http://localhost:5173")

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=[FRONTEND_URL],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
):
    payload = verify_token(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="유효하지 않은 토큰입니다"
        )
    
    user = crud.get_user_by_id(db, payload.get("sub"))
    if not user:
        raise HTTPException(status_code=404, detail="사용자를 찾을 수 없습니다")
    
    return user

# ==================== Google 로그인 ====================

@app.get('/login/google')
async def login_google(request: Request):
    redirect_uri = request.url_for('auth_google')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get('/auth/google')
async def auth_google(request: Request, db: Session = Depends(get_db)):
    try:
        token = await oauth.google.authorize_access_token(request)
        user_info = token.get('userinfo')
        
        user = crud.get_user_by_provider(db, "google", user_info['sub'])
        
        if not user:
            user = crud.get_user_by_email(db, user_info['email'])
            
            if not user:
                user = crud.create_user(
                    db=db,
                    email=user_info['email'],
                    name=user_info['name'],
                    provider="google",
                    provider_id=user_info['sub'],
                    profile_image=user_info.get('picture')
                )
        
        crud.update_last_login(db, user.user_id)
        
        access_token = create_access_token(
            data={
                "sub": user.user_id,
                "email": user.email,
                "name": user.name,
                "provider": "google"
            }
        )
        
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?token={access_token}"
        )
        
    except Exception as e:
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?error={str(e)}"
        )

# ==================== Kakao 로그인 ====================

@app.get('/login/kakao')
async def login_kakao(request: Request):
    redirect_uri = request.url_for('auth_kakao')
    return await oauth.kakao.authorize_redirect(request, redirect_uri)

@app.get('/auth/kakao')
async def auth_kakao(request: Request, db: Session = Depends(get_db)):
    try:
        token = await oauth.kakao.authorize_access_token(request)
        
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                'https://kapi.kakao.com/v2/user/me',
                headers={'Authorization': f"Bearer {token['access_token']}"}
            )
            kakao_user = resp.json()
        
        provider_id = str(kakao_user['id'])
        email = kakao_user.get('kakao_account', {}).get('email')
        name = kakao_user.get('properties', {}).get('nickname', 'Kakao User')
        profile_image = kakao_user.get('properties', {}).get('profile_image')
        
        user = crud.get_user_by_provider(db, "kakao", provider_id)
        
        if not user:
            user = crud.create_user(
                db=db,
                email=email,
                name=name,
                provider="kakao",
                provider_id=provider_id,
                profile_image=profile_image
            )
        
        crud.update_last_login(db, user.user_id)
        
        access_token = create_access_token(
            data={
                "sub": user.user_id,
                "email": user.email,
                "name": user.name,
                "provider": "kakao"
            }
        )
        
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?token={access_token}"
        )
        
    except Exception as e:
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?error={str(e)}"
        )

# ==================== GitHub 로그인 ====================

@app.get('/login/github')
async def login_github(request: Request):
    redirect_uri = request.url_for('auth_github')
    return await oauth.github.authorize_redirect(request, redirect_uri)

@app.get('/auth/github')
async def auth_github(request: Request, db: Session = Depends(get_db)):
    try:
        token = await oauth.github.authorize_access_token(request)
        
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                'https://api.github.com/user',
                headers={'Authorization': f"Bearer {token['access_token']}"}
            )
            github_user = resp.json()
            
            email_resp = await client.get(
                'https://api.github.com/user/emails',
                headers={'Authorization': f"Bearer {token['access_token']}"}
            )
            emails = email_resp.json()
            primary_email = next(
                (e['email'] for e in emails if e['primary']), 
                None
            )
        
        provider_id = str(github_user['id'])
        name = github_user.get('name') or github_user.get('login')
        profile_image = github_user.get('avatar_url')
        
        user = crud.get_user_by_provider(db, "github", provider_id)
        
        if not user:
            user = crud.create_user(
                db=db,
                email=primary_email,
                name=name,
                provider="github",
                provider_id=provider_id,
                profile_image=profile_image
            )
        
        crud.update_last_login(db, user.user_id)
        
        access_token = create_access_token(
            data={
                "sub": user.user_id,
                "email": user.email,
                "name": user.name,
                "provider": "github"
            }
        )
        
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?token={access_token}"
        )
        
    except Exception as e:
        return RedirectResponse(
            url=f"{FRONTEND_URL}/callback?error={str(e)}"
        )

# ==================== API ====================

@app.get("/me", response_model=UserResponse)
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user

@app.get("/users", response_model=list[UserResponse])
async def list_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
```

---

## 4. 프론트엔드 코드 작성

### 4-1. src/utils/auth.js

```javascript
export const authUtils = {
  setToken: (token) => {
    localStorage.setItem('access_token', token);
  },

  getToken: () => {
    return localStorage.getItem('access_token');
  },

  removeToken: () => {
    localStorage.removeItem('access_token');
  },

  isAuthenticated: () => {
    return !!localStorage.getItem('access_token');
  }
};
```

### 4-2. src/services/api.js

```javascript
import axios from 'axios';
import { authUtils } from '../utils/auth';

const API_URL = 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
});

api.interceptors.request.use(
  (config) => {
    const token = authUtils.getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authUtils.removeToken();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export const authAPI = {
  getMe: async () => {
    const response = await api.get('/me');
    return response.data;
  },

  getUsers: async () => {
    const response = await api.get('/users');
    return response.data;
  }
};

export default api;
```

### 4-3. src/components/Login.jsx

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';
import { authUtils } from '../utils/auth';

const Login = () => {
  if (authUtils.isAuthenticated()) {
    return <Navigate to="/profile" />;
  }

  const handleGoogleLogin = () => {
    window.location.href = 'http://localhost:8000/login/google';
  };

  const handleKakaoLogin = () => {
    window.location.href = 'http://localhost:8000/login/kakao';
  };

  const handleGithubLogin = () => {
    window.location.href = 'http://localhost:8000/login/github';
  };

  return (
    <div style={styles.container}>
      <div style={styles.box}>
        <h1>🔐 로그인</h1>
        <p>원하는 서비스로 간편하게 로그인하세요</p>

        <div style={styles.buttons}>
          <button style={{...styles.btn, ...styles.google}} onClick={handleGoogleLogin}>
            🔵 Google로 로그인
          </button>

          <button style={{...styles.btn, ...styles.kakao}} onClick={handleKakaoLogin}>
            💬 Kakao로 로그인
          </button>

          <button style={{...styles.btn, ...styles.github}} onClick={handleGithubLogin}>
            🐙 GitHub로 로그인
          </button>
        </div>
      </div>
    </div>
  );
};

const styles = {
  container: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    minHeight: '100vh',
    background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
  },
  box: {
    background: 'white',
    padding: '40px',
    borderRadius: '20px',
    boxShadow: '0 10px 40px rgba(0, 0, 0, 0.1)',
    maxWidth: '400px',
    width: '90%',
    textAlign: 'center',
  },
  buttons: {
    display: 'flex',
    flexDirection: 'column',
    gap: '15px',
    marginTop: '20px',
  },
  btn: {
    padding: '15px 30px',
    fontSize: '16px',
    fontWeight: '600',
    border: 'none',
    borderRadius: '10px',
    cursor: 'pointer',
    transition: 'all 0.3s ease',
  },
  google: {
    backgroundColor: '#4285f4',
    color: 'white',
  },
  kakao: {
    backgroundColor: '#fee500',
    color: '#000',
  },
  github: {
    backgroundColor: '#333',
    color: 'white',
  }
};

export default Login;
```

### 4-4. src/components/Callback.jsx

```javascript
import React, { useEffect, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import { authUtils } from '../utils/auth';

const Callback = () => {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const [status, setStatus] = useState('처리 중...');

  useEffect(() => {
    const token = searchParams.get('token');
    const error = searchParams.get('error');

    if (token) {
      authUtils.setToken(token);
      setStatus('✅ 로그인 성공! 이동 중...');
      
      setTimeout(() => {
        navigate('/profile');
      }, 1000);
    } else if (error) {
      setStatus(`❌ 로그인 실패: ${error}`);
      
      setTimeout(() => {
        navigate('/login');
      }, 3000);
    } else {
      setStatus('❌ 잘못된 접근입니다');
      setTimeout(() => {
        navigate('/login');
      }, 2000);
    }
  }, [searchParams, navigate]);

  return (
    <div style={{
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      height: '100vh',
      fontSize: '24px'
    }}>
      {status}
    </div>
  );
};

export default Callback;
```

### 4-5. src/components/Profile.jsx

```javascript
import React, { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { authAPI } from '../services/api';
import { authUtils } from '../utils/auth';

const Profile = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const navigate = useNavigate();

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      setLoading(true);
      const userData = await authAPI.getMe();
      setUser(userData);
    } catch (err) {
      setError('사용자 정보를 불러올 수 없습니다');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  const handleLogout = () => {
    authUtils.removeToken();
    navigate('/login');
  };

  if (loading) {
    return <div style={styles.center}>로딩 중...</div>;
  }

  if (error) {
    return <div style={styles.center}>{error}</div>;
  }

  return (
    <div style={styles.container}>
      <div style={styles.card}>
        <h1>👤 내 프로필</h1>
        
        {user.profile_image && (
          <img src={user.profile_image} alt="프로필" style={styles.image} />
        )}

        <div style={styles.info}>
          <div style={styles.row}>
            <span style={styles.label}>이름:</span>
            <span style={styles.value}>{user.name}</span>
          </div>

          <div style={styles.row}>
            <span style={styles.label}>이메일:</span>
            <span style={styles.value}>{user.email || '없음'}</span>
          </div>

          <div style={styles.row}>
            <span style={styles.label}>로그인 방법:</span>
            <span style={{...styles.value, ...styles.provider}}>{user.provider}</span>
          </div>

          <div style={styles.row}>
            <span style={styles.label}>가입일:</span>
            <span style={styles.value}>
              {new Date(user.created_at).toLocaleDateString('ko-KR')}
            </span>
          </div>
        </div>

        <button style={styles.logoutBtn} onClick={handleLogout}>
          로그아웃
        </button>
      </div>
    </div>
  );
};

const styles = {
  container: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    minHeight: '100vh',
    background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    padding: '20px',
  },
  card: {
    background: 'white',
    padding: '40px',
    borderRadius: '20px',
    boxShadow: '0 10px 40px rgba(0, 0, 0, 0.1)',
    maxWidth: '500px',
    width: '100%',
  },
  image: {
    width: '150px',
    height: '150px',
    borderRadius: '50%',
    objectFit: 'cover',
    display: 'block',
    margin: '0 auto 30px',
    border: '4px solid #667eea',
  },
  info: {
    display: 'flex',
    flexDirection: 'column',
    gap: '15px',
    marginBottom: '30px',
  },
  row: {
    display: 'flex',
    justifyContent: 'space-between',
    padding: '10px',
    background: '#f8f9fa',
    borderRadius: '8px',
  },
  label: {
    fontWeight: '600',
    color: '#666',
  },
  value: {
    color: '#333',
  },
  provider: {
    textTransform: 'capitalize',
    background: '#667eea',
    color: 'white',
    padding: '2px 10px',
    borderRadius: '5px',
    fontSize: '14px',
  },
  logoutBtn: {
    width: '100%',
    padding: '15px',
    background: '#dc3545',
    color: 'white',
    border: 'none',
    borderRadius: '10px',
    fontSize: '16px',
    fontWeight: '600',
    cursor: 'pointer',
  },
  center: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    height: '100vh',
    fontSize: '24px',
  }
};

export default Profile;
```

### 4-6. src/components/ProtectedRoute.jsx

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';
import { authUtils } from '../utils/auth';

const ProtectedRoute = ({ children }) => {
  if (!authUtils.isAuthenticated()) {
    return <Navigate to="/login" />;
  }

  return children;
};

export default ProtectedRoute;
```

### 4-7. src/App.jsx

```javascript
import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import Login from './components/Login';
import Callback from './components/Callback';
import Profile from './components/Profile';
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Navigate to="/login" />} />
        <Route path="/login" element={<Login />} />
        <Route path="/callback" element={<Callback />} />
        <Route 
          path="/profile" 
          element={
            <ProtectedRoute>
              <Profile />
            </ProtectedRoute>
          } 
        />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

---

## 5. 실행 및 테스트

### 5-1. 백엔드 실행

```bash
cd my-oauth-backend
uvicorn main:app --reload
```

### 5-2. 프론트엔드 실행

```bash
cd my-oauth-frontend
npm run dev
```

### 5-3. 테스트

1. 브라우저에서 `http://localhost:5173` 접속
2. 원하는 OAuth 버튼 클릭
3. OAuth 제공자에서 로그인
4. 자동으로 프로필 페이지로 이동
5. 사용자 정보 확인

---

## 6. 주의사항

### 개발 환경
- 백엔드: `http://localhost:8000`
- 프론트엔드: `http://localhost:5173`
- `.env` 파일에 `FRONTEND_URL=http://localhost:5173` 설정 필수

### 배포 시
- `.env`의 `FRONTEND_URL`을 실제 도메인으로 변경
- CORS 설정 업데이트
- OAuth 제공자에 프로덕션 Redirect URI 추가
