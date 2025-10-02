# 🔐 Authentication & Authorization API

This document covers all authentication and authorization mechanisms in AI ETL Assistant, including JWT tokens, role-based access control (RBAC), and security flows.

## Authentication Overview

```mermaid
sequenceDiagram
    participant Client
    participant Frontend
    participant Backend
    participant Database
    participant LLM Gateway

    Client->>Frontend: Login Request
    Frontend->>Backend: POST /api/v1/auth/login
    Backend->>Database: Validate Credentials
    Database-->>Backend: User Data + Roles
    Backend->>Backend: Generate JWT Token
    Backend-->>Frontend: JWT + Refresh Token
    Frontend-->>Client: Authentication Success

    Note over Frontend, Backend: Subsequent API Calls
    Client->>Frontend: API Request
    Frontend->>Backend: Request + JWT Header
    Backend->>Backend: Validate JWT + Check Permissions
    Backend->>LLM Gateway: Authorized Request
    Backend-->>Frontend: API Response
```

## Authentication Endpoints

### POST /api/v1/auth/login

Authenticate user and receive JWT tokens.

**Request:**
```json
{
  "email": "user@company.com",
  "password": "secure_password",
  "remember_me": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "id": "uuid-here",
      "email": "user@company.com",
      "full_name": "John Doe",
      "role": "data_engineer",
      "permissions": [
        "pipelines:read",
        "pipelines:create",
        "connectors:read"
      ],
      "profile": {
        "avatar_url": "https://...",
        "timezone": "UTC",
        "language": "en"
      }
    }
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "AUTH_INVALID_CREDENTIALS",
    "message": "Invalid email or password",
    "details": {
      "attempts_remaining": 3,
      "lockout_time": null
    }
  }
}
```

**Implementation Example:**
```python
from fastapi import APIRouter, HTTPException, Depends
from fastapi.security import HTTPBearer
from backend.auth import AuthService
from backend.schemas.auth import LoginRequest, LoginResponse

router = APIRouter()
auth_service = AuthService()
security = HTTPBearer()

@router.post("/login", response_model=LoginResponse)
async def login(request: LoginRequest):
    try:
        result = await auth_service.authenticate_user(
            email=request.email,
            password=request.password,
            remember_me=request.remember_me
        )
        return LoginResponse(success=True, data=result)
    except AuthService.InvalidCredentialsError as e:
        raise HTTPException(
            status_code=401,
            detail={
                "code": "AUTH_INVALID_CREDENTIALS",
                "message": str(e),
                "details": e.details
            }
        )
```

### POST /api/v1/auth/refresh

Refresh access token using refresh token.

**Request:**
```json
{
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "expires_in": 3600
  }
}
```

### POST /api/v1/auth/logout

Logout user and invalidate tokens.

**Request Headers:**
```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

**Response:**
```json
{
  "success": true,
  "message": "Successfully logged out"
}
```

### GET /api/v1/auth/me

Get current user information.

**Request Headers:**
```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "email": "user@company.com",
    "full_name": "John Doe",
    "role": "data_engineer",
    "permissions": ["pipelines:read", "pipelines:create"],
    "profile": {
      "avatar_url": "https://...",
      "timezone": "UTC",
      "language": "en"
    },
    "last_login": "2024-01-26T10:30:00Z",
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

## User Registration

### POST /api/v1/auth/register

Register new user account.

**Request:**
```json
{
  "email": "newuser@company.com",
  "password": "secure_password",
  "full_name": "Jane Smith",
  "organization": "TechCorp",
  "role_request": "data_analyst",
  "terms_accepted": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user_id": "new-uuid-here",
    "email": "newuser@company.com",
    "status": "pending_verification",
    "verification_required": true,
    "message": "Please check your email for verification link"
  }
}
```

### POST /api/v1/auth/verify-email

Verify email address after registration.

**Request:**
```json
{
  "token": "email-verification-token-here"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Email verified successfully",
  "data": {
    "user_id": "uuid-here",
    "status": "active",
    "auto_login": {
      "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
      "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
    }
  }
}
```

## Password Management

### POST /api/v1/auth/forgot-password

Request password reset.

**Request:**
```json
{
  "email": "user@company.com"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Password reset link sent to your email"
}
```

### POST /api/v1/auth/reset-password

