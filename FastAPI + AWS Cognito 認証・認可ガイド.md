# FastAPI + AWS Cognito 認証・認可ガイド
## 🧪 起動とテスト

### 環境変数の設定
```bash
# 開発環境で起動
export ENVIRONMENT=development
# または
echo "ENVIRONMENT=development" > .env
```

### 開発サーバー起動
```bash
# 基本的な起動
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# ログレベルを指定して起動
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000 --log-level debug

# 本番環境での起動例
export ENVIRONMENT=production
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

### デバッグ情報の確認
```bash
# アプリケーションの状態確認
curl http://localhost:8000/

# デバッグ情報取得（開発環境のみ）
curl -H# 📘 FastAPI + AWS Cognito 認証・認可ガイド（`fastapi-cloudauth` 編）

このドキュメントは、`fastapi-cloudauth` を使って FastAPI アプリケーションを AWS Cognito と連携させ、**IDトークンの検証と認可処理**を実装する方法を示します。

---

## 📦 必要なライブラリのインストール

```bash
pip install fastapi fastapi-cloudauth[cognito] uvicorn python-multipart
pip install python-decouple python-json-logger
```

### requirements.txt
```txt
fastapi==0.104.1
fastapi-cloudauth[cognito]==0.5.1
uvicorn[standard]==0.24.0
python-decouple==3.8
python-json-logger==2.0.7
python-multipart==0.0.6
pydantic==2.5.0
```

---

## 🧩 使用技術

- **FastAPI**: Python製の高性能Web APIフレームワーク
- **AWS Cognito**: マネージド認証サービス（OIDC準拠のIDトークン発行）
- **fastapi-cloudauth**: 主要クラウドIDプロバイダ向けの FastAPI 認証ライブラリ
- **python-decouple**: 環境変数管理ライブラリ

---

## 🔐 アーキテクチャ概要

```text
[フロントエンド/モバイルクライアント]
    ↓ ログイン要求
[AWS Cognito User Pool]
    ↓ IDトークン (JWT) 発行
[クライアント]
    ↓ Authorization: Bearer <JWT>
[FastAPI + fastapi-cloudauth]
    ↓ JWTの署名検証・有効期限チェック・Claims抽出
[保護されたAPIエンドポイント]
```

---

## 🔧 AWS Cognito の事前準備

### 1. ユーザープール作成
- AWS Cognito コンソールでユーザープールを作成
- サインインオプションを設定（ユーザー名/メール等）

### 2. アプリクライアント設定
- アプリクライアントを作成し、`client_id` を取得
- 認証フローを設定（ALLOW_USER_SRP_AUTH、ALLOW_REFRESH_TOKEN_AUTH等）
- トークンの有効期限を適切に設定

### 3. カスタム属性設定（オプション）
- ロール管理用に `custom:role` 属性を追加
- 属性は可変（Mutable）に設定

### 4. 必要な情報を記録
- ユーザープールID: `ap-northeast-1_XXXXXXXXX`
- リージョン: `ap-northeast-1`
- アプリクライアントID: `xxxxxxxxxxxxxxxxxxxxxxxxx`

---

## 📄 プロジェクト構成例

```
backend/
├── config/
│   ├── auth_settings.py   # 認証・認可の基本設定
│   ├── logging_config.py  # ログ設定
│   └── ...
├── api/
│   └── api_v1/
│       └── auth/
│           └── dependencies.py
├── business/
├── models/
├── schemas/
├── utils/
├── requirements.txt
└── README.md
```

---

## 🔧 設定管理（`backend/config/auth_settings.py`）

