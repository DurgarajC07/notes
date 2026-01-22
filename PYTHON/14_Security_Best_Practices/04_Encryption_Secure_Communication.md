# 🔐 Encryption and Secure Communication

## Overview

Implementing encryption, secure communication, and protecting sensitive data in Python applications.

---

## Symmetric Encryption

### AES Encryption with Fernet

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import base64
import os

class SecureEncryption:
    """Secure symmetric encryption using Fernet (AES-128)"""

    def __init__(self, password: str, salt: bytes = None):
        """Initialize with password"""
        if salt is None:
            salt = os.urandom(16)

        self.salt = salt

        # Derive key from password
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )

        key = base64.urlsafe_b64encode(
            kdf.derive(password.encode())
        )

        self.cipher = Fernet(key)

    def encrypt(self, plaintext: str) -> bytes:
        """Encrypt plaintext"""
        return self.cipher.encrypt(plaintext.encode())

    def decrypt(self, ciphertext: bytes) -> str:
        """Decrypt ciphertext"""
        return self.cipher.decrypt(ciphertext).decode()

# Usage
password = "my-secure-password"
encryptor = SecureEncryption(password)

# Encrypt sensitive data
plaintext = "Secret message: credit card 4532-1234-5678-9012"
encrypted = encryptor.encrypt(plaintext)
print(f"Encrypted: {encrypted}")

# Decrypt
decrypted = encryptor.decrypt(encrypted)
print(f"Decrypted: {decrypted}")

# Store salt with encrypted data
stored_data = {
    "salt": base64.b64encode(encryptor.salt).decode(),
    "ciphertext": base64.b64encode(encrypted).decode()
}

# Later, decrypt with same password and salt
def decrypt_stored_data(password: str, stored_data: dict) -> str:
    """Decrypt stored encrypted data"""
    salt = base64.b64decode(stored_data["salt"])
    ciphertext = base64.b64decode(stored_data["ciphertext"])

    decryptor = SecureEncryption(password, salt)
    return decryptor.decrypt(ciphertext)

# Encrypt files
def encrypt_file(input_file: str, output_file: str, password: str):
    """Encrypt file"""
    encryptor = SecureEncryption(password)

    with open(input_file, 'rb') as f:
        plaintext = f.read()

    encrypted = encryptor.cipher.encrypt(plaintext)

    # Store salt + encrypted data
    with open(output_file, 'wb') as f:
        f.write(encryptor.salt)
        f.write(encrypted)

def decrypt_file(input_file: str, output_file: str, password: str):
    """Decrypt file"""
    with open(input_file, 'rb') as f:
        salt = f.read(16)
        encrypted = f.read()

    decryptor = SecureEncryption(password, salt)
    plaintext = decryptor.cipher.decrypt(encrypted)

    with open(output_file, 'wb') as f:
        f.write(plaintext)
```

### AES with Custom Mode

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

class AESCipher:
    """AES encryption with custom mode"""

    def __init__(self, key: bytes):
        """
        Initialize with 256-bit key
        key must be 32 bytes for AES-256
        """
        if len(key) != 32:
            raise ValueError("Key must be 32 bytes for AES-256")

        self.key = key

    def encrypt(self, plaintext: bytes) -> dict:
        """Encrypt with AES-256-GCM"""
        # Generate random IV (12 bytes for GCM)
        iv = os.urandom(12)

        # Create cipher
        cipher = Cipher(
            algorithms.AES(self.key),
            modes.GCM(iv),
            backend=default_backend()
        )

        encryptor = cipher.encryptor()

        # Encrypt
        ciphertext = encryptor.update(plaintext) + encryptor.finalize()

        return {
            "iv": iv,
            "ciphertext": ciphertext,
            "tag": encryptor.tag  # Authentication tag
        }

    def decrypt(self, encrypted_data: dict) -> bytes:
        """Decrypt with AES-256-GCM"""
        cipher = Cipher(
            algorithms.AES(self.key),
            modes.GCM(encrypted_data["iv"], encrypted_data["tag"]),
            backend=default_backend()
        )

        decryptor = cipher.decryptor()

        plaintext = (
            decryptor.update(encrypted_data["ciphertext"])
            + decryptor.finalize()
        )

        return plaintext

# Generate secure key
key = os.urandom(32)  # 256-bit key

cipher = AESCipher(key)

# Encrypt
plaintext = b"Sensitive data"
encrypted = cipher.encrypt(plaintext)

# Decrypt
decrypted = cipher.decrypt(encrypted)
print(decrypted)  # b'Sensitive data'
```

---

## Asymmetric Encryption (RSA)

### RSA Key Generation and Usage

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import serialization, hashes