Reset password using token from email.

**Request:**
```json
{
  "token": "password-reset-token-here",
  "new_password": "new_secure_password",
  "confirm_password": "new_secure_password"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Password reset successfully"
}
```

### PUT /api/v1/auth/change-password

Change password for authenticated user.

**Request Headers:**
```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

**Request:**
```json
{
  "current_password": "old_password",
  "new_password": "new_secure_password",
  "confirm_password": "new_secure_password"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

## Role-Based Access Control (RBAC)

### User Roles

```python
class UserRole(str, Enum):
    ADMIN = "admin"                    # Full system access
    ARCHITECT = "data_architect"       # Architecture decisions, high-level design
    ENGINEER = "data_engineer"         # Pipeline development, deployment
    ANALYST = "data_analyst"          # Pipeline creation, data analysis
    VIEWER = "viewer"                 # Read-only access
```

### Role Permissions Matrix

| Resource | Admin | Architect | Engineer | Analyst | Viewer |
|----------|-------|-----------|----------|---------|--------|
| **Users** | | | | | |
| Create/Delete Users | ✅ | ❌ | ❌ | ❌ | ❌ |
| Modify Roles | ✅ | ❌ | ❌ | ❌ | ❌ |
| View Users | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Pipelines** | | | | | |
| Create Pipeline | ✅ | ✅ | ✅ | ✅ | ❌ |
| Deploy Pipeline | ✅ | ✅ | ✅ | ❌ | ❌ |
| Delete Pipeline | ✅ | ✅ | Owner Only | Owner Only | ❌ |
| View Pipeline | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Connectors** | | | | | |
| Create Connector | ✅ | ✅ | ✅ | ❌ | ❌ |
| Configure Connector | ✅ | ✅ | ✅ | ✅ | ❌ |
| View Connector | ✅ | ✅ | ✅ | ✅ | ✅ |
| **System** | | | | | |
| System Settings | ✅ | ❌ | ❌ | ❌ | ❌ |
| Monitoring | ✅ | ✅ | ✅ | ❌ | ❌ |
| Logs Access | ✅ | ✅ | ✅ | ❌ | ❌ |

### Permission Checking

```python
# backend/auth/permissions.py
from enum import Enum
from typing import List, Set

class Permission(str, Enum):
    # User management
    USERS_CREATE = "users:create"
    USERS_READ = "users:read"
    USERS_UPDATE = "users:update"
    USERS_DELETE = "users:delete"

    # Pipeline management
    PIPELINES_CREATE = "pipelines:create"
    PIPELINES_READ = "pipelines:read"
    PIPELINES_UPDATE = "pipelines:update"
    PIPELINES_DELETE = "pipelines:delete"
    PIPELINES_DEPLOY = "pipelines:deploy"
    PIPELINES_EXECUTE = "pipelines:execute"

    # Connector management
    CONNECTORS_CREATE = "connectors:create"
    CONNECTORS_READ = "connectors:read"
    CONNECTORS_UPDATE = "connectors:update"
    CONNECTORS_DELETE = "connectors:delete"

    # System administration
    SYSTEM_SETTINGS = "system:settings"
    SYSTEM_MONITORING = "system:monitoring"
    SYSTEM_LOGS = "system:logs"

ROLE_PERMISSIONS = {
    UserRole.ADMIN: [
        Permission.USERS_CREATE, Permission.USERS_READ, Permission.USERS_UPDATE, Permission.USERS_DELETE,
        Permission.PIPELINES_CREATE, Permission.PIPELINES_READ, Permission.PIPELINES_UPDATE,
        Permission.PIPELINES_DELETE, Permission.PIPELINES_DEPLOY, Permission.PIPELINES_EXECUTE,
        Permission.CONNECTORS_CREATE, Permission.CONNECTORS_READ, Permission.CONNECTORS_UPDATE,
        Permission.CONNECTORS_DELETE,
        Permission.SYSTEM_SETTINGS, Permission.SYSTEM_MONITORING, Permission.SYSTEM_LOGS
    ],
    UserRole.ARCHITECT: [
        Permission.USERS_READ,
        Permission.PIPELINES_CREATE, Permission.PIPELINES_READ, Permission.PIPELINES_UPDATE,
        Permission.PIPELINES_DELETE, Permission.PIPELINES_DEPLOY, Permission.PIPELINES_EXECUTE,
        Permission.CONNECTORS_CREATE, Permission.CONNECTORS_READ, Permission.CONNECTORS_UPDATE,
        Permission.SYSTEM_MONITORING, Permission.SYSTEM_LOGS
    ],
    UserRole.ENGINEER: [
        Permission.USERS_READ,
        Permission.PIPELINES_CREATE, Permission.PIPELINES_READ, Permission.PIPELINES_UPDATE,
        Permission.PIPELINES_DELETE, Permission.PIPELINES_DEPLOY, Permission.PIPELINES_EXECUTE,
        Permission.CONNECTORS_CREATE, Permission.CONNECTORS_READ, Permission.CONNECTORS_UPDATE,
        Permission.SYSTEM_MONITORING, Permission.SYSTEM_LOGS
    ],
    UserRole.ANALYST: [
        Permission.PIPELINES_CREATE, Permission.PIPELINES_READ, Permission.PIPELINES_UPDATE,
        Permission.PIPELINES_EXECUTE,
        Permission.CONNECTORS_READ, Permission.CONNECTORS_UPDATE
    ],
    UserRole.VIEWER: [
        Permission.PIPELINES_READ,
        Permission.CONNECTORS_READ
    ]
}

def has_permission(user_role: UserRole, permission: Permission) -> bool:
    """Check if user role has specific permission"""
    return permission in ROLE_PERMISSIONS.get(user_role, [])
```

## JWT Token Structure

### Access Token Payload

```json
{
  "sub": "user-uuid-here",
  "email": "user@company.com",
  "role": "data_engineer",
  "permissions": [
    "pipelines:read",
    "pipelines:create",
    "connectors:read"
  ],
  "org_id": "organization-uuid",
  "iat": 1706275200,
  "exp": 1706278800,
  "type": "access"
}
```

### Refresh Token Payload

```json
{
  "sub": "user-uuid-here",
  "jti": "refresh-token-id",
  "iat": 1706275200,
  "exp": 1706361600,
  "type": "refresh"
}
```

### Token Configuration

```python
# backend/config/auth.py
import os
from datetime import timedelta

class AuthConfig:
    SECRET_KEY = os.getenv("JWT_SECRET_KEY", "your-secret-key")
    ALGORITHM = "HS256"

    ACCESS_TOKEN_EXPIRE_MINUTES = int(os.getenv("JWT_ACCESS_TOKEN_EXPIRE_MINUTES", "30"))
    REFRESH_TOKEN_EXPIRE_DAYS = int(os.getenv("JWT_REFRESH_TOKEN_EXPIRE_DAYS", "7"))

    PASSWORD_MIN_LENGTH = 8
    PASSWORD_REQUIRE_SPECIAL = True
    PASSWORD_REQUIRE_NUMERIC = True
    PASSWORD_REQUIRE_UPPERCASE = True

    MAX_LOGIN_ATTEMPTS = 5
    LOCKOUT_DURATION_MINUTES = 15
```

## Authentication Middleware

### FastAPI Dependency

```python
# backend/auth/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from backend.auth.service import AuthService
from backend.models.user import User
from backend.auth.permissions import Permission

security = HTTPBearer()
auth_service = AuthService()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Get current authenticated user"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            credentials.credentials,
            auth_service.secret_key,
            algorithms=[auth_service.algorithm]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception

    except JWTError:
        raise credentials_exception

    user = await auth_service.get_user(user_id)
    if user is None:
        raise credentials_exception

    return user

def require_permission(permission: Permission):
    """Dependency factory for permission checking"""
    async def permission_checker(
        current_user: User = Depends(get_current_user)
    ) -> User:
        if not auth_service.has_permission(current_user.role, permission):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permission denied: {permission}"
            )
        return current_user

    return permission_checker

# Usage in routes
@router.post("/pipelines/")
async def create_pipeline(
    pipeline_data: PipelineCreate,
    current_user: User = Depends(require_permission(Permission.PIPELINES_CREATE))
):
    # Only users with pipelines:create permission can access
    return await pipeline_service.create_pipeline(pipeline_data, current_user)
```

## OAuth 2.0 Integration

### Google OAuth Configuration

```python
# backend/auth/oauth.py
from authlib.integrations.fastapi_oauth2 import OAuth2AuthorizeCallback
from authlib.integrations.httpx_oauth2 import AsyncOAuth2Client

class GoogleOAuthConfig:
    CLIENT_ID = os.getenv("GOOGLE_CLIENT_ID")
    CLIENT_SECRET = os.getenv("GOOGLE_CLIENT_SECRET")
    REDIRECT_URI = os.getenv("GOOGLE_REDIRECT_URI", "http://localhost:3000/auth/callback/google")
    SCOPES = ["openid", "email", "profile"]

google_oauth = AsyncOAuth2Client(
    client_id=GoogleOAuthConfig.CLIENT_ID,
    client_secret=GoogleOAuthConfig.CLIENT_SECRET,
    authorize_url="https://accounts.google.com/o/oauth2/auth",
    token_url="https://oauth2.googleapis.com/token"
)
```

### OAuth Endpoints

```python
@router.get("/auth/google")
async def google_auth():
    """Redirect to Google OAuth"""
    redirect_uri = GoogleOAuthConfig.REDIRECT_URI
    authorization_url = await google_oauth.create_authorization_url(
        redirect_uri,
        scope=" ".join(GoogleOAuthConfig.SCOPES)
    )
    return {"authorization_url": authorization_url}

@router.post("/auth/google/callback")
async def google_callback(code: str, state: str = None):
    """Handle Google OAuth callback"""
    try:
        token = await google_oauth.fetch_token(
            GoogleOAuthConfig.REDIRECT_URI,
            authorization_response=request.url
        )

        # Get user info from Google
        resp = await google_oauth.get(
            "https://www.googleapis.com/oauth2/v2/userinfo",
            token=token
        )
        user_info = resp.json()

        # Create or update user
        user = await auth_service.create_or_update_oauth_user(
            provider="google",
            provider_id=user_info["id"],
            email=user_info["email"],
            full_name=user_info["name"],
            avatar_url=user_info.get("picture")
        )

        # Generate JWT tokens
        tokens = await auth_service.create_tokens(user)

        return {
            "success": True,
            "data": {
                "access_token": tokens.access_token,
                "refresh_token": tokens.refresh_token,
                "user": user.dict()
            }
        }

    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

## Security Features

### Rate Limiting

```python
# backend/auth/rate_limiter.py
from fastapi import Request, HTTPException
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)