```python
# backend/config/auth_settings.py
import os
from typing import Optional
from pydantic import BaseSettings, validator
from enum import Enum

class Environment(str, Enum):
    DEVELOPMENT = "development"
    STAGING = "staging"
    PRODUCTION = "production"

class LogLevel(str, Enum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"

class Settings(BaseSettings):
    # 環境設定
    environment: Environment = Environment.DEVELOPMENT
    debug: bool = False
    
    # ログ設定
    log_level: LogLevel = LogLevel.INFO
    log_format: str = "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    
    # AWS Cognito設定
    cognito_user_pool_id: str
    cognito_region: str = "ap-northeast-1"
    cognito_app_client_id: str
    
    # API設定
    api_title: str = "FastAPI + AWS Cognito API"
    api_version: str = "1.0.0"
    api_prefix: str = "/api/v1"
    
    # CORS設定
    allowed_origins: list[str] = ["http://localhost:3000"]
    allowed_methods: list[str] = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allowed_headers: list[str] = ["Authorization", "Content-Type"]
    
    # JWT設定
    jwt_algorithm: str = "RS256"
    jwt_audience: Optional[str] = None
    
    @validator("cognito_user_pool_id")
    def validate_user_pool_id(cls, v):
        if not v.startswith(("us-", "eu-", "ap-", "ca-", "sa-")):
            raise ValueError("Invalid Cognito User Pool ID format")
        return v
    
    @validator("allowed_origins", pre=True)
    def parse_origins(cls, v):
        if isinstance(v, str):
            return [origin.strip() for origin in v.split(",")]
        return v
    
    class Config:
        env_file = ".env"
        case_sensitive = False
        
        @classmethod
        def customise_sources(cls, init_settings, env_settings, file_secret_settings):
            # 環境ごとの設定ファイルを読み込む
            env = os.getenv("ENVIRONMENT", "development")
            env_file_path = f".env.{env}"
            
            if os.path.exists(env_file_path):
                env_settings.env_file = env_file_path
            
            return (
                init_settings,
                env_settings,
                file_secret_settings,
            )

# シングルトンパターンで設定を管理
_settings: Optional[Settings] = None

def get_settings() -> Settings:
    global _settings
    if _settings is None:
        _settings = Settings()
    return _settings

settings = get_settings()
```

---

## 🔐 認証依存性とエラーハンドリング（`backend/api/api_v1/auth/dependencies.py`）

```python
# backend/api/api_v1/auth/dependencies.py
import logging  # ロギング用モジュール
from typing import Optional  # 型ヒント用
from fastapi import HTTPException, status, Request  # FastAPIの主要クラス
from fastapi_cloudauth.cognito import Cognito, CognitoClaims, CognitoCurrentUser  # Cognito認証用
from fastapi_cloudauth.exceptions import CloudAuthError  # 認証例外
from backend.config.auth_settings import get_settings  # 設定取得

logger = logging.getLogger(__name__)  # このモジュール用のロガー取得
settings = get_settings()  # 設定インスタンス取得

# Cognito認証の設定
auth = Cognito(
    user_pool_id=settings.cognito_user_pool_id,  # ユーザープールIDを設定
    region=settings.cognito_region,  # リージョンを設定
    client_id=settings.cognito_app_client_id,  # アプリクライアントIDを設定
    auto_error=False  # 手動でエラーハンドリングを行う
)

class AuthenticationError(Exception):
    """認証エラーの基底クラス"""
    def __init__(self, message: str, status_code: int = 401):
        self.message = message  # エラーメッセージ
        self.status_code = status_code  # ステータスコード
        super().__init__(self.message)  # 親クラス初期化

class AuthorizationError(Exception):
    """認可エラーの基底クラス"""
    def __init__(self, message: str, status_code: int = 403):
        self.message = message  # エラーメッセージ
        self.status_code = status_code  # ステータスコード
        super().__init__(self.message)  # 親クラス初期化

async def get_current_user_with_error_handling(request: Request) -> Optional[CognitoClaims]:
    """エラーハンドリング付きの現在ユーザー取得"""
    try:
        # Authorization ヘッダーの存在確認
        authorization = request.headers.get("Authorization")  # ヘッダーからトークン取得
        if not authorization:
            logger.warning("Authorization header is missing")  # ヘッダーがなければ警告
            return None
            
        if not authorization.startswith("Bearer "):
            logger.warning("Invalid authorization header format")  # フォーマット不正なら警告
            return None
            
        # トークン検証
        user = await auth.current_user(request)  # トークンからユーザー情報取得
        if user:
            logger.info(f"User authenticated successfully: {user.username}")  # 成功ログ
            return user
        else:
            logger.warning("Token validation failed")  # 失敗ログ
            return None
            
    except CloudAuthError as e:
        logger.error(f"CloudAuth error: {str(e)}")  # 認証系エラー
        return None
    except Exception as e:
        logger.error(f"Unexpected authentication error: {str(e)}")  # その他例外
        return None

async def require_authentication(request: Request) -> CognitoClaims:
    """認証が必須のエンドポイント用依存性"""
    user = await get_current_user_with_error_handling(request)  # ユーザー取得
    if not user:
        logger.warning(f"Authentication failed for request to {request.url.path}")  # 失敗ログ
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,  # 401エラー
            detail={
                "error": "authentication_required",
                "message": "有効な認証トークンが必要です"
            },
            headers={"WWW-Authenticate": "Bearer"}
        )
    return user  # 認証済みユーザー返却

async def require_admin_role(request: Request) -> CognitoClaims:
    """管理者権限が必要なエンドポイント用の依存性"""
    user = await require_authentication(request)  # 認証必須
    user_role = user.get("custom:role")  # ロール取得
    
    if user_role != "admin":
        logger.warning(f"Access denied for user {user.username} (role: {user_role}) to admin endpoint")  # 権限不足
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,  # 403エラー
            detail={
                "error": "insufficient_permissions",
                "message": "管理者権限が必要です",
                "required_role": "admin",
                "current_role": user_role
            }
        )
    
    logger.info(f"Admin access granted for user: {user.username}")  # 成功ログ
    return user  # 管理者ユーザー返却

async def require_user_or_admin_role(request: Request) -> CognitoClaims:
    """一般ユーザーまたは管理者権限が必要なエンドポイント用の依存性"""
    user = await require_authentication(request)  # 認証必須
    user_role = user.get("custom:role")  # ロール取得
    allowed_roles = ["user", "admin"]  # 許可ロール
    
    if user_role not in allowed_roles:
        logger.warning(f"Access denied for user {user.username} (role: {user_role})")  # 権限不足
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,  # 403エラー
            detail={
                "error": "insufficient_permissions", 
                "message": "適切な権限がありません",
                "required_roles": allowed_roles,
                "current_role": user_role
            }
        )
    
    logger.info(f"User access granted for user: {user.username} (role: {user_role})")  # 成功ログ
    return user  # 許可ユーザー返却

# オプション: 現在のユーザー情報を取得（認証失敗時はNoneを返す）
current_user_optional = get_current_user_with_error_handling  # 認証失敗時None返却の依存性
```

