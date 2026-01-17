# 🛡️ Input Validation and Sanitization

## Overview

Input validation is the first line of defense against injection attacks, data corruption, and security vulnerabilities. Proper validation ensures data integrity and application security.

---

## Validation Strategies

### 1. **Whitelist vs Blacklist**

```python
from typing import Optional
import re

# ❌ BLACKLIST: Easy to bypass
FORBIDDEN_CHARS = ["<", ">", "'", "\"", "--", ";"]

def blacklist_validation(user_input: str) -> bool:
    """Blacklist approach - NOT RECOMMENDED"""
    for char in FORBIDDEN_CHARS:
        if char in user_input:
            return False
    return True

# ✅ WHITELIST: Much more secure
ALLOWED_USERNAME_PATTERN = r"^[a-zA-Z0-9_-]{3,20}$"

def whitelist_validation(username: str) -> bool:
    """Whitelist approach - RECOMMENDED"""
    return bool(re.match(ALLOWED_USERNAME_PATTERN, username))

# Example usage
whitelist_validation("john_doe")  # True
whitelist_validation("john<script>")  # False
```

### 2. **Multi-Layer Validation**

```python
from pydantic import BaseModel, Field, validator, root_validator
from typing import Optional
from datetime import date

class UserRegistration(BaseModel):
    """Multi-layer validation"""

    # Layer 1: Field-level validation
    username: str = Field(
        ...,
        min_length=3,
        max_length=20,
        regex=r"^[a-zA-Z0-9_-]+$",
        description="Alphanumeric username"
    )

    email: str = Field(
        ...,
        regex=r"^[\w\.-]+@[\w\.-]+\.\w{2,}$"
    )

    password: str = Field(..., min_length=12)
    password_confirm: str

    age: int = Field(..., ge=18, le=120)

    phone: Optional[str] = Field(
        None,
        regex=r"^\+?1?\d{10,15}$"
    )

    website: Optional[str] = Field(
        None,
        regex=r"^https?://[\w\.-]+\.\w{2,}(/.*)?$"
    )

    # Layer 2: Custom field validators
    @validator("username")
    def username_no_profanity(cls, v):
        """Check against profanity list"""
        profanity_list = ["badword1", "badword2"]
        if v.lower() in profanity_list:
            raise ValueError("Username contains inappropriate content")
        return v

    @validator("email")
    def email_domain_allowed(cls, v):
        """Check if email domain is allowed"""
        blocked_domains = ["tempmail.com", "throwaway.email"]
        domain = v.split("@")[1]
        if domain in blocked_domains:
            raise ValueError("Email domain not allowed")
        return v

    @validator("password")
    def password_strength(cls, v):
        """Validate password strength"""
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain uppercase letter")
        if not any(c.islower() for c in v):
            raise ValueError("Password must contain lowercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain digit")
        if not any(c in "!@#$%^&*()_+-=[]{}|;:,.<>?" for c in v):
            raise ValueError("Password must contain special character")
        return v

    @validator("age")
    def age_reasonable(cls, v):
        """Additional age validation"""
        if v > 100:
            raise ValueError("Please verify age")
        return v

    # Layer 3: Cross-field validation
    @root_validator
    def passwords_match(cls, values):
        """Ensure passwords match"""
        pw1 = values.get("password")
        pw2 = values.get("password_confirm")

        if pw1 != pw2:
            raise ValueError("Passwords do not match")

        return values

    @root_validator
    def username_not_in_email(cls, values):
        """Prevent username revealing email"""
        username = values.get("username")
        email = values.get("email")

        if username and email and username.lower() in email.lower():
            raise ValueError("Username should not be part of email")

        return values

# Usage
try:
    user = UserRegistration(
        username="john_doe",
        email="john@example.com",
        password="SecurePass123!",
        password_confirm="SecurePass123!",
        age=25
    )
except ValueError as e:
    print(f"Validation error: {e}")
```

---

## SQL Injection Prevention

### Vulnerable vs Secure

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

# ❌ VULNERABLE: String concatenation
def get_user_vulnerable(db: Session, username: str):
    """NEVER DO THIS"""
    query = f"SELECT * FROM users WHERE username = '{username}'"
    # Attacker can inject: ' OR '1'='1
    result = db.execute(text(query))
    return result.fetchall()

# ✅ SECURE: Parameterized queries
def get_user_secure(db: Session, username: str):
    """Use parameterized queries"""
    query = text("SELECT * FROM users WHERE username = :username")
    result = db.execute(query, {"username": username})
    return result.fetchall()

# ✅ SECURE: ORM (best approach)
def get_user_orm(db: Session, username: str):
    """Use ORM - automatically safe"""
    return db.query(User).filter(User.username == username).first()

# ✅ SECURE: Dynamic WHERE conditions
def search_users(
    db: Session,
    filters: dict[str, any]
) -> list[User]:
    """Dynamic filters safely"""
    query = db.query(User)

    # Build conditions safely
    if "username" in filters:
        query = query.filter(User.username.like(f"%{filters['username']}%"))

    if "min_age" in filters:
        query = query.filter(User.age >= filters["min_age"])

    if "email_domain" in filters:
        query = query.filter(User.email.like(f"%@{filters['email_domain']}"))

    return query.all()
```

### NoSQL Injection Prevention

```python
from pymongo import MongoClient
import re

# ❌ VULNERABLE: Direct user input in query
def find_user_vulnerable(username: str, password: str):
    """Vulnerable to NoSQL injection"""
    client = MongoClient()
    db = client.mydb

    # Attacker can pass: {"$ne": null}
    user = db.users.find_one({
        "username": username,
        "password": password
    })
    return user

# ✅ SECURE: Validate and sanitize
def find_user_secure(username: str, password: str):
    """Secure NoSQL query"""
    client = MongoClient()
    db = client.mydb

    # Validate types
    if not isinstance(username, str) or not isinstance(password, str):
        raise ValueError("Invalid input types")

    # Escape special MongoDB operators
    username = re.sub(r'[${}]', '', username)
    password = re.sub(r'[${}]', '', password)

    # Use exact match
    user = db.users.find_one({
        "username": {"$eq": username},
        "password": {"$eq": password}
    })

    return user
```

---

## XSS Prevention

### HTML Sanitization

```python
import bleach
from markupsafe import escape

# ❌ VULNERABLE: No sanitization
@app.post("/comments")
async def create_comment_vulnerable(content: str):
    """Stores and displays raw HTML"""
    save_comment(content)
    # If displayed without escaping: XSS vulnerability
    return {"content": content}

# ✅ SECURE: Escape HTML
@app.post("/comments")
async def create_comment_escaped(content: str):
    """Escape all HTML"""
    safe_content = escape(content)
    save_comment(safe_content)
    return {"content": safe_content}

# ✅ SECURE: Allow limited HTML with bleach
ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li']
ALLOWED_ATTRIBUTES = {'a': ['href', 'title']}

def sanitize_html(content: str) -> str:
    """Allow only safe HTML tags"""
    return bleach.clean(
        content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        strip=True
    )

@app.post("/posts")
async def create_post(content: str):
    """Allow limited safe HTML"""
    safe_content = sanitize_html(content)
    save_post(safe_content)
    return {"content": safe_content}

# ✅ SECURE: Markdown to HTML
import markdown

@app.post("/articles")
async def create_article(markdown_content: str):
    """Convert markdown safely"""
    # Sanitize markdown first
    safe_markdown = bleach.clean(markdown_content)

    # Convert to HTML
    html_content = markdown.markdown(
        safe_markdown,
        extensions=['extra', 'codehilite']
    )

    # Sanitize HTML output
    safe_html = bleach.clean(
        html_content,
        tags=ALLOWED_TAGS + ['code', 'pre'],
        attributes={**ALLOWED_ATTRIBUTES, 'code': ['class']}
    )

    save_article(safe_html)
    return {"content": safe_html}
```

---

## File Upload Validation

### Comprehensive File Validation

```python
import magic
from pathlib import Path
import hashlib
from PIL import Image