@limiter.limit("5/minute")
@router.post("/auth/login")
async def login(request: Request, login_data: LoginRequest):
    # Login endpoint with rate limiting
    pass

@limiter.limit("3/minute")
@router.post("/auth/forgot-password")
async def forgot_password(request: Request, email: str):
    # Password reset with stricter rate limiting
    pass
```

### Account Lockout

```python
# backend/auth/lockout.py
from datetime import datetime, timedelta
from backend.models.user import User, LoginAttempt

class AccountLockoutService:
    MAX_ATTEMPTS = 5
    LOCKOUT_DURATION = timedelta(minutes=15)

    async def record_failed_attempt(self, user: User, ip_address: str):
        """Record failed login attempt"""
        attempt = LoginAttempt(
            user_id=user.id,
            ip_address=ip_address,
            success=False,
            timestamp=datetime.utcnow()
        )
        await self.db.save(attempt)

        # Check if account should be locked
        recent_attempts = await self.get_recent_failed_attempts(user.id)
        if len(recent_attempts) >= self.MAX_ATTEMPTS:
            user.locked_until = datetime.utcnow() + self.LOCKOUT_DURATION
            await self.db.save(user)

    async def is_account_locked(self, user: User) -> bool:
        """Check if account is currently locked"""
        if user.locked_until:
            return datetime.utcnow() < user.locked_until
        return False