---

## 📊 ログ設定（`backend/config/logging_config.py`）

```python
# backend/config/logging_config.py
import logging
import logging.config
import sys
from typing import Dict, Any
from backend.config.auth_settings import get_settings

settings = get_settings()

def get_logging_config() -> Dict[str, Any]:
    """環境に応じたログ設定を返す"""
    
    base_config = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "format": settings.log_format,
                "datefmt": "%Y-%m-%d %H:%M:%S",
            },
            "detailed": {
                "format": "%(asctime)s - %(name)s - %(levelname)s - %(funcName)s:%(lineno)d - %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S",
            },
            "json": {
                "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
                "format": "%(asctime)s %(name)s %(levelname)s %(funcName)s %(lineno)d %(message)s"
            }
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "level": settings.log_level.value,
                "formatter": "default" if settings.environment.value == "development" else "json",
                "stream": sys.stdout,
            },
        },
        "loggers": {
            "app": {
                "level": settings.log_level.value,
                "handlers": ["console"],
                "propagate": False,
            },
            "fastapi_cloudauth": {
                "level": "INFO",
                "handlers": ["console"],
                "propagate": False,
            },
            "uvicorn": {
                "level": "INFO",
                "handlers": ["console"],
                "propagate": False,
            },
        },
        "root": {
            "level": settings.log_level.value,
            "handlers": ["console"],
        },
    }
    
    # 本番環境では詳細なログを出力
    if settings.environment.value == "production":
        base_config["handlers"]["file"] = {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "WARNING",
            "formatter": "json",
            "filename": "/var/log/app/app.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,
        }
        base_config["loggers"]["app"]["handlers"].append("file")
    
    return base_config

def setup_logging():
    """ログ設定の初期化"""
    config = get_logging_config()
    logging.config.dictConfig(config)
    
    # 設定情報をログ出力
    logger = logging.getLogger("app.setup")
    logger.info(f"Logging configured for environment: {settings.environment.value}")
    logger.info(f"Log level: {settings.log_level.value}")
```