class FileValidator:
    """Secure file upload validation"""

    ALLOWED_MIME_TYPES = {
        "image/jpeg",
        "image/png",
        "image/gif",
        "application/pdf"
    }

    ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".gif", ".pdf"}

    MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB
    MAX_IMAGE_PIXELS = 50_000_000  # 50 megapixels

    @classmethod
    def validate_file(
        cls,
        file: UploadFile,
        allowed_types: set[str] = None
    ) -> tuple[bool, str]:
        """Comprehensive file validation"""
        allowed_types = allowed_types or cls.ALLOWED_MIME_TYPES

        # 1. Check file extension
        ext = Path(file.filename).suffix.lower()
        if ext not in cls.ALLOWED_EXTENSIONS:
            return False, f"Extension {ext} not allowed"

        # 2. Check file size
        file.file.seek(0, 2)  # Seek to end
        size = file.file.tell()
        file.file.seek(0)  # Reset

        if size > cls.MAX_FILE_SIZE:
            return False, f"File too large: {size} bytes"

        if size == 0:
            return False, "Empty file"

        # 3. Check MIME type from content
        file_content = file.file.read(2048)
        file.file.seek(0)

        mime = magic.from_buffer(file_content, mime=True)
        if mime not in allowed_types:
            return False, f"Invalid file type: {mime}"

        # 4. Additional image validation
        if mime.startswith("image/"):
            try:
                img = Image.open(file.file)

                # Check image dimensions
                pixels = img.width * img.height
                if pixels > cls.MAX_IMAGE_PIXELS:
                    return False, "Image resolution too high"

                # Verify image format matches extension
                expected_format = {
                    ".jpg": "JPEG",
                    ".jpeg": "JPEG",
                    ".png": "PNG",
                    ".gif": "GIF"
                }

                if img.format != expected_format.get(ext):
                    return False, "File extension doesn't match content"

                file.file.seek(0)
            except Exception as e:
                return False, f"Invalid image file: {str(e)}"

        return True, "Valid"

    @staticmethod
    def generate_safe_filename(original_filename: str) -> str:
        """Generate safe, unique filename"""
        # Remove path separators and dangerous characters
        safe_name = re.sub(r'[^\w\.-]', '_', original_filename)

        # Limit length
        name_part = safe_name[:50]

        # Add hash for uniqueness
        timestamp = datetime.utcnow().isoformat()
        hash_input = f"{name_part}{timestamp}"
        file_hash = hashlib.md5(hash_input.encode()).hexdigest()[:8]

        # Preserve extension
        ext = Path(safe_name).suffix
        return f"{file_hash}_{name_part}{ext}"

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    """Secure file upload"""
    # Validate file
    is_valid, message = FileValidator.validate_file(file)

    if not is_valid:
        raise HTTPException(400, message)

    # Generate safe filename
    safe_filename = FileValidator.generate_safe_filename(file.filename)

    # Save to secure location (outside webroot)
    upload_dir = Path("/secure/uploads")
    upload_dir.mkdir(parents=True, exist_ok=True)

    file_path = upload_dir / safe_filename

    # Save file
    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)

    return {
        "filename": safe_filename,
        "size": len(content),
        "mime_type": magic.from_buffer(content, mime=True)
    }
```

---

## Command Injection Prevention

```python
import subprocess
import shlex

# ❌ VULNERABLE: Shell injection
def run_command_vulnerable(user_input: str):
    """NEVER DO THIS"""
    command = f"ls {user_input}"
    # Attacker can inject: "; rm -rf /"
    subprocess.run(command, shell=True)

# ✅ SECURE: Use list arguments, no shell
def run_command_secure(directory: str):
    """Safe command execution"""
    # Validate input first
    if not re.match(r'^[a-zA-Z0-9_/.-]+$', directory):
        raise ValueError("Invalid directory path")

    # Use list, don't use shell
    subprocess.run(
        ["ls", "-la", directory],
        shell=False,
        capture_output=True,
        text=True,
        timeout=5
    )

# ✅ SECURE: Escape shell arguments if shell is needed
def run_with_shell_escaped(user_input: str):
    """If shell is required, escape properly"""
    escaped = shlex.quote(user_input)
    command = f"echo {escaped}"

    subprocess.run(
        command,
        shell=True,
        timeout=5
    )