```

### Password Security

```python
# backend/auth/password.py
import re
import bcrypt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class PasswordValidator:
    MIN_LENGTH = 8
    REQUIRE_UPPERCASE = True
    REQUIRE_LOWERCASE = True
    REQUIRE_DIGIT = True
    REQUIRE_SPECIAL = True

    @classmethod
    def validate_password(cls, password: str) -> tuple[bool, list[str]]:
        """Validate password against security requirements"""
        errors = []

        if len(password) < cls.MIN_LENGTH:
            errors.append(f"Password must be at least {cls.MIN_LENGTH} characters")

        if cls.REQUIRE_UPPERCASE and not re.search(r"[A-Z]", password):
            errors.append("Password must contain uppercase letter")

        if cls.REQUIRE_LOWERCASE and not re.search(r"[a-z]", password):
            errors.append("Password must contain lowercase letter")

        if cls.REQUIRE_DIGIT and not re.search(r"\d", password):
            errors.append("Password must contain digit")

        if cls.REQUIRE_SPECIAL and not re.search(r"[!@#$%^&*(),.?\":{}|<>]", password):
            errors.append("Password must contain special character")

        return len(errors) == 0, errors

def hash_password(password: str) -> str:
    """Hash password using bcrypt"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)
```

## API Key Authentication

### API Key Management

```python
# backend/models/api_key.py
from sqlalchemy import Column, String, DateTime, Boolean, Text
from sqlalchemy.dialects.postgresql import UUID
import uuid

class APIKey(Base):
    __tablename__ = "api_keys"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(100), nullable=False)
    key_hash = Column(String(255), nullable=False, unique=True)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    permissions = Column(ARRAY(String), default=[])

    is_active = Column(Boolean, default=True)
    last_used = Column(DateTime)
    expires_at = Column(DateTime)

    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

### API Key Endpoints

```python
@router.post("/api-keys/")
async def create_api_key(
    key_data: APIKeyCreate,
    current_user: User = Depends(get_current_user)
):
    """Create new API key"""
    api_key = await auth_service.create_api_key(
        user_id=current_user.id,
        name=key_data.name,
        permissions=key_data.permissions,
        expires_in_days=key_data.expires_in_days
    )

    return {
        "success": True,
        "data": {
            "id": api_key.id,
            "name": api_key.name,
            "key": api_key.plain_key,  # Only returned once
            "permissions": api_key.permissions,
            "expires_at": api_key.expires_at
        }
    }

@router.get("/api-keys/")
async def list_api_keys(
    current_user: User = Depends(get_current_user)
):
    """List user's API keys"""
    keys = await auth_service.get_user_api_keys(current_user.id)
    return {
        "success": True,
        "data": [
            {
                "id": key.id,
                "name": key.name,
                "permissions": key.permissions,
                "is_active": key.is_active,
                "last_used": key.last_used,
                "expires_at": key.expires_at,
                "created_at": key.created_at
            }
            for key in keys
        ]
    }