```python
# backend/api/main.py
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from fastapi_cloudauth.cognito import CognitoClaims
from fastapi_cloudauth.exceptions import CloudAuthError

from backend.config.auth_settings import get_settings
from backend.config.logging_config import setup_logging
from backend.api.auth.dependencies import (
    require_authentication,
    require_admin_role, 
    require_user_or_admin_role,
    current_user_optional,
    AuthenticationError,
    AuthorizationError
)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """アプリケーションのライフサイクル管理"""
    # 起動時の処理
    setup_logging()
    logger = logging.getLogger("app.startup")
    logger.info("FastAPI application starting up...")
    logger.info(f"Environment: {settings.environment.value}")
    logger.info(f"Debug mode: {settings.debug}")
    
    yield
    
    # 終了時の処理
    logger.info("FastAPI application shutting down...")

settings = get_settings()

app = FastAPI(
    title=settings.api_title,
    description="fastapi-cloudauthを使用したCognito認証の実装例",
    version=settings.api_version,
    debug=settings.debug,
    lifespan=lifespan
)

# ログ設定
logger = logging.getLogger("app.main")

# グローバル例外ハンドラー
@app.exception_handler(AuthenticationError)
async def authentication_exception_handler(request: Request, exc: AuthenticationError):
    logger.warning(f"Authentication error: {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": "authentication_failed",
            "message": exc.message,
            "path": str(request.url.path)
        }
    )

@app.exception_handler(AuthorizationError)
async def authorization_exception_handler(request: Request, exc: AuthorizationError):
    logger.warning(f"Authorization error: {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": "authorization_failed", 
            "message": exc.message,
            "path": str(request.url.path)
        }
    )

@app.exception_handler(CloudAuthError)
async def cloudauth_exception_handler(request: Request, exc: CloudAuthError):
    logger.error(f"CloudAuth error: {str(exc)}")
    return JSONResponse(
        status_code=401,
        content={
            "error": "token_validation_failed",
            "message": "トークンの検証に失敗しました",
            "details": str(exc) if settings.debug else None
        }
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "internal_server_error",
            "message": "内部サーバーエラーが発生しました",
            "details": str(exc) if settings.debug else None
        }
    )

# CORS設定
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=settings.allowed_methods,
    allow_headers=settings.allowed_headers,
)

# リクエストログミドルウェア
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    
    # リクエストログ
    logger.info(f"Request: {request.method} {request.url.path}")
    if settings.log_level.value == "DEBUG":
        logger.debug(f"Request headers: {dict(request.headers)}")
    
    response = await call_next(request)
    
    # レスポンスログ
    process_time = time.time() - start_time
    logger.info(
        f"Response: {response.status_code} - {request.method} {request.url.path} - {process_time:.3f}s"
    )
    
    return response

@app.get("/")
def root():
    """ヘルスチェック用エンドポイント"""
    return {"message": "FastAPI + AWS Cognito認証サーバーが稼働中です"}

@app.get("/public")
def public_endpoint():
    """認証不要のパブリックエンドポイント"""
    return {"message": "認証不要のエンドポイントです"}

@app.get("/protected")
async def protected_endpoint(user: CognitoClaims = Depends(require_authentication)):
    """認証が必要な保護されたエンドポイント"""
    logger.info(f"Protected endpoint accessed by user: {user.username}")
    return {
        "message": f"こんにちは、{user.username}さん",
        "user_info": {
            "username": user.username,
            "email": user.email,
            "sub": user.sub,
            "role": user.get("custom:role", "未設定")
        }
    }

@app.get("/user")
async def user_endpoint(user: CognitoClaims = Depends(require_user_or_admin_role)):
    """一般ユーザー以上の権限が必要なエンドポイント"""
    logger.info(f"User endpoint accessed by: {user.username} (role: {user.get('custom:role')})")
    return {
        "message": "ユーザー専用エンドポイントです",
        "user_data": {
            "username": user.username,
            "role": user.get("custom:role")
        }
    }

@app.get("/admin")
async def admin_only_endpoint(user: CognitoClaims = Depends(require_admin_role)):
    """管理者専用エンドポイント"""
    logger.info(f"Admin endpoint accessed by: {user.username}")
    return {
        "message": "管理者専用エンドポイントです",
        "admin_info": {
            "username": user.username,
            "email": user.email,
            "permissions": ["read", "write", "delete"]
        }
    }

@app.get("/profile")
async def get_user_profile(user: CognitoClaims = Depends(require_authentication)):
    """ユーザープロファイル取得"""
    logger.debug(f"Profile requested for user: {user.username}")
    return {
        "profile": {
            "sub": user.sub,
            "username": user.username,
            "email": user.email,
            "email_verified": user.get("email_verified", False),
            "role": user.get("custom:role", "未設定"),
            "created_at": user.get("auth_time"),
            "token_use": user.get("token_use")
        }
    }

# デバッグ用エンドポイント（開発環境のみ）
if settings.debug:
    @app.get("/debug/token")
    async def debug_token_info(request: Request, user: CognitoClaims = Depends(current_user_optional)):
        """トークンの詳細情報を返す（デバッグ用）"""
        authorization = request.headers.get("Authorization", "")
        return {
            "has_token": bool(authorization),
            "user_authenticated": user is not None,
            "user_info": dict(user) if user else None,
            "raw_claims": user.raw_claims if user else None
        }
```

