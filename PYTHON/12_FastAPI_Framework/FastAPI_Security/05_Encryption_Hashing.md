# 🔐 Data Encryption and Hashing

## Overview

Encryption and hashing are fundamental security practices for protecting sensitive data at rest and in transit. Understanding when and how to use each is critical for building secure FastAPI applications.

---

## Hashing vs Encryption

### Key Differences

| Aspect           | Hashing                           | Encryption                 |
| ---------------- | --------------------------------- | -------------------------- |
| **Reversible**   | No (one-way)                      | Yes (two-way)              |
| **Purpose**      | Verify integrity, store passwords | Protect confidential data  |
| **Output**       | Fixed-length hash                 | Variable-length ciphertext |
| **Key Required** | No                                | Yes                        |
| **Use Cases**    | Passwords, checksums              | Credit cards, PII, secrets |

---

## Password Hashing

### 1. **Bcrypt (Recommended)**

```python
from passlib.context import CryptContext

# Create password context
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    """Hash password using bcrypt"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

# Usage
password = "mySecurePassword123!"
hashed = hash_password(password)
print(hashed)  # $2b$12$...

# Verify
is_valid = verify_password("mySecurePassword123!", hashed)
print(is_valid)  # True

is_invalid = verify_password("wrongPassword", hashed)
print(is_invalid)  # False
```

### 2. **Argon2 (Modern Alternative)**

```python
from passlib.context import CryptContext

# Argon2 - winner of Password Hashing Competition
pwd_context_argon2 = CryptContext(
    schemes=["argon2"],
    deprecated="auto",
    argon2__memory_cost=65536,  # 64 MB
    argon2__time_cost=3,         # 3 iterations
    argon2__parallelism=4        # 4 threads
)

def hash_password_argon2(password: str) -> str:
    """Hash password using Argon2"""
    return pwd_context_argon2.hash(password)

def verify_password_argon2(plain_password: str, hashed_password: str) -> bool:
    """Verify password with Argon2"""
    return pwd_context_argon2.verify(plain_password, hashed_password)
```

### 3. **Multi-Scheme Support**

```python
# Support multiple schemes with migration
pwd_context_multi = CryptContext(
    schemes=["bcrypt", "argon2", "pbkdf2_sha256"],
    deprecated=["pbkdf2_sha256"],  # Mark old scheme as deprecated
    bcrypt__default_rounds=12,
    argon2__memory_cost=65536
)

def hash_password_multi(password: str) -> str:
    """Hash with default scheme (bcrypt or argon2)"""
    return pwd_context_multi.hash(password)

def verify_and_migrate_password(
    plain_password: str,
    hashed_password: str
) -> tuple[bool, Optional[str]]:
    """Verify password and migrate if using deprecated scheme"""
    # Verify password
    is_valid = pwd_context_multi.verify(plain_password, hashed_password)

    if not is_valid:
        return False, None

    # Check if needs rehashing
    if pwd_context_multi.needs_update(hashed_password):
        new_hash = pwd_context_multi.hash(plain_password)
        return True, new_hash

    return True, None

# Usage in login
def login_user(email: str, password: str, db: Session):
    user = db.query(User).filter(User.email == email).first()

    if not user:
        return None

    # Verify and potentially migrate hash
    is_valid, new_hash = verify_and_migrate_password(
        password,
        user.hashed_password
    )

    if not is_valid:
        return None

    # Update hash if migrated
    if new_hash:
        user.hashed_password = new_hash
        db.commit()

    return user
```

---

## Symmetric Encryption

### 1. **Fernet (Recommended for Simple Cases)**

```python
from cryptography.fernet import Fernet
import base64

class EncryptionManager:
    """Symmetric encryption using Fernet"""

    def __init__(self, key: bytes = None):
        if key is None:
            key = Fernet.generate_key()
        self.cipher = Fernet(key)
        self.key = key

    def encrypt(self, data: str) -> str:
        """Encrypt string data"""
        encrypted = self.cipher.encrypt(data.encode())
        return base64.urlsafe_b64encode(encrypted).decode()

    def decrypt(self, encrypted_data: str) -> str:
        """Decrypt string data"""
        encrypted_bytes = base64.urlsafe_b64decode(encrypted_data.encode())
        decrypted = self.cipher.decrypt(encrypted_bytes)
        return decrypted.decode()

    @staticmethod
    def generate_key() -> str:
        """Generate new encryption key"""
        return Fernet.generate_key().decode()

# Usage
encryption_manager = EncryptionManager()

# Encrypt sensitive data
credit_card = "4532-1234-5678-9012"
encrypted_card = encryption_manager.encrypt(credit_card)
print(f"Encrypted: {encrypted_card}")

# Decrypt when needed
decrypted_card = encryption_manager.decrypt(encrypted_card)
print(f"Decrypted: {decrypted_card}")

# Save key securely
encryption_key = encryption_manager.key
# Store in environment variable or secrets manager
```