```

### API Key Authentication

```python
# backend/auth/api_key.py
from fastapi import HTTPException, status, Depends
from fastapi.security.api_key import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def authenticate_api_key(
    api_key: str = Depends(api_key_header)
) -> User:
    """Authenticate using API key"""
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key required"
        )

    # Validate API key
    key_record = await auth_service.validate_api_key(api_key)
    if not key_record:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key"
        )

    # Update last used timestamp
    await auth_service.update_api_key_usage(key_record.id)

    # Return associated user
    return await auth_service.get_user(key_record.user_id)
```

## Frontend Integration

### React Authentication Hook

```typescript
// frontend/hooks/useAuth.ts
import { useState, useEffect, createContext, useContext } from 'react'
import { User, LoginCredentials, AuthTokens } from '../types/auth'

interface AuthContextType {
  user: User | null
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => void
  refreshToken: () => Promise<void>
  isAuthenticated: boolean
  isLoading: boolean
}

const AuthContext = createContext<AuthContextType | null>(null)

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  const login = async (credentials: LoginCredentials) => {
    const response = await fetch('/api/v1/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    })

    if (!response.ok) {
      throw new Error('Login failed')
    }

    const { data } = await response.json()

    // Store tokens
    localStorage.setItem('access_token', data.access_token)
    localStorage.setItem('refresh_token', data.refresh_token)

    setUser(data.user)
  }

  const logout = () => {
    localStorage.removeItem('access_token')
    localStorage.removeItem('refresh_token')
    setUser(null)
  }

  const refreshToken = async () => {
    const refresh_token = localStorage.getItem('refresh_token')
    if (!refresh_token) return

    const response = await fetch('/api/v1/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refresh_token })
    })

    if (response.ok) {
      const { data } = await response.json()
      localStorage.setItem('access_token', data.access_token)
      localStorage.setItem('refresh_token', data.refresh_token)
    }
  }

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      refreshToken,
      isAuthenticated: !!user,
      isLoading
    }}>
      {children}
    </AuthContext.Provider>
  )
}
```

### Axios Interceptor

```typescript
// frontend/lib/api.ts
import axios from 'axios'

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000'
})