class RSAEncryption:
    """RSA asymmetric encryption"""

    def __init__(self, key_size: int = 2048):
        """Generate RSA key pair"""
        self.private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=key_size,
        )

        self.public_key = self.private_key.public_key()

    def save_private_key(self, filename: str, password: bytes):
        """Save private key to file (encrypted)"""
        pem = self.private_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.PKCS8,
            encryption_algorithm=serialization.BestAvailableEncryption(password)
        )

        with open(filename, 'wb') as f:
            f.write(pem)

    def save_public_key(self, filename: str):
        """Save public key to file"""
        pem = self.public_key.public_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PublicFormat.SubjectPublicKeyInfo
        )

        with open(filename, 'wb') as f:
            f.write(pem)

    @staticmethod
    def load_private_key(filename: str, password: bytes):
        """Load private key from file"""
        with open(filename, 'rb') as f:
            private_key = serialization.load_pem_private_key(
                f.read(),
                password=password,
            )

        instance = RSAEncryption.__new__(RSAEncryption)
        instance.private_key = private_key
        instance.public_key = private_key.public_key()
        return instance

    @staticmethod
    def load_public_key(filename: str):
        """Load public key from file"""
        with open(filename, 'rb') as f:
            return serialization.load_pem_public_key(f.read())

    def encrypt(self, plaintext: bytes) -> bytes:
        """Encrypt with public key"""
        ciphertext = self.public_key.encrypt(
            plaintext,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        return ciphertext

    def decrypt(self, ciphertext: bytes) -> bytes:
        """Decrypt with private key"""
        plaintext = self.private_key.decrypt(
            ciphertext,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        return plaintext

    def sign(self, message: bytes) -> bytes:
        """Sign message with private key"""
        signature = self.private_key.sign(
            message,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return signature

    def verify(self, message: bytes, signature: bytes) -> bool:
        """Verify signature with public key"""
        try:
            self.public_key.verify(
                signature,
                message,
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            return True
        except Exception:
            return False

# Usage
rsa_enc = RSAEncryption(key_size=2048)

# Encrypt (anyone with public key can encrypt)
message = b"Secret message"
encrypted = rsa_enc.encrypt(message)

# Decrypt (only holder of private key can decrypt)
decrypted = rsa_enc.decrypt(encrypted)
print(decrypted)  # b'Secret message'

# Sign message
signature = rsa_enc.sign(message)

# Verify signature
is_valid = rsa_enc.verify(message, signature)
print(f"Signature valid: {is_valid}")  # True

# Save keys
password = b"key-password"
rsa_enc.save_private_key("private_key.pem", password)
rsa_enc.save_public_key("public_key.pem")

# Load keys
loaded = RSAEncryption.load_private_key("private_key.pem", password)
```

---

## TLS/SSL Communication

### HTTPS Client with Cert Verification

```python
import httpx
import ssl
from pathlib import Path

class SecureHTTPClient:
    """Secure HTTP client with TLS/SSL"""

    def __init__(
        self,
        ca_cert_path: str = None,
        client_cert_path: str = None,
        client_key_path: str = None
    ):
        """
        Initialize secure client

        Args:
            ca_cert_path: CA certificate for server verification
            client_cert_path: Client certificate for mutual TLS
            client_key_path: Client private key
        """
        # Create SSL context
        context = ssl.create_default_context()

        # Load CA certificate if provided
        if ca_cert_path:
            context.load_verify_locations(ca_cert_path)

        # Load client certificate for mutual TLS
        if client_cert_path and client_key_path:
            context.load_cert_chain(client_cert_path, client_key_path)

        self.client = httpx.Client(verify=context)

    def get(self, url: str, **kwargs) -> httpx.Response:
        """Secure GET request"""
        return self.client.get(url, **kwargs)

    def post(self, url: str, **kwargs) -> httpx.Response:
        """Secure POST request"""
        return self.client.post(url, **kwargs)

    def close(self):
        """Close client"""
        self.client.close()

# Usage
client = SecureHTTPClient(
    ca_cert_path="/path/to/ca.crt",
    client_cert_path="/path/to/client.crt",
    client_key_path="/path/to/client.key"
)

response = client.get("https://api.example.com/data")
print(response.json())

client.close()

# FastAPI with TLS
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/secure-endpoint")
async def secure_endpoint():
    """Secure endpoint"""
    return {"message": "Secure communication"}

# Run with TLS
if __name__ == "__main__":
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=443,
        ssl_keyfile="/path/to/server.key",
        ssl_certfile="/path/to/server.crt",
        ssl_ca_certs="/path/to/ca.crt"  # For mutual TLS
    )
```

---

## Secure Token Generation

### JWT with Encryption

```python
from jose import jwt, JWTError
from datetime import datetime, timedelta
import secrets

class SecureJWT:
    """Secure JWT token generation"""

    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        """
        Initialize JWT handler

        Args:
            secret_key: Secret for signing (keep secure!)
            algorithm: HS256, HS384, HS512, RS256, etc.
        """
        self.secret_key = secret_key
        self.algorithm = algorithm

    def create_access_token(
        self,
        data: dict,
        expires_delta: timedelta = None
    ) -> str:
        """Create JWT access token"""
        to_encode = data.copy()

        if expires_delta:
            expire = datetime.utcnow() + expires_delta
        else:
            expire = datetime.utcnow() + timedelta(minutes=15)

        to_encode.update({
            "exp": expire,
            "iat": datetime.utcnow(),
            "jti": secrets.token_urlsafe(16)  # JWT ID for tracking
        })

        encoded_jwt = jwt.encode(
            to_encode,
            self.secret_key,
            algorithm=self.algorithm
        )

        return encoded_jwt

    def decode_token(self, token: str) -> dict:
        """Decode and verify JWT token"""
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm]
            )
            return payload

        except JWTError as e:
            raise ValueError(f"Invalid token: {e}")

    def create_refresh_token(self, data: dict) -> str:
        """Create refresh token (longer expiry)"""
        return self.create_access_token(
            data,
            expires_delta=timedelta(days=7)
        )

# Usage
jwt_handler = SecureJWT(
    secret_key="your-secret-key-keep-this-secure",
    algorithm="HS256"
)

# Create token
user_data = {"sub": "user@example.com", "role": "admin"}
access_token = jwt_handler.create_access_token(user_data)

print(f"Token: {access_token}")

# Verify token
try:
    payload = jwt_handler.decode_token(access_token)
    print(f"Payload: {payload}")
except ValueError as e:
    print(f"Invalid token: {e}")

# FastAPI integration
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()
jwt_handler = SecureJWT("secret-key")

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    """Get current user from JWT token"""
    try:
        payload = jwt_handler.decode_token(credentials.credentials)
        return payload
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

@app.get("/protected")
async def protected_route(current_user: dict = Depends(get_current_user)):
    """Protected route requiring valid JWT"""
    return {"message": f"Hello {current_user['sub']}"}
```

---

## Secure Password Storage

### Argon2 Password Hashing

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

class SecurePasswordManager:
    """Secure password hashing with Argon2"""

    def __init__(self):
        """Initialize password hasher"""
        self.ph = PasswordHasher(
            time_cost=2,        # Iterations
            memory_cost=65536,  # Memory in KB (64 MB)
            parallelism=4,      # Threads
            hash_len=32,        # Hash length
            salt_len=16         # Salt length
        )

    def hash_password(self, password: str) -> str:
        """Hash password securely"""
        return self.ph.hash(password)

    def verify_password(self, hash: str, password: str) -> bool:
        """Verify password against hash"""
        try:
            self.ph.verify(hash, password)

            # Check if rehashing needed (parameters changed)
            if self.ph.check_needs_rehash(hash):
                return "rehash"

            return True

        except VerifyMismatchError:
            return False

# Usage
pm = SecurePasswordManager()

# Hash password
password = "user-secure-password"
hashed = pm.hash_password(password)
print(f"Hashed: {hashed}")

# Verify
is_valid = pm.verify_password(hashed, password)
print(f"Valid: {is_valid}")  # True

is_valid = pm.verify_password(hashed, "wrong-password")
print(f"Valid: {is_valid}")  # False

# FastAPI integration
from fastapi import HTTPException

@app.post("/register")
async def register(username: str, password: str):
    """Register new user"""
    # Hash password
    hashed_password = pm.hash_password(password)

    # Store in database
    user = User(username=username, password_hash=hashed_password)
    db.add(user)
    db.commit()

    return {"message": "User registered"}

@app.post("/login")
async def login(username: str, password: str):
    """Login user"""
    # Get user from database
    user = db.query(User).filter(User.username == username).first()

    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Verify password
    result = pm.verify_password(user.password_hash, password)

    if result == "rehash":
        # Password valid but needs rehashing
        user.password_hash = pm.hash_password(password)
        db.commit()
        result = True

    if not result:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Create token
    token = jwt_handler.create_access_token({"sub": username})

    return {"access_token": token, "token_type": "bearer"}
```

---

## Secrets Management

### Environment Variables and Vault

```python
import os
from pathlib import Path
from typing import Optional

class SecretsManager:
    """Manage application secrets"""

    def __init__(self):
        """Load secrets from environment or vault"""
        self._secrets = {}
        self._load_from_env()

    def _load_from_env(self):
        """Load secrets from environment variables"""
        # Critical secrets
        self._secrets = {
            "database_url": os.getenv("DATABASE_URL"),
            "redis_url": os.getenv("REDIS_URL"),
            "jwt_secret": os.getenv("JWT_SECRET_KEY"),
            "encryption_key": os.getenv("ENCRYPTION_KEY"),
            "api_key": os.getenv("API_KEY"),
        }

    def get_secret(self, key: str) -> Optional[str]:
        """Get secret by key"""
        value = self._secrets.get(key)

        if value is None:
            raise ValueError(f"Secret '{key}' not found")

        return value

    def get_database_url(self) -> str:
        """Get database connection URL"""
        return self.get_secret("database_url")

# Singleton instance
secrets = SecretsManager()

# Usage in application
DATABASE_URL = secrets.get_database_url()
JWT_SECRET = secrets.get_secret("jwt_secret")

# .env file (never commit to git!)
"""
DATABASE_URL=postgresql://user:pass@localhost/db
REDIS_URL=redis://localhost:6379
JWT_SECRET_KEY=your-secret-key-here
ENCRYPTION_KEY=your-encryption-key-here
API_KEY=your-api-key-here
"""

# Load from .env file
from dotenv import load_dotenv

load_dotenv()  # Load from .env file

# HashiCorp Vault integration
import hvac

class VaultSecretsManager:
    """Secrets from HashiCorp Vault"""

    def __init__(self, vault_url: str, token: str):
        """Connect to Vault"""
        self.client = hvac.Client(url=vault_url, token=token)

    def get_secret(self, path: str, key: str) -> str:
        """Get secret from Vault"""
        response = self.client.secrets.kv.v2.read_secret_version(
            path=path
        )

        return response['data']['data'][key]

# Usage
vault = VaultSecretsManager(
    vault_url="http://vault.example.com:8200",
    token=os.getenv("VAULT_TOKEN")
)

db_password = vault.get_secret("database", "password")
```

---

## Best Practices

### ✅ Do's:

1. **Use strong encryption** - AES-256, RSA-2048+
2. **Store keys securely** - Environment vars, Vault
3. **Use TLS/SSL** for network communication
4. **Hash passwords** with Argon2 or bcrypt
5. **Rotate keys** regularly
6. **Use secrets manager** (Vault, AWS Secrets Manager)
7. **Never log** secrets or keys
8. **Use HTTPS** everywhere
9. **Implement** certificate pinning for mobile
10. **Encrypt data** at rest and in transit

### ❌ Don'ts:

1. **Don't hardcode** secrets in code
2. **Don't commit** secrets to git
3. **Don't use** weak algorithms (MD5, SHA1, DES)
4. **Don't reuse** IVs or salts
5. **Don't roll** your own crypto
6. **Don't store** passwords in plaintext
7. **Don't transmit** sensitive data over HTTP

---

## Interview Questions

### Q1: Symmetric vs Asymmetric encryption?

**Answer**:

- **Symmetric**: Same key encrypt/decrypt (AES, ChaCha20)
- **Asymmetric**: Public/private key pair (RSA, ECC)
- **Symmetric**: Fast, smaller keys, key distribution problem
- **Asymmetric**: Slower, larger keys, no key sharing needed
  Use symmetric for data, asymmetric for key exchange.

### Q2: How to securely store passwords?

**Answer**:

- **Never plaintext** - always hash
- **Use Argon2** or bcrypt (slow, salted)
- **Don't use** MD5 or SHA (too fast)
- **Add salt** - random per password
- **Increase cost** - slow down brute force

### Q3: What is TLS and why use it?

**Answer**: Transport Layer Security:

- **Encrypts** data in transit
- **Authenticates** server (certificate)
- **Prevents** MITM attacks
- **Mutual TLS**: Both sides authenticate
  Always use HTTPS (HTTP over TLS).

### Q4: How to manage secrets in production?

**Answer**:

- **Use secrets manager** (Vault, AWS Secrets Manager)
- **Environment variables** for less critical
- **Never commit** to version control
- **Rotate regularly** - especially after exposure
- **Principle of least privilege** - only what's needed

### Q5: What is certificate pinning?

**Answer**: Validate specific certificate:

- **Prevent MITM** - even with valid cert
- **Pin certificate** or public key hash
- **Mobile apps** - prevent proxy inspection
- **Trade-off**: Harder to update certs
  Use for high-security mobile apps.

---

## Summary

Encryption essentials:

- **Symmetric**: AES for fast data encryption
- **Asymmetric**: RSA for key exchange, signatures
- **TLS/SSL**: Secure network communication
- **Password hashing**: Argon2 with salt
- **Secrets management**: Vault or environment vars
- **Never roll your own** crypto

Security is not optional! 🔐