---

## 🔑 クライアント側の認証フロー

### 1. ログイン（例：JavaScript）
```javascript
// AWS Amplify を使用した例
import { Auth } from 'aws-amplify';

// ログイン
const signIn = async (username, password) => {
  try {
    const user = await Auth.signIn(username, password);
    const idToken = user.signInUserSession.idToken.jwtToken;
    
    // IDトークンを保存（sessionStorageまたはメモリ）
    sessionStorage.setItem('idToken', idToken);
    return idToken;
  } catch (error) {
    console.error('ログインエラー:', error);
  }
};
```

### 2. API呼び出し
```javascript
// 保護されたエンドポイントへのリクエスト
const callProtectedAPI = async () => {
  const idToken = sessionStorage.getItem('idToken');
  
  const response = await fetch('http://localhost:8000/protected', {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${idToken}`,
      'Content-Type': 'application/json'
    }
  });
  
  return response.json();
};
```

### 3. cURLでのテスト例
```bash
# IDトークンを取得後、以下のようにテスト
curl -H "Authorization: Bearer eyJraWQiOiJrZX..." \
     -H "Content-Type: application/json" \
     http://localhost:8000/protected
```

---

## 🧪 起動とテスト

### 開発サーバー起動
```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### 動作確認
1. **パブリックエンドポイント**: `http://localhost:8000/public`
2. **API仕様書**: `http://localhost:8000/docs`
3. **保護されたエンドポイント**: 認証トークンが必要

---

## 🔒 セキュリティベストプラクティス

### 1. 環境変数の管理
- 本番環境では `.env` ファイルをGitに含めない
- AWS Systems Manager Parameter Store や AWS Secrets Manager の利用を検討

### 2. トークンの取り扱い
- IDトークンはHTTPS通信でのみ送信
- クライアント側でのトークン保存は最小限に（sessionStorage推奨）
- リフレッシュトークンは適切に管理

### 3. CORS設定
```python
# 本番環境では具体的なオリジンを指定
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

---

## 🧭 他のIDプロバイダーへの移行

`fastapi-cloudauth` は複数のプロバイダーをサポートしているため、将来的な移行が容易です。

### Auth0への移行例
```python
from fastapi_cloudauth.auth0 import Auth0, Auth0CurrentUser

auth = Auth0(domain="your-domain.auth0.com", client_id="your-client-id")
current_user: Auth0CurrentUser = auth.current_user
```

### Google OAuth への移行例
```python
from fastapi_cloudauth.google import Google, GoogleCurrentUser

auth = Google(client_id="your-google-client-id.googleusercontent.com")
current_user: GoogleCurrentUser = auth.current_user
```

---

## ⚡ パフォーマンス最適化

### 1. JWKSキャッシュ
`fastapi-cloudauth` は自動的にJWKS（JSON Web Key Set）をキャッシュしますが、必要に応じて設定を調整できます。

### 2. 依存性キャッシュ
頻繁にアクセスされるエンドポイントでは、依存性の結果をキャッシュすることを検討してください。

---

## 🚨 トラブルシューティング

### よくあるエラーと解決方法

1. **401 Unauthorized**
   - IDトークンの有効期限切れ → トークンを再取得
   - 不正な形式のトークン → Authorization ヘッダーの形式を確認

2. **403 Forbidden**  
   - 権限不足 → ユーザーのロール設定を確認
   - カスタム属性の設定ミス → Cognito側の属性設定を確認

3. **Configuration Error**
   - 環境変数の設定ミス → `.env` ファイルと設定値を再確認
   - リージョンの不一致 → USER_POOL_IDとREGIONの整合性を確認

---

## ✅ まとめ

| 機能 | 実装方法 |
|------|----------|
| 認証 | `fastapi_cloudauth.cognito.Cognito` による自動JWT検証 |
| 認可 | カスタム依存性による権限チェック（`custom:role`ベース） |
| SSO対応 | プロバイダー変更時も同じインターフェースで対応可能 |
| セキュリティ | JWKS自動取得・署名検証・有効期限チェック |
| 拡張性 | 依存性注入パターンで柔軟な認可ロジックを実装可能 |

---

## 📚 参考リンク

- [fastapi-cloudauth GitHub](https://github.com/tokusumi/fastapi-cloudauth)
- [AWS Cognito JWT トークンの使用](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-with-identity-providers.html)
- [FastAPI 公式ドキュメント](https://fastapi.tiangolo.com/)
- [AWS Cognito User Pools](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-identity-pools.html)
- [JWT.io - JWT デバッガー](https://jwt.io/)

---

## 📝 ローカル開発用 `.env` サンプル

```env
# 環境名
ENVIRONMENT=development

# AWS Cognito 設定
COGNITO_USER_POOL_ID=ap-northeast-1_xxxxxxxxx
COGNITO_REGION=ap-northeast-1
COGNITO_APP_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxx

# ログ設定
LOG_LEVEL=DEBUG

# CORS設定（カンマ区切りで複数指定可）
ALLOWED_ORIGINS=http://localhost:3000
```

> ※ 必要な変数は `backend/config/auth_settings.py` の `Settings` クラスを参照してください。

---

## 🛡️ AWS本番環境での安全な設定値管理（Parameter Store / Secrets Manager）

本番環境では `.env` ファイルを使わず、AWSの公式サービスで機密情報を安全に管理してください。

### Parameter Store/Secrets Manager での推奨設定例

- **Parameter Store** 例:
  - `/your-app/ENVIRONMENT` = `production`
  - `/your-app/COGNITO_USER_POOL_ID` = `ap-northeast-1_xxxxxxxxx`
  - `/your-app/COGNITO_REGION` = `ap-northeast-1`
  - `/your-app/COGNITO_APP_CLIENT_ID` = `xxxxxxxxxxxxxxxxxxxxxxxxx`
  - `/your-app/LOG_LEVEL` = `INFO`
  - `/your-app/ALLOWED_ORIGINS` = `https://yourdomain.com`

- **Secrets Manager** 例:
  - シークレット名: `your-app/production/cognito`
  - キーと値:
    - `COGNITO_USER_POOL_ID`: `ap-northeast-1_xxxxxxxxx`
    - `COGNITO_APP_CLIENT_ID`: `xxxxxxxxxxxxxxxxxxxxxxxxx`
    - ...（必要に応じて追加）

### 使い方のポイント
- ECSやLambdaのタスク定義で、Parameter StoreやSecrets Managerから値を取得し、環境変数として渡す設定が可能です。
- IaC（CloudFormation, CDK, Terraform等）で自動化するのが推奨です。
- FastAPI/Pydanticの設定クラスは、通常の環境変数として値が渡されればそのまま利用できます。

> 詳細はAWS公式ドキュメントも参照してください。
> - [Parameter Store](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/systems-manager-parameter-store.html)
> - [Secrets Manager](https://docs.aws.amazon.com/ja_jp/secretsmanager/latest/userguide/intro.html)

---

## 📝 変更履歴

- v1.0.0: 初版作成
- v1.1.0: セキュリティベストプラクティス追加、エラーハンドリング改善
- v1.2.0: AWS環境でのCognito設定値の管理方法・Parameter Store/Secrets Manager運用例を追記
- v1.3.0: ローカル用.envサンプルと本番用AWS設定例を明記、変更履歴を末尾に移動