### 2. **AES Encryption**

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding
from cryptography.hazmat.backends import default_backend
import os
import base64

class AESEncryption:
    """AES-256 encryption"""

    def __init__(self, key: bytes):
        if len(key) not in [16, 24, 32]:
            raise ValueError("Key must be 16, 24, or 32 bytes")
        self.key = key
        self.backend = default_backend()

    def encrypt(self, plaintext: str) -> tuple[str, str]:
        """Encrypt plaintext, return (ciphertext, iv)"""
        # Generate random IV
        iv = os.urandom(16)

        # Create cipher
        cipher = Cipher(
            algorithms.AES(self.key),
            modes.CBC(iv),
            backend=self.backend
        )

        # Pad plaintext
        padder = padding.PKCS7(128).padder()
        padded_data = padder.update(plaintext.encode()) + padder.finalize()

        # Encrypt
        encryptor = cipher.encryptor()
        ciphertext = encryptor.update(padded_data) + encryptor.finalize()

        return (
            base64.b64encode(ciphertext).decode(),
            base64.b64encode(iv).decode()
        )

    def decrypt(self, ciphertext: str, iv: str) -> str:
        """Decrypt ciphertext using IV"""
        # Decode
        ciphertext_bytes = base64.b64decode(ciphertext)
        iv_bytes = base64.b64decode(iv)

        # Create cipher
        cipher = Cipher(
            algorithms.AES(self.key),
            modes.CBC(iv_bytes),
            backend=self.backend
        )

        # Decrypt
        decryptor = cipher.decryptor()
        padded_plaintext = decryptor.update(ciphertext_bytes) + decryptor.finalize()

        # Unpad
        unpadder = padding.PKCS7(128).unpadder()
        plaintext = unpadder.update(padded_plaintext) + unpadder.finalize()

        return plaintext.decode()

    @staticmethod
    def generate_key() -> bytes:
        """Generate 256-bit key"""
        return os.urandom(32)

# Usage
key = AESEncryption.generate_key()
aes = AESEncryption(key)

# Encrypt
data = "Sensitive information"
ciphertext, iv = aes.encrypt(data)
print(f"Encrypted: {ciphertext}")
print(f"IV: {iv}")

# Decrypt
decrypted = aes.decrypt(ciphertext, iv)
print(f"Decrypted: {decrypted}")
```

---

## Database Field Encryption

### 1. **Encrypted Model Fields**

```python
from sqlalchemy import Column, Integer, String, Text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.types import TypeDecorator

Base = declarative_base()

class EncryptedString(TypeDecorator):
    """SQLAlchemy type that encrypts/decrypts strings"""

    impl = Text
    cache_ok = True

    def __init__(self, encryption_manager: EncryptionManager):
        super().__init__()
        self.encryption_manager = encryption_manager

    def process_bind_param(self, value, dialect):
        """Encrypt value before storing"""
        if value is not None:
            return self.encryption_manager.encrypt(value)
        return value

    def process_result_value(self, value, dialect):
        """Decrypt value after retrieving"""
        if value is not None:
            return self.encryption_manager.decrypt(value)
        return value

# Initialize encryption manager
encryption_manager = EncryptionManager()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)

    # Encrypted fields
    ssn = Column(EncryptedString(encryption_manager))
    credit_card = Column(EncryptedString(encryption_manager))
    api_key = Column(EncryptedString(encryption_manager))

# Usage
user = User(
    email="john@example.com",
    ssn="123-45-6789",  # Automatically encrypted
    credit_card="4532-1234-5678-9012"
)
db.add(user)
db.commit()

# Retrieval - automatically decrypted
retrieved_user = db.query(User).filter(User.email == "john@example.com").first()
print(retrieved_user.ssn)  # "123-45-6789" (decrypted)
```

### 2. **Hybrid Approach (Hash + Encrypt)**

```python
class SecureUser(Base):
    __tablename__ = "secure_users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)

    # Hash password (one-way)
    password_hash = Column(String)

    # Encrypt sensitive data (two-way)
    ssn_encrypted = Column(Text)
    credit_card_encrypted = Column(Text)

    def set_password(self, password: str):
        """Hash password"""
        self.password_hash = hash_password(password)

    def verify_password(self, password: str) -> bool:
        """Verify password"""
        return verify_password(password, self.password_hash)

    def set_ssn(self, ssn: str, encryption_manager: EncryptionManager):
        """Encrypt SSN"""
        self.ssn_encrypted = encryption_manager.encrypt(ssn)

    def get_ssn(self, encryption_manager: EncryptionManager) -> str:
        """Decrypt SSN"""
        return encryption_manager.decrypt(self.ssn_encrypted)