// Request interceptor to add auth token
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Response interceptor to handle token refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      const refresh_token = localStorage.getItem('refresh_token')

      if (refresh_token) {
        try {
          const response = await axios.post('/api/v1/auth/refresh', {
            refresh_token
          })

          const { access_token, refresh_token: new_refresh } = response.data.data
          localStorage.setItem('access_token', access_token)
          localStorage.setItem('refresh_token', new_refresh)

          // Retry original request
          error.config.headers.Authorization = `Bearer ${access_token}`
          return axios.request(error.config)

        } catch (refreshError) {
          // Refresh failed, redirect to login
          localStorage.removeItem('access_token')
          localStorage.removeItem('refresh_token')
          window.location.href = '/auth/login'
        }
      }
    }

    return Promise.reject(error)
  }
)

export default api
```

## Error Handling

### Authentication Errors

```python
# backend/auth/exceptions.py
class AuthError(Exception):
    """Base authentication error"""
    pass

class InvalidCredentialsError(AuthError):
    """Invalid email/password combination"""
    def __init__(self, attempts_remaining: int = None):
        self.attempts_remaining = attempts_remaining
        super().__init__("Invalid email or password")

class AccountLockedError(AuthError):
    """Account temporarily locked due to failed attempts"""
    def __init__(self, locked_until: datetime):
        self.locked_until = locked_until
        super().__init__("Account temporarily locked")

class TokenExpiredError(AuthError):
    """JWT token has expired"""
    pass

class InsufficientPermissionsError(AuthError):
    """User lacks required permissions"""
    def __init__(self, required_permission: str):
        self.required_permission = required_permission
        super().__init__(f"Permission required: {required_permission}")
```

### Global Error Handler

```python
# backend/api/error_handlers.py
from fastapi import Request
from fastapi.responses import JSONResponse
from backend.auth.exceptions import AuthError, InvalidCredentialsError

async def auth_exception_handler(request: Request, exc: AuthError):
    """Handle authentication-related exceptions"""
    if isinstance(exc, InvalidCredentialsError):
        return JSONResponse(
            status_code=401,
            content={
                "success": False,
                "error": {
                    "code": "AUTH_INVALID_CREDENTIALS",
                    "message": str(exc),
                    "details": {
                        "attempts_remaining": exc.attempts_remaining
                    }
                }
            }
        )

    # Handle other auth errors...
    return JSONResponse(
        status_code=401,
        content={
            "success": False,
            "error": {
                "code": "AUTH_ERROR",
                "message": str(exc)
            }
        }
    )
```

## Testing Authentication

### Unit Tests

```python
# tests/test_auth.py
import pytest
from fastapi.testclient import TestClient
from backend.auth.service import AuthService
from backend.models.user import User

@pytest.mark.asyncio
async def test_login_success():
    """Test successful login"""
    client = TestClient(app)

    response = client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "testpassword123"
    })

    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert "access_token" in data["data"]
    assert "refresh_token" in data["data"]

@pytest.mark.asyncio
async def test_login_invalid_credentials():
    """Test login with invalid credentials"""
    client = TestClient(app)

    response = client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "wrongpassword"
    })

    assert response.status_code == 401
    data = response.json()
    assert data["success"] is False
    assert data["error"]["code"] == "AUTH_INVALID_CREDENTIALS"

@pytest.mark.asyncio
async def test_protected_endpoint():
    """Test accessing protected endpoint"""
    client = TestClient(app)

    # Without token
    response = client.get("/api/v1/auth/me")
    assert response.status_code == 401

    # With valid token
    token = await create_test_token()
    headers = {"Authorization": f"Bearer {token}"}
    response = client.get("/api/v1/auth/me", headers=headers)
    assert response.status_code == 200
```

## Related Documentation

- [Security Overview](../security/overview.md)
- [RBAC Implementation](../security/auth.md)
- [User Management API](./users.md)
- [Frontend Authentication](../development/frontend.md#authentication)

---

[← Back to API Reference](./README.md) | [User Management →](./users.md)