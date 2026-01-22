# Django Forms

## 📖 Concept Explanation

Django Forms handle user input, validation, and rendering. They provide a secure way to accept data from users, validate it, and save it to the database.

### Form Processing Flow

```
GET Request → Render Empty Form
POST Request → Validate Data → Save/Error
```

**Process**:

1. User requests form (GET) → Display empty form
2. User submits form (POST) → Validate data
3. If valid → Process/save data, redirect
4. If invalid → Re-render form with errors

### Basic Form

```python
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)

    def clean_email(self):
        """Custom validation for email field"""
        email = self.cleaned_data.get('email')
        if not email.endswith('@company.com'):
            raise forms.ValidationError("Must use company email")
        return email
```

### View Handling Form

```python
from django.shortcuts import render, redirect

def contact_view(request):
    if request.method == 'POST':
        form = ContactForm(request.POST)
        if form.is_valid():
            # Process form data
            name = form.cleaned_data['name']
            email = form.cleaned_data['email']
            message = form.cleaned_data['message']
            # Send email, save to DB, etc.
            return redirect('thank_you')
    else:
        form = ContactForm()

    return render(request, 'contact.html', {'form': form})
```

## 🧠 Forms vs ModelForms

### 1. Regular Form

```python
from django import forms

class UserRegistrationForm(forms.Form):
    """Manual form definition"""
    username = forms.CharField(max_length=150)
    email = forms.EmailField()
    password = forms.CharField(widget=forms.PasswordInput)
    password_confirm = forms.CharField(widget=forms.PasswordInput, label="Confirm Password")

    def clean(self):
        """Cross-field validation"""
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        password_confirm = cleaned_data.get('password_confirm')

        if password != password_confirm:
            raise forms.ValidationError("Passwords don't match")

        return cleaned_data

    def save(self):
        """Manual save logic"""
        user = User.objects.create_user(
            username=self.cleaned_data['username'],
            email=self.cleaned_data['email'],
            password=self.cleaned_data['password']
        )
        return user
```

### 2. ModelForm (Recommended)

```python
from django import forms
from django.contrib.auth.models import User
from .models import Post

class PostForm(forms.ModelForm):
    """Form automatically generated from model"""

    class Meta:
        model = Post
        fields = ['title', 'content', 'tags', 'published']
        # Or exclude fields
        # exclude = ['author', 'created_at']

        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter post title'
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 10
            }),
            'tags': forms.CheckboxSelectMultiple(),
        }

        labels = {
            'title': 'Post Title',
            'content': 'Post Content',
        }

        help_texts = {
            'title': 'Choose a descriptive title',
        }

    def clean_title(self):
        """Field-level validation"""
        title = self.cleaned_data.get('title')
        if Post.objects.filter(title=title).exists():
            raise forms.ValidationError("Post with this title already exists")
        return title
```

## 🏗️ Form Fields & Widgets

### 1. Common Form Fields

```python
from django import forms

class ComprehensiveForm(forms.Form):
    # Text inputs
    text_field = forms.CharField(max_length=200)
    email_field = forms.EmailField()
    url_field = forms.URLField()
    slug_field = forms.SlugField()

    # Text area
    content = forms.CharField(widget=forms.Textarea)

    # Numbers
    integer_field = forms.IntegerField(min_value=0, max_value=100)
    decimal_field = forms.DecimalField(max_digits=10, decimal_places=2)
    float_field = forms.FloatField()

    # Boolean
    agree_terms = forms.BooleanField(required=True)

    # Choices
    CHOICES = [
        ('option1', 'Option 1'),
        ('option2', 'Option 2'),
    ]
    choice_field = forms.ChoiceField(choices=CHOICES)
    multiple_choice = forms.MultipleChoiceField(choices=CHOICES)

    # Date/Time
    date_field = forms.DateField(widget=forms.DateInput(attrs={'type': 'date'}))
    time_field = forms.TimeField()
    datetime_field = forms.DateTimeField()

    # Files
    file_field = forms.FileField()
    image_field = forms.ImageField()

    # Related fields
    category = forms.ModelChoiceField(queryset=Category.objects.all())
    tags = forms.ModelMultipleChoiceField(queryset=Tag.objects.all())
```

### 2. Custom Widgets

```python
from django import forms

class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = '__all__'
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter title',
                'maxlength': '200',
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 10,
                'cols': 80,
            }),
            'published_date': forms.DateInput(attrs={
                'type': 'date',
                'class': 'form-control',
            }),
            'category': forms.Select(attrs={
                'class': 'form-select',
            }),
            'tags': forms.CheckboxSelectMultiple(),
        }
```

## 🎯 Form Validation

### 1. Field-Level Validation

```python
class UserProfileForm(forms.ModelForm):
    class Meta:
        model = UserProfile
        fields = ['username', 'email', 'age', 'website']

    def clean_username(self):
        """Validate username"""
        username = self.cleaned_data.get('username')

        if len(username) < 3:
            raise forms.ValidationError("Username must be at least 3 characters")

        if not username.isalnum():
            raise forms.ValidationError("Username must be alphanumeric")

        # Check uniqueness (exclude current instance)
        if UserProfile.objects.filter(username=username).exclude(pk=self.instance.pk).exists():
            raise forms.ValidationError("Username already taken")

        return username

    def clean_age(self):
        """Validate age"""
        age = self.cleaned_data.get('age')
        if age < 18:
            raise forms.ValidationError("You must be at least 18 years old")
        return age

    def clean_website(self):
        """Validate website URL"""
        website = self.cleaned_data.get('website')
        if website and not website.startswith(('http://', 'https://')):
            website = f'https://{website}'
        return website
```

### 2. Form-Level Validation (Cross-Field)

```python
class EventForm(forms.ModelForm):
    class Meta:
        model = Event
        fields = ['title', 'start_date', 'end_date', 'max_attendees', 'registered_attendees']

    def clean(self):
        """Cross-field validation"""
        cleaned_data = super().clean()

        start_date = cleaned_data.get('start_date')
        end_date = cleaned_data.get('end_date')
        max_attendees = cleaned_data.get('max_attendees')
        registered = cleaned_data.get('registered_attendees')

        # Validate date range
        if start_date and end_date:
            if end_date < start_date:
                raise forms.ValidationError("End date must be after start date")

            from datetime import datetime
            if start_date < datetime.now().date():
                raise forms.ValidationError("Event cannot start in the past")

        # Validate attendee numbers
        if registered and max_attendees:
            if registered > max_attendees:
                self.add_error('registered_attendees', "Registered attendees cannot exceed max")

        return cleaned_data
```

### 3. Custom Validators

```python
from django.core.exceptions import ValidationError
from django.core.validators import RegexValidator
import re

def validate_phone_number(value):
    """Custom validator function"""
    pattern = r'^\+?1?\d{9,15}$'
    if not re.match(pattern, value):
        raise ValidationError('Invalid phone number format')

class ContactForm(forms.Form):
    phone = forms.CharField(
        validators=[
            validate_phone_number,
            RegexValidator(r'^\+?1?\d{9,15}$', 'Invalid phone number')
        ]
    )

    postal_code = forms.CharField(
        validators=[
            RegexValidator(r'^\d{5}(-\d{4})?$', 'Invalid US postal code')
        ]
    )
```

## 🏗️ Advanced Forms

### 1. Formsets (Multiple Forms)

```python
from django.forms import modelformset_factory, inlineformset_factory

# Model formset - multiple instances of same model
PostFormSet = modelformset_factory(
    Post,
    fields=['title', 'content'],
    extra=3,  # Number of empty forms
    can_delete=True
)

def manage_posts(request):
    if request.method == 'POST':
        formset = PostFormSet(request.POST)
        if formset.is_valid():
            formset.save()
            return redirect('post_list')
    else:
        formset = PostFormSet(queryset=Post.objects.filter(author=request.user))

    return render(request, 'manage_posts.html', {'formset': formset})

# Inline formset - related models (e.g., Order with OrderItems)
OrderItemFormSet = inlineformset_factory(
    Order,           # Parent model
    OrderItem,       # Child model
    fields=['product', 'quantity', 'price'],
    extra=1,
    can_delete=True
)

def edit_order(request, pk):
    order = get_object_or_404(Order, pk=pk)

    if request.method == 'POST':
        formset = OrderItemFormSet(request.POST, instance=order)
        if formset.is_valid():
            formset.save()
            return redirect('order_detail', pk=order.pk)
    else:
        formset = OrderItemFormSet(instance=order)

    return render(request, 'edit_order.html', {
        'order': order,
        'formset': formset
    })
```

### 2. Dynamic Forms

```python
class DynamicCategoryForm(forms.Form):
    """Form with dynamically generated fields"""

    def __init__(self, *args, **kwargs):
        categories = kwargs.pop('categories', [])
        super().__init__(*args, **kwargs)

        # Add field for each category
        for category in categories:
            field_name = f'category_{category.id}'
            self.fields[field_name] = forms.BooleanField(
                label=category.name,
                required=False
            )

# Usage
def select_categories(request):
    categories = Category.objects.all()

    if request.method == 'POST':
        form = DynamicCategoryForm(request.POST, categories=categories)
        if form.is_valid():
            selected = [
                cat.id for cat in categories
                if form.cleaned_data.get(f'category_{cat.id}')
            ]
            # Process selected categories
    else:
        form = DynamicCategoryForm(categories=categories)

    return render(request, 'select_categories.html', {'form': form})
```

## ✅ Best Practices

### 1. Proper Form Handling

```python
from django.contrib import messages
from django.shortcuts import render, redirect

def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()  # Save many-to-many relationships

            messages.success(request, 'Post created successfully!')
            return redirect('post_detail', pk=post.pk)
        else:
            messages.error(request, 'Please correct the errors below.')
    else:
        form = PostForm()

    return render(request, 'post_form.html', {'form': form})
```

### 2. Template Rendering

```html
<!-- Manual rendering -->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %} {{ form.title.label_tag }} {{ form.title }} {% if
  form.title.errors %}
  <div class="error">{{ form.title.errors }}</div>
  {% endif %} {{ form.content.label_tag }} {{ form.content }} {% if
  form.content.errors %}
  <div class="error">{{ form.content.errors }}</div>
  {% endif %} {% if form.non_field_errors %}
  <div class="error">{{ form.non_field_errors }}</div>
  {% endif %}

  <button type="submit">Submit</button>
</form>

<!-- Auto-rendering (quick but less control) -->
<form method="post">
  {% csrf_token %} {{ form.as_p }}
  <button type="submit">Submit</button>
</form>

<!-- With crispy forms (recommended) -->
{% load crispy_forms_tags %}
<form method="post">
  {% csrf_token %} {{ form|crispy }}
  <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

## 🔐 Security Best Practices

### 1. CSRF Protection

```html
<!-- Always include {% csrf_token %} -->
<form method="post">
  {% csrf_token %} {{ form.as_p }}
  <button type="submit">Submit</button>
</form>
```

### 2. File Upload Security

```python
from django.core.exceptions import ValidationError

class DocumentForm(forms.ModelForm):
    class Meta:
        model = Document
        fields = ['title', 'file']

    def clean_file(self):
        """Validate uploaded file"""
        file = self.cleaned_data.get('file')

        # Check file size (max 5MB)
        if file.size > 5 * 1024 * 1024:
            raise ValidationError("File size cannot exceed 5MB")

        # Check file extension
        allowed_extensions = ['.pdf', '.doc', '.docx']
        ext = os.path.splitext(file.name)[1].lower()
        if ext not in allowed_extensions:
            raise ValidationError(f"Only {', '.join(allowed_extensions)} files allowed")

        # Check MIME type
        import magic
        mime = magic.from_buffer(file.read(1024), mime=True)
        file.seek(0)  # Reset file pointer
        allowed_types = ['application/pdf', 'application/msword']
        if mime not in allowed_types:
            raise ValidationError("Invalid file type")

        return file
```

## ⚡ Performance Optimization

### 1. Optimize QuerySets in Forms

```python
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category', 'tags']

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # Optimize queryset for dropdown
        self.fields['category'].queryset = Category.objects.only('id', 'name')

        # Prefetch for M2M
        self.fields['tags'].queryset = Tag.objects.all().order_by('name')
```

## ❓ Interview Questions

### Q1: What is the difference between Form and ModelForm?

**Answer**:

- **Form**: Manual field definition, custom save logic
- **ModelForm**: Automatically generated from model, handles model saving

Use ModelForm for model-based forms.

### Q2: How do you validate across multiple fields?

**Answer**:
Override the `clean()` method:

```python
def clean(self):
    cleaned_data = super().clean()
    # Cross-field validation logic
    return cleaned_data
```

### Q3: What is cleaned_data?

**Answer**:
Dictionary containing validated and normalized form data after `is_valid()` is called. Always use `cleaned_data`, not `request.POST` directly.

## 📚 Summary

**Key Takeaways**:

1. Use ModelForm for model-backed forms
2. Validate with clean\_<field>() and clean()
3. Always include {% csrf_token %} in POST forms
4. Use cleaned_data after is_valid()
5. Validate file uploads (size, type, MIME)
6. Use formsets for multiple instances
7. Optimize querysets in form **init**()
8. Render errors properly in templates

Django forms provide secure, validated user input handling!