```

---

## JSON Validation

```python
from pydantic import BaseModel, Field, validator
from typing import Optional, Literal
from enum import Enum

class OrderStatus(str, Enum):
    """Enum for allowed values"""
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"

class Address(BaseModel):
    """Nested validation"""
    street: str = Field(..., min_length=5, max_length=100)
    city: str = Field(..., min_length=2, max_length=50)
    state: str = Field(..., regex=r'^[A-Z]{2}$')
    zip_code: str = Field(..., regex=r'^\d{5}(-\d{4})?$')
    country: Literal["US", "CA", "MX"] = "US"

class OrderItem(BaseModel):
    """Order item validation"""
    product_id: int = Field(..., gt=0)
    quantity: int = Field(..., gt=0, le=100)
    price: float = Field(..., gt=0, le=10000)

    @validator("price")
    def price_two_decimals(cls, v):
        """Ensure price has max 2 decimal places"""
        if round(v, 2) != v:
            raise ValueError("Price must have at most 2 decimal places")
        return v

class Order(BaseModel):
    """Complete order validation"""
    customer_id: int = Field(..., gt=0)
    items: list[OrderItem] = Field(..., min_items=1, max_items=50)
    shipping_address: Address
    billing_address: Optional[Address] = None
    status: OrderStatus = OrderStatus.PENDING
    notes: Optional[str] = Field(None, max_length=500)

    @root_validator
    def validate_order_total(cls, values):
        """Calculate and validate total"""
        items = values.get("items", [])
        total = sum(item.price * item.quantity for item in items)

        if total > 50000:
            raise ValueError("Order total exceeds maximum allowed")

        values["total"] = total
        return values

@app.post("/orders")
async def create_order(order: Order):
    """Fully validated order creation"""
    # All validation done by Pydantic
    save_order(order)
    return {"order_id": 12345, "total": order.total}
```

---

## Best Practices

### ✅ Do's:

1. **Whitelist validation** over blacklist
2. **Validate at multiple layers** (client, API, database)
3. **Use parameterized queries**
4. **Sanitize HTML output**
5. **Validate file content**, not just extension
6. **Use strong typing** (Pydantic)
7. **Set reasonable limits** (size, length, range)
8. **Log validation failures**

### ❌ Don'ts:

1. **Don't trust client-side validation**
2. **Don't use string concatenation** in queries
3. **Don't allow unlimited file sizes**
4. **Don't execute user input** as code
5. **Don't rely on file extensions**
6. **Don't skip sanitization** before display
7. **Don't use blacklist** for security

---

## Interview Questions

### Q1: What's the difference between validation and sanitization?

**Answer**:

- **Validation**: Check if input meets requirements (reject if invalid)
- **Sanitization**: Clean/modify input to make it safe (transform input)
  Example: Validation rejects `<script>`, sanitization removes/escapes it.

### Q2: How do you prevent SQL injection?

**Answer**:

- Use parameterized queries/prepared statements
- Use ORM (SQLAlchemy, Django ORM)
- Never concatenate user input into SQL
- Validate input types and format
- Use least privilege database accounts

### Q3: What is whitelist vs blacklist validation?

**Answer**:

- **Whitelist**: Define what IS allowed (more secure)
- **Blacklist**: Define what is NOT allowed (easy to bypass)
  Always prefer whitelist - attackers find new ways to bypass blacklists.

### Q4: How do you validate file uploads securely?

**Answer**:

- Check file size
- Validate MIME type from content (not extension)
- Scan file content
- Rename files (don't trust user filenames)
- Store outside webroot
- Set execution permissions correctly

### Q5: What is the principle of least privilege?

**Answer**: Grant minimum permissions needed:

- Database users with read-only where possible
- File system permissions restricted
- API users with limited scopes
- Reduces blast radius of security breach

---

## Summary

Input validation layers:

- **Whitelist over blacklist**
- **Multi-layer validation** (field, custom, cross-field)
- **Parameterized queries** for SQL
- **HTML sanitization** for XSS prevention
- **File content validation** beyond extensions
- **Type safety** with Pydantic
- **Reasonable limits** on all inputs

Validation is your first defense - never skip it! 🛡️