```

---

## Hashing for Data Integrity

### 1. **File Integrity Verification**

```python
import hashlib
from pathlib import Path

def calculate_file_hash(file_path: str, algorithm: str = "sha256") -> str:
    """Calculate hash of file for integrity verification"""
    hash_func = hashlib.new(algorithm)

    with open(file_path, "rb") as f:
        # Read in chunks for large files
        for chunk in iter(lambda: f.read(4096), b""):
            hash_func.update(chunk)

    return hash_func.hexdigest()

# Usage
file_hash = calculate_file_hash("document.pdf")
print(f"SHA-256: {file_hash}")

# Verify integrity later
def verify_file_integrity(file_path: str, expected_hash: str) -> bool:
    """Verify file hasn't been tampered with"""
    current_hash = calculate_file_hash(file_path)
    return current_hash == expected_hash

is_intact = verify_file_integrity("document.pdf", file_hash)
```

### 2. **Message Authentication (HMAC)**

```python
import hmac
import hashlib

def create_hmac(message: str, secret_key: str) -> str:
    """Create HMAC for message authentication"""
    return hmac.new(
        secret_key.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()

def verify_hmac(message: str, signature: str, secret_key: str) -> bool:
    """Verify HMAC signature"""
    expected_signature = create_hmac(message, secret_key)
    return hmac.compare_digest(signature, expected_signature)

# Usage
secret_key = "my-secret-key"
message = "Important message"

# Create signature
signature = create_hmac(message, secret_key)

# Verify signature
is_valid = verify_hmac(message, signature, secret_key)
print(f"Valid: {is_valid}")

# Tampered message
is_valid_tampered = verify_hmac("Tampered message", signature, secret_key)
print(f"Tampered Valid: {is_valid_tampered}")  # False
```

---

## FastAPI Integration

### 1. **Encrypt API Response Data**

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI()

class SensitiveData(BaseModel):
    user_id: int
    credit_card: str
    ssn: str

class EncryptedResponse(BaseModel):
    encrypted_data: str

def get_encryption_manager() -> EncryptionManager:
    """Dependency for encryption manager"""
    key = os.getenv("ENCRYPTION_KEY")
    return EncryptionManager(key.encode())

@app.get("/api/sensitive/{user_id}", response_model=EncryptedResponse)
async def get_sensitive_data(
    user_id: int,
    encryption: EncryptionManager = Depends(get_encryption_manager)
):
    """Return encrypted sensitive data"""
    # Fetch sensitive data
    data = SensitiveData(
        user_id=user_id,
        credit_card="4532-1234-5678-9012",
        ssn="123-45-6789"
    )

    # Encrypt entire response
    encrypted = encryption.encrypt(data.json())

    return EncryptedResponse(encrypted_data=encrypted)

@app.post("/api/sensitive/decrypt")
async def decrypt_sensitive_data(
    encrypted_response: EncryptedResponse,
    encryption: EncryptionManager = Depends(get_encryption_manager)
):
    """Decrypt sensitive data"""
    decrypted_json = encryption.decrypt(encrypted_response.encrypted_data)
    data = SensitiveData.parse_raw(decrypted_json)
    return data
```

### 2. **Token Signing and Verification**

```python
from jose import jwt, JWTError

def create_signed_token(data: dict, secret_key: str) -> str:
    """Create signed JWT token"""
    return jwt.encode(data, secret_key, algorithm="HS256")

def verify_signed_token(token: str, secret_key: str) -> dict:
    """Verify and decode signed token"""
    try:
        payload = jwt.decode(token, secret_key, algorithms=["HS256"])
        return payload
    except JWTError:
        raise HTTPException(401, "Invalid token signature")

@app.post("/api/create-token")
async def create_token(data: dict):
    """Create signed token"""
    secret_key = os.getenv("JWT_SECRET_KEY")
    token = create_signed_token(data, secret_key)
    return {"token": token}

@app.post("/api/verify-token")
async def verify_token(token: str):
    """Verify token signature"""
    secret_key = os.getenv("JWT_SECRET_KEY")
    payload = verify_signed_token(token, secret_key)
    return {"valid": True, "payload": payload}
```

---

## Key Management

### 1. **Environment Variables**

```python
import os
from dotenv import load_dotenv

# Load from .env file
load_dotenv()

# Get encryption key
ENCRYPTION_KEY = os.getenv("ENCRYPTION_KEY")
if not ENCRYPTION_KEY:
    raise ValueError("ENCRYPTION_KEY not set")

# Get JWT secret
JWT_SECRET_KEY = os.getenv("JWT_SECRET_KEY")
if not JWT_SECRET_KEY:
    raise ValueError("JWT_SECRET_KEY not set")
```

### 2. **Key Rotation**

```python
from datetime import datetime

class KeyManager:
    """Manage multiple encryption keys with rotation"""

    def __init__(self):
        self.keys = {}
        self.active_key_id = None

    def add_key(self, key_id: str, key: bytes, is_active: bool = False):
        """Add encryption key"""
        self.keys[key_id] = {
            "key": key,
            "cipher": Fernet(key),
            "created_at": datetime.utcnow(),
            "is_active": is_active
        }

        if is_active:
            self.active_key_id = key_id

    def encrypt(self, data: str) -> tuple[str, str]:
        """Encrypt with active key"""
        if not self.active_key_id:
            raise ValueError("No active key")

        cipher = self.keys[self.active_key_id]["cipher"]
        encrypted = cipher.encrypt(data.encode())

        # Return encrypted data + key ID
        return (
            base64.urlsafe_b64encode(encrypted).decode(),
            self.active_key_id
        )

    def decrypt(self, encrypted_data: str, key_id: str) -> str:
        """Decrypt with specified key"""
        if key_id not in self.keys:
            raise ValueError(f"Key {key_id} not found")

        cipher = self.keys[key_id]["cipher"]
        encrypted_bytes = base64.urlsafe_b64decode(encrypted_data)
        decrypted = cipher.decrypt(encrypted_bytes)

        return decrypted.decode()

    def rotate_key(self, new_key: bytes):
        """Rotate to new active key"""
        new_key_id = f"key_{datetime.utcnow().timestamp()}"

        # Mark old key as inactive
        if self.active_key_id:
            self.keys[self.active_key_id]["is_active"] = False

        # Add new active key
        self.add_key(new_key_id, new_key, is_active=True)

        return new_key_id

# Usage
key_manager = KeyManager()

# Add initial key
key1 = Fernet.generate_key()
key_manager.add_key("key_2024_01", key1, is_active=True)

# Encrypt data
encrypted, key_id = key_manager.encrypt("Sensitive data")
print(f"Encrypted with key: {key_id}")

# Rotate key
key2 = Fernet.generate_key()
new_key_id = key_manager.rotate_key(key2)
print(f"Rotated to new key: {new_key_id}")

# Old data can still be decrypted with old key
decrypted = key_manager.decrypt(encrypted, key_id)
print(f"Decrypted: {decrypted}")

# New data encrypted with new key
new_encrypted, new_key_id = key_manager.encrypt("New sensitive data")
```

---

## Best Practices

### ✅ Do's:

1. **Use bcrypt or Argon2** for passwords
2. **Use Fernet or AES** for data encryption
3. **Store keys securely** (environment variables, secrets manager)
4. **Rotate keys** regularly
5. **Use HMAC** for message authentication
6. **Encrypt sensitive data** at rest
7. **Use HTTPS** for data in transit
8. **Hash before comparing** (timing-safe)

### ❌ Don'ts:

1. **Don't hash with MD5 or SHA1** for passwords
2. **Don't store keys** in code or database
3. **Don't reuse encryption keys** across environments
4. **Don't encrypt passwords** (hash instead)
5. **Don't roll your own** crypto
6. **Don't use ECB mode** for AES
7. **Don't forget IV/salt** for encryption

---

## Interview Questions

### Q1: What's the difference between hashing and encryption?

**Answer**:

- **Hashing**: One-way function, cannot be reversed, used for passwords and integrity checks
- **Encryption**: Two-way function, can be decrypted with key, used for protecting confidential data

### Q2: Why use bcrypt instead of SHA-256 for passwords?

**Answer**: Bcrypt is designed for password hashing with:

- Built-in salt
- Configurable cost factor (slow by design)
- Resistant to rainbow tables and brute force
  SHA-256 is too fast and lacks these security features.

### Q3: When should you encrypt data vs hash it?

**Answer**:

- **Hash**: Passwords, checksums, when you never need original value
- **Encrypt**: Credit cards, SSN, PII, when you need to retrieve original value

### Q4: What is a salt in password hashing?

**Answer**: Random data added to password before hashing to ensure same password produces different hashes. Prevents rainbow table attacks. Bcrypt includes automatic salting.

### Q5: How do you securely store encryption keys?

**Answer**:

- Environment variables (dev/staging)
- Cloud secrets managers (AWS Secrets Manager, Azure Key Vault)
- Hardware Security Modules (HSMs) for production
- Never in code or database

---

## Summary

Data encryption and hashing provide:

- **Password security** with bcrypt/Argon2
- **Data confidentiality** with Fernet/AES
- **Data integrity** with SHA-256/HMAC
- **Message authentication** with digital signatures
- **Key rotation** for long-term security

Properly implemented encryption and hashing are essential for protecting sensitive data in FastAPI applications! 🔐
