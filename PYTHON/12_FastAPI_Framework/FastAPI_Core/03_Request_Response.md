# FastAPI Core: Request & Response

## 📖 Introduction

FastAPI provides powerful tools for handling HTTP requests and responses. You can access request headers, cookies, form data, files, and customize response status codes, headers, and media types.

## 📨 Request Object

### Accessing Request Object

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/headers")
async def get_headers(request: Request):
    """
    Access request object

    Request provides access to:
    - Headers
    - Cookies
    - Client info
    - URL
    - Method
    """
    return {
        "headers": dict(request.headers),
        "method": request.method,
        "url": str(request.url),
        "client": request.client.host if request.client else None,
        "cookies": request.cookies
    }
```

### Request Headers

```python
from fastapi import Header
from typing import Optional

@app.get("/user-agent")
async def read_user_agent(user_agent: Optional[str] = Header(None)):
    """
    Get User-Agent header

    FastAPI converts 'User-Agent' -> 'user_agent'
    """
    return {"User-Agent": user_agent}

# Multiple headers
@app.get("/headers-info")
async def read_headers(
    user_agent: Optional[str] = Header(None),
    accept_language: Optional[str] = Header(None),
    x_request_id: Optional[str] = Header(None)
):
    """
    Multiple headers

    Header names converted: X-Request-ID -> x_request_id
    """
    return {
        "user_agent": user_agent,
        "accept_language": accept_language,
        "x_request_id": x_request_id
    }

# Custom header validation
@app.get("/api-key")
async def check_api_key(
    x_api_key: str = Header(
        ...,
        description="API Key for authentication",
        min_length=32,
        max_length=64
    )
):
    """Header with validation"""
    return {"api_key_length": len(x_api_key)}
```

### Request Body - JSON

```python
from pydantic import BaseModel
from typing import List, Dict, Any

class Item(BaseModel):
    """Item model"""
    name: str
    description: Optional[str] = None
    price: float
    tags: List[str] = []

@app.post("/items")
async def create_item(item: Item):
    """
    JSON request body

    Content-Type: application/json
    """
    return {"item": item, "price_with_tax": item.price * 1.1}

# Nested models
class Image(BaseModel):
    """Image model"""
    url: str
    name: str

class ItemWithImages(BaseModel):
    """Item with images"""
    name: str
    images: List[Image] = []

@app.post("/items-with-images")
async def create_item_with_images(item: ItemWithImages):
    """Nested Pydantic models"""
    return item

# Arbitrary dict
@app.post("/data")
async def receive_data(data: Dict[str, Any]):
    """
    Accept arbitrary JSON object

    No validation of structure
    """
    return {"received": data}
```

### Request Body - Form Data

```python
from fastapi import Form

@app.post("/login")
async def login(
    username: str = Form(...),
    password: str = Form(...)
):
    """
    Form data (application/x-www-form-urlencoded)

    Install: pip install python-multipart

    Content-Type: application/x-www-form-urlencoded
    """
    return {"username": username}

# Form with optional fields
@app.post("/contact")
async def contact_form(
    name: str = Form(...),
    email: str = Form(...),
    message: str = Form(...),
    phone: Optional[str] = Form(None)
):
    """Contact form with optional field"""
    return {
        "name": name,
        "email": email,
        "message": message,
        "phone": phone
    }
```

### File Uploads

```python
from fastapi import File, UploadFile
from typing import List

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    """
    Upload single file

    UploadFile benefits:
    - Stored in memory up to limit, then disk
    - Has file-like interface
    - Async methods (read, write, seek)
    """
    contents = await file.read()

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents)
    }

# Multiple files
@app.post("/upload-multiple")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    """Upload multiple files"""
    results = []

    for file in files:
        contents = await file.read()
        results.append({
            "filename": file.filename,
            "size": len(contents)
        })

    return {"files": results}

# File with form data
@app.post("/upload-with-data")
async def upload_with_data(
    file: UploadFile = File(...),
    description: str = Form(...),
    tags: str = Form(None)
):
    """File upload with additional form data"""
    return {
        "filename": file.filename,
        "description": description,
        "tags": tags
    }

# Save uploaded file
@app.post("/save-file")
async def save_file(file: UploadFile = File(...)):
    """Save uploaded file to disk"""
    file_path = f"uploads/{file.filename}"

    with open(file_path, "wb") as f:
        contents = await file.read()
        f.write(contents)

    return {"filename": file.filename, "path": file_path}
```

### Cookies

```python
from fastapi import Cookie

@app.get("/cookie")
async def read_cookie(session_id: Optional[str] = Cookie(None)):
    """
    Read cookie

    Cookie name: session_id
    """
    return {"session_id": session_id}

# Multiple cookies
@app.get("/cookies")
async def read_cookies(
    session_id: Optional[str] = Cookie(None),
    tracking_id: Optional[str] = Cookie(None)
):
    """Read multiple cookies"""
    return {
        "session_id": session_id,
        "tracking_id": tracking_id
    }
```

## 📤 Response Customization

### Response Status Code

```python
from fastapi import status

@app.post("/items", status_code=status.HTTP_201_CREATED)
async def create_item(item: Item):
    """
    Custom status code

    Use status.HTTP_* constants for readability
    """
    return item

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    """
    204 No Content - no response body

    Return None or empty Response
    """
    # Delete item from database
    return None

# Dynamic status code
from fastapi import Response

@app.put("/items/{item_id}")
async def update_or_create_item(item_id: int, item: Item, response: Response):
    """
    Dynamic status code

    - 200 if updated
    - 201 if created
    """
    # Check if exists
    exists = False  # Check database

    if exists:
        response.status_code = status.HTTP_200_OK
    else:
        response.status_code = status.HTTP_201_CREATED

    return item
```

### Response Headers

```python
from fastapi import Response
from fastapi.responses import JSONResponse

@app.get("/headers")
async def set_headers(response: Response):
    """
    Set response headers

    Modify response object directly
    """
    response.headers["X-Custom-Header"] = "Value"
    response.headers["X-Request-ID"] = "123456"

    return {"message": "Headers set"}

# JSONResponse with headers
@app.get("/headers-json")
async def set_headers_json():
    """
    JSONResponse with custom headers

    Return JSONResponse directly
    """
    content = {"message": "Hello"}
    headers = {
        "X-Custom-Header": "Value",
        "X-Request-ID": "123456"
    }

    return JSONResponse(content=content, headers=headers)
```

### Set Cookies

```python
from fastapi.responses import JSONResponse

@app.post("/login")
async def login(username: str, password: str):
    """
    Set cookie in response

    Cookie attributes:
    - max_age: seconds until expiry
    - expires: absolute expiry datetime
    - httponly: not accessible via JavaScript
    - secure: only sent over HTTPS
    - samesite: CSRF protection
    """
    # Validate credentials
    if username == "admin" and password == "secret":
        response = JSONResponse(content={"message": "Login successful"})

        response.set_cookie(
            key="session_id",
            value="abc123",
            max_age=3600,  # 1 hour
            httponly=True,
            secure=True,
            samesite="lax"
        )

        return response

    return {"message": "Invalid credentials"}

@app.post("/logout")
async def logout():
    """Delete cookie"""
    response = JSONResponse(content={"message": "Logged out"})
    response.delete_cookie(key="session_id")

    return response
```

## 📄 Response Types

### JSONResponse (Default)

```python
from fastapi.responses import JSONResponse

@app.get("/items")
async def list_items():
    """
    Default response type

    Automatically serializes to JSON
    """
    return {"items": [1, 2, 3]}

# Explicit JSONResponse
@app.get("/items-explicit")
async def list_items_explicit():
    """Explicit JSONResponse"""
    return JSONResponse(
        content={"items": [1, 2, 3]},
        status_code=200,
        headers={"X-Custom": "Header"}
    )
```

### HTMLResponse

```python
from fastapi.responses import HTMLResponse

@app.get("/html", response_class=HTMLResponse)
async def get_html():
    """
    Return HTML

    Content-Type: text/html
    """
    html_content = """
    <!DOCTYPE html>
    <html>
        <head>
            <title>FastAPI</title>
        </head>
        <body>
            <h1>Hello from FastAPI</h1>
        </body>
    </html>
    """
    return HTMLResponse(content=html_content)
```

### PlainTextResponse

```python
from fastapi.responses import PlainTextResponse

@app.get("/text", response_class=PlainTextResponse)
async def get_text():
    """
    Return plain text

    Content-Type: text/plain
    """
    return "Hello, World!"
```

### RedirectResponse

```python
from fastapi.responses import RedirectResponse

@app.get("/redirect")
async def redirect():
    """
    Redirect to another URL

    Status code: 307 (Temporary Redirect)
    """
    return RedirectResponse(url="/new-url")

@app.get("/old-url")
async def old_url():
    """Permanent redirect"""
    return RedirectResponse(
        url="/new-url",
        status_code=status.HTTP_301_MOVED_PERMANENTLY
    )
```

### FileResponse

```python
from fastapi.responses import FileResponse

@app.get("/download")
async def download_file():
    """
    Download file

    Content-Disposition: attachment
    """
    file_path = "files/document.pdf"

    return FileResponse(
        path=file_path,
        filename="download.pdf",
        media_type="application/pdf"
    )

# Inline file (display in browser)
@app.get("/view")
async def view_file():
    """View file inline"""
    return FileResponse(
        path="files/image.jpg",
        media_type="image/jpeg"
    )
```

### StreamingResponse

```python
from fastapi.responses import StreamingResponse
import io
import csv

@app.get("/stream-csv")
async def stream_csv():
    """
    Stream CSV file

    - Generates data on-the-fly
    - Low memory usage for large files
    """
    def generate_csv():
        output = io.StringIO()
        writer = csv.writer(output)

        # Write header
        writer.writerow(["id", "name", "price"])
        yield output.getvalue()
        output.seek(0)
        output.truncate(0)

        # Write rows
        for i in range(1000):
            writer.writerow([i, f"Item {i}", i * 10.0])
            yield output.getvalue()
            output.seek(0)
            output.truncate(0)

    return StreamingResponse(
        generate_csv(),
        media_type="text/csv",
        headers={
            "Content-Disposition": "attachment; filename=items.csv"
        }
    )

# Stream from generator
@app.get("/stream")
async def stream_data():
    """Stream data from async generator"""
    async def generate():
        for i in range(10):
            yield f"Chunk {i}\n"
            await asyncio.sleep(0.1)

    return StreamingResponse(generate(), media_type="text/plain")
```

## 🎯 Response Models

### Response Model with Filtering

```python
from pydantic import BaseModel

class UserIn(BaseModel):
    """User input with password"""
    username: str
    password: str
    email: str

class UserOut(BaseModel):
    """User output without password"""
    username: str
    email: str

@app.post("/users", response_model=UserOut)
async def create_user(user: UserIn):
    """
    Response model filters password

    - Input includes password
    - Output excludes password
    - Automatic filtering by FastAPI
    """
    # Save user to database
    return user  # Password automatically excluded
```

### Multiple Response Models

```python
from fastapi import HTTPException
from typing import Union

class ErrorResponse(BaseModel):
    """Error response"""
    detail: str

class SuccessResponse(BaseModel):
    """Success response"""
    message: str
    data: dict

@app.get(
    "/items/{item_id}",
    response_model=Union[SuccessResponse, ErrorResponse],
    responses={
        200: {"model": SuccessResponse},
        404: {"model": ErrorResponse}
    }
)
async def read_item(item_id: int):
    """
    Multiple response models

    - Different models for different status codes
    - Documents all possible responses
    """
    if item_id < 0:
        raise HTTPException(status_code=404, detail="Item not found")

    return SuccessResponse(
        message="Success",
        data={"item_id": item_id}
    )
```

## ❓ Interview Questions

### Q1: How to access request headers in FastAPI?

**Answer**:

- **Header parameter**: `user_agent: str = Header(None)`
- **Request object**: `request.headers`
- **FastAPI converts** header names (User-Agent → user_agent)

### Q2: What is the difference between File and UploadFile?

**Answer**:

- **File**: Bytes in memory, simple for small files
- **UploadFile**: Spooled file (memory → disk), async methods
- **UploadFile better** for large files (saves memory)

### Q3: How to set custom response headers and cookies?

**Answer**:

- **Headers**: `response.headers["X-Custom"] = "Value"`
- **Cookies**: `response.set_cookie(key="name", value="value")`
- **JSONResponse**: Return with custom headers/cookies

### Q4: When to use StreamingResponse?

**Answer**:

- **Large files**: Stream data in chunks
- **Generated content**: CSV, logs, reports
- **Real-time data**: Server-sent events
- **Low memory**: Don't load entire file into memory

## 📚 Summary

**Key Takeaways**:

1. **Request object** provides access to headers, cookies, client info
2. **Header()** extracts request headers with validation
3. **Form()** handles form data (application/x-www-form-urlencoded)
4. **File() and UploadFile** for file uploads
5. **Cookie()** reads cookies, response.set_cookie() sets them
6. **Response types**: JSON, HTML, PlainText, File, Streaming
7. **Status codes** with status.HTTP\_\* constants
8. **Response headers** via response.headers or JSONResponse
9. **response_model** filters and validates output
10. **StreamingResponse** for large/generated content

FastAPI makes request/response handling intuitive and type-safe!
