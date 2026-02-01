# Forms & HTML5 Validation

## Core Concepts

HTML5 introduced native form validation that works without JavaScript, providing immediate user feedback through browser UI. Modern form validation combines HTML5 constraint validation attributes, the Constraint Validation API, CSS pseudo-classes, and custom JavaScript validation for comprehensive user experiences. Understanding these layers enables building accessible, user-friendly forms that guide users toward successful submission while maintaining data integrity.

### HTML5 Validation Attributes

HTML5 validation attributes enforce constraints directly in markup: `required`, `pattern`, `min`/`max`, `minlength`/`maxlength`, `step`, and input `type` validation. These attributes trigger browser-native validation UI without JavaScript, providing instant feedback when users submit forms or blur fields.

### Constraint Validation API

JavaScript's Constraint Validation API exposes validation state through the `validity` object, methods like `checkValidity()` and `reportValidity()`, and allows custom validation messages via `setCustomValidity()`. This API enables programmatic validation control while leveraging native browser behavior.

### CSS Validation States

CSS pseudo-classes (`:valid`, `:invalid`, `:in-range`, `:out-of-range`, `:required`, `:optional`) style form elements based on their validation state, providing real-time visual feedback without JavaScript event listeners.

---

## HTML/CSS/TypeScript Code Examples

### Example 1: Basic HTML5 Validation Attributes

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HTML5 Form Validation</title>
    <style>
      body {
        font-family:
          system-ui,
          -apple-system,
          sans-serif;
        max-width: 600px;
        margin: 2rem auto;
        padding: 0 1rem;
      }

      .form-group {
        margin-bottom: 1.5rem;
      }

      label {
        display: block;
        margin-bottom: 0.5rem;
        font-weight: 500;
      }

      input,
      textarea {
        width: 100%;
        padding: 0.75rem;
        border: 2px solid #e5e7eb;
        border-radius: 0.375rem;
        font-size: 1rem;
      }

      input:focus,
      textarea:focus {
        outline: none;
        border-color: #3b82f6;
      }

      button {
        background: #3b82f6;
        color: white;
        border: none;
        padding: 0.75rem 2rem;
        border-radius: 0.375rem;
        font-size: 1rem;
        cursor: pointer;
      }

      button:hover {
        background: #2563eb;
      }
    </style>
  </head>
  <body>
    <h1>Registration Form</h1>

    <form>
      <!-- Required Field -->
      <div class="form-group">
        <label for="username">Username (required)</label>
        <input
          type="text"
          id="username"
          name="username"
          required
          aria-required="true"
        />
      </div>

      <!-- Email Validation -->
      <div class="form-group">
        <label for="email">Email (required)</label>
        <input
          type="email"
          id="email"
          name="email"
          required
          aria-required="true"
        />
      </div>

      <!-- Min/Max Length -->
      <div class="form-group">
        <label for="password">Password (8-20 characters)</label>
        <input
          type="password"
          id="password"
          name="password"
          minlength="8"
          maxlength="20"
          required
          aria-required="true"
        />
      </div>

      <!-- Number with Min/Max -->
      <div class="form-group">
        <label for="age">Age (18-120)</label>
        <input
          type="number"
          id="age"
          name="age"
          min="18"
          max="120"
          required
          aria-required="true"
        />
      </div>

      <!-- URL Validation -->
      <div class="form-group">
        <label for="website">Website</label>
        <input
          type="url"
          id="website"
          name="website"
          placeholder="https://example.com"
        />
      </div>

      <!-- Tel Validation -->
      <div class="form-group">
        <label for="phone">Phone</label>
        <input
          type="tel"
          id="phone"
          name="phone"
          placeholder="+1 (555) 123-4567"
        />
      </div>

      <button type="submit">Register</button>
    </form>
  </body>
</html>
```

### Example 2: Pattern Attribute with Regex

```html
<form>
  <!-- Username: alphanumeric, 3-15 characters -->
  <div class="form-group">
    <label for="username">Username</label>
    <input
      type="text"
      id="username"
      name="username"
      pattern="[a-zA-Z0-9]{3,15}"
      title="Username must be 3-15 alphanumeric characters"
      required
    />
    <small>3-15 alphanumeric characters only</small>
  </div>

  <!-- US Phone Number: (555) 123-4567 -->
  <div class="form-group">
    <label for="phone">US Phone Number</label>
    <input
      type="tel"
      id="phone"
      name="phone"
      pattern="\([0-9]{3}\) [0-9]{3}-[0-9]{4}"
      title="Phone format: (555) 123-4567"
      placeholder="(555) 123-4567"
      required
    />
  </div>

  <!-- ZIP Code: 5 digits or 5+4 format -->
  <div class="form-group">
    <label for="zipcode">ZIP Code</label>
    <input
      type="text"
      id="zipcode"
      name="zipcode"
      pattern="[0-9]{5}(-[0-9]{4})?"
      title="ZIP code format: 12345 or 12345-6789"
      placeholder="12345 or 12345-6789"
      required
    />
  </div>

  <!-- Credit Card: 16 digits, spaces allowed -->
  <div class="form-group">
    <label for="card">Credit Card</label>
    <input
      type="text"
      id="card"
      name="card"
      pattern="[0-9]{4} [0-9]{4} [0-9]{4} [0-9]{4}"
      title="Card format: 1234 5678 9012 3456"
      placeholder="1234 5678 9012 3456"
      required
    />
  </div>

  <!-- Hex Color: #RGB or #RRGGBB -->
  <div class="form-group">
    <label for="color">Hex Color</label>
    <input
      type="text"
      id="color"
      name="color"
      pattern="#[0-9A-Fa-f]{6}"
      title="Hex color format: #RRGGBB"
      placeholder="#3B82F6"
      required
    />
  </div>

  <!-- Strong Password: min 8 chars, upper, lower, digit, special -->
  <div class="form-group">
    <label for="strongPassword">Strong Password</label>
    <input
      type="password"
      id="strongPassword"
      name="strongPassword"
      pattern="(?=.*\d)(?=.*[a-z])(?=.*[A-Z])(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}"
      title="Must contain: uppercase, lowercase, digit, special character, min 8 chars"
      required
    />
    <small
      >Must include uppercase, lowercase, number, and special character</small
    >
  </div>

  <button type="submit">Submit</button>
</form>
```

### Example 3: Custom Validation Messages

```html
<form id="customValidationForm">
  <div class="form-group">
    <label for="email">Email</label>
    <input type="email" id="email" name="email" required />
  </div>

  <div class="form-group">
    <label for="password">Password</label>
    <input
      type="password"
      id="password"
      name="password"
      minlength="8"
      required
    />
  </div>

  <div class="form-group">
    <label for="confirmPassword">Confirm Password</label>
    <input
      type="password"
      id="confirmPassword"
      name="confirmPassword"
      required
    />
  </div>

  <button type="submit">Register</button>
</form>

<script>
  const form = document.getElementById("customValidationForm");
  const email = document.getElementById("email");
  const password = document.getElementById("password");
  const confirmPassword = document.getElementById("confirmPassword");

  // Custom validation messages
  email.addEventListener("invalid", (e) => {
    if (email.validity.valueMissing) {
      email.setCustomValidity("Please enter your email address");
    } else if (email.validity.typeMismatch) {
      email.setCustomValidity(
        "Please enter a valid email address (e.g., user@example.com)",
      );
    }
  });

  email.addEventListener("input", () => {
    email.setCustomValidity(""); // Clear custom message on input
  });

  password.addEventListener("invalid", (e) => {
    if (password.validity.valueMissing) {
      password.setCustomValidity("Please create a password");
    } else if (password.validity.tooShort) {
      password.setCustomValidity(
        `Password must be at least 8 characters (current: ${password.value.length})`,
      );
    }
  });

  password.addEventListener("input", () => {
    password.setCustomValidity("");
  });

  // Password confirmation validation
  confirmPassword.addEventListener("input", () => {
    if (confirmPassword.value !== password.value) {
      confirmPassword.setCustomValidity("Passwords do not match");
    } else {
      confirmPassword.setCustomValidity("");
    }
  });

  form.addEventListener("submit", (e) => {
    if (confirmPassword.value !== password.value) {
      e.preventDefault();
      confirmPassword.setCustomValidity("Passwords do not match");
      confirmPassword.reportValidity();
    }
  });
</script>
```

### Example 4: Constraint Validation API

```typescript
interface ValidationResult {
  isValid: boolean;
  errors: string[];
}

class FormValidator {
  private form: HTMLFormElement;

  constructor(formSelector: string) {
    this.form = document.querySelector(formSelector)!;
  }

  // Check validity of single element
  validateElement(element: HTMLInputElement): ValidationResult {
    const result: ValidationResult = { isValid: true, errors: [] };
    const validity = element.validity;

    if (validity.valueMissing) {
      result.errors.push(
        `${element.labels?.[0]?.textContent || "This field"} is required`,
      );
      result.isValid = false;
    }

    if (validity.typeMismatch) {
      result.errors.push(`Please enter a valid ${element.type}`);
      result.isValid = false;
    }

    if (validity.patternMismatch) {
      result.errors.push(element.title || "Please match the requested format");
      result.isValid = false;
    }

    if (validity.tooShort) {
      result.errors.push(
        `Minimum length is ${element.minLength} characters (current: ${element.value.length})`,
      );
      result.isValid = false;
    }

    if (validity.tooLong) {
      result.errors.push(
        `Maximum length is ${element.maxLength} characters (current: ${element.value.length})`,
      );
      result.isValid = false;
    }

    if (validity.rangeUnderflow) {
      result.errors.push(`Minimum value is ${element.min}`);
      result.isValid = false;
    }

    if (validity.rangeOverflow) {
      result.errors.push(`Maximum value is ${element.max}`);
      result.isValid = false;
    }

    if (validity.stepMismatch) {
      result.errors.push(`Please enter a valid value (step: ${element.step})`);
      result.isValid = false;
    }

    if (validity.badInput) {
      result.errors.push("Please enter a valid value");
      result.isValid = false;
    }

    return result;
  }

  // Validate entire form
  validateForm(): ValidationResult {
    const result: ValidationResult = { isValid: true, errors: [] };
    const elements = this.form.querySelectorAll<HTMLInputElement>(
      "input, textarea, select",
    );

    elements.forEach((element) => {
      if (!element.checkValidity()) {
        const elementResult = this.validateElement(element);
        result.errors.push(...elementResult.errors);
        result.isValid = false;
      }
    });

    return result;
  }

  // Get validation state
  getValidityState(element: HTMLInputElement): ValidityState {
    return element.validity;
  }

  // Check if form can be submitted
  canSubmit(): boolean {
    return this.form.checkValidity();
  }

  // Show validation errors using browser UI
  showValidationErrors(): void {
    this.form.reportValidity();
  }

  // Get validation message
  getValidationMessage(element: HTMLInputElement): string {
    return element.validationMessage;
  }

  // Set custom validation message
  setCustomError(element: HTMLInputElement, message: string): void {
    element.setCustomValidity(message);
  }

  // Clear custom validation message
  clearCustomError(element: HTMLInputElement): void {
    element.setCustomValidity("");
  }
}

// Usage
const validator = new FormValidator("#registrationForm");

document.getElementById("registrationForm")?.addEventListener("submit", (e) => {
  e.preventDefault();

  if (validator.canSubmit()) {
    console.log("Form is valid, submitting...");
    // Submit form
  } else {
    validator.showValidationErrors();
    const result = validator.validateForm();
    console.error("Validation errors:", result.errors);
  }
});
```

### Example 5: CSS Validation States

```html
<style>
  /* Valid state styling */
  input:valid {
    border-color: #10b981;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='%2310b981' viewBox='0 0 16 16'%3E%3Cpath d='M13.854 3.646a.5.5 0 0 1 0 .708l-7 7a.5.5 0 0 1-.708 0l-3.5-3.5a.5.5 0 1 1 .708-.708L6.5 10.293l6.646-6.647a.5.5 0 0 1 .708 0z'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 0.75rem center;
    background-size: 1rem;
    padding-right: 2.5rem;
  }

  /* Invalid state styling */
  input:invalid {
    border-color: #ef4444;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='%23ef4444' viewBox='0 0 16 16'%3E%3Cpath d='M8 15A7 7 0 1 1 8 1a7 7 0 0 1 0 14zm0 1A8 8 0 1 0 8 0a8 8 0 0 0 0 16z'/%3E%3Cpath d='M7.002 11a1 1 0 1 1 2 0 1 1 0 0 1-2 0zM7.1 4.995a.905.905 0 1 1 1.8 0l-.35 3.507a.552.552 0 0 1-1.1 0L7.1 4.995z'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 0.75rem center;
    background-size: 1rem;
    padding-right: 2.5rem;
  }

  /* Don't show invalid state until user has interacted */
  input:invalid:not(:focus):not(:placeholder-shown) {
    border-color: #ef4444;
  }

  /* Required field indicator */
  input:required + label::after {
    content: " *";
    color: #ef4444;
  }

  /* Optional field styling */
  input:optional {
    border-style: dashed;
  }

  /* In-range and out-of-range */
  input[type="number"]:in-range {
    border-color: #10b981;
  }

  input[type="number"]:out-of-range {
    border-color: #ef4444;
  }

  /* Focus styling */
  input:focus:valid {
    outline: 2px solid #10b981;
    outline-offset: 2px;
  }

  input:focus:invalid {
    outline: 2px solid #ef4444;
    outline-offset: 2px;
  }

  /* Disabled state */
  input:disabled {
    background-color: #f3f4f6;
    cursor: not-allowed;
    opacity: 0.6;
  }

  /* Read-only state */
  input:read-only {
    background-color: #f9fafb;
    border-style: dotted;
  }

  /* Placeholder styling */
  input::placeholder {
    color: #9ca3af;
    font-style: italic;
  }

  input:focus::placeholder {
    color: #d1d5db;
  }

  /* Error message styling */
  .error-message {
    display: none;
    color: #ef4444;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }

  input:invalid:not(:focus):not(:placeholder-shown) + .error-message {
    display: block;
  }
</style>

<form>
  <div class="form-group">
    <label for="email">Email</label>
    <input
      type="email"
      id="email"
      name="email"
      placeholder="Enter your email"
      required
    />
    <span class="error-message">Please enter a valid email address</span>
  </div>

  <div class="form-group">
    <label for="age">Age</label>
    <input
      type="number"
      id="age"
      name="age"
      min="18"
      max="120"
      placeholder="Enter your age"
      required
    />
    <span class="error-message">Age must be between 18 and 120</span>
  </div>
</form>
```

### Example 6: Real-Time Validation with Debouncing

```typescript
class RealTimeValidator {
  private element: HTMLInputElement;
  private errorElement: HTMLElement | null;
  private validationTimeout: number | null = null;
  private debounceDelay: number = 300;

  constructor(elementSelector: string, errorSelector?: string) {
    this.element = document.querySelector(elementSelector)!;
    this.errorElement = errorSelector
      ? document.querySelector(errorSelector)
      : this.createErrorElement();

    this.attachListeners();
  }

  private createErrorElement(): HTMLElement {
    const errorDiv = document.createElement("div");
    errorDiv.className = "validation-error";
    errorDiv.style.cssText =
      "color: #ef4444; font-size: 0.875rem; margin-top: 0.25rem; display: none;";
    this.element.parentElement?.appendChild(errorDiv);
    return errorDiv;
  }

  private attachListeners(): void {
    // Validate on input with debounce
    this.element.addEventListener("input", () => {
      if (this.validationTimeout) {
        clearTimeout(this.validationTimeout);
      }

      this.validationTimeout = window.setTimeout(() => {
        this.validate();
      }, this.debounceDelay);
    });

    // Immediate validation on blur
    this.element.addEventListener("blur", () => {
      if (this.validationTimeout) {
        clearTimeout(this.validationTimeout);
      }
      this.validate();
    });

    // Clear error on focus
    this.element.addEventListener("focus", () => {
      this.clearError();
    });
  }

  private validate(): void {
    const isValid = this.element.checkValidity();

    if (!isValid) {
      this.showError(this.element.validationMessage);
    } else {
      this.clearError();
    }
  }

  private showError(message: string): void {
    if (this.errorElement) {
      this.errorElement.textContent = message;
      this.errorElement.style.display = "block";
    }
    this.element.setAttribute("aria-invalid", "true");
  }

  private clearError(): void {
    if (this.errorElement) {
      this.errorElement.style.display = "none";
    }
    this.element.setAttribute("aria-invalid", "false");
  }

  public setCustomValidator(validator: (value: string) => string | null): void {
    this.element.addEventListener("input", () => {
      const error = validator(this.element.value);
      if (error) {
        this.element.setCustomValidity(error);
      } else {
        this.element.setCustomValidity("");
      }
    });
  }
}

// Usage
const emailValidator = new RealTimeValidator("#email", "#email-error");

// Add custom validation
emailValidator.setCustomValidator((value) => {
  if (value && !value.includes("@")) {
    return "Email must contain @ symbol";
  }
  if (value && value.endsWith("@example.com")) {
    return "Please use your real email address";
  }
  return null;
});

const passwordValidator = new RealTimeValidator("#password", "#password-error");
passwordValidator.setCustomValidator((value) => {
  if (value.length > 0 && value.length < 8) {
    return `Password too short (${value.length}/8 characters)`;
  }
  if (!/[A-Z]/.test(value)) {
    return "Password must contain at least one uppercase letter";
  }
  if (!/[0-9]/.test(value)) {
    return "Password must contain at least one number";
  }
  return null;
});
```

### Example 7: Password Strength Meter

```html
<style>
  .password-strength {
    margin-top: 0.5rem;
  }

  .strength-meter {
    height: 4px;
    background: #e5e7eb;
    border-radius: 2px;
    overflow: hidden;
    margin-bottom: 0.5rem;
  }

  .strength-meter-fill {
    height: 100%;
    transition:
      width 0.3s,
      background-color 0.3s;
    width: 0%;
  }

  .strength-weak {
    background: #ef4444;
  }
  .strength-fair {
    background: #f59e0b;
  }
  .strength-good {
    background: #3b82f6;
  }
  .strength-strong {
    background: #10b981;
  }

  .strength-text {
    font-size: 0.875rem;
    font-weight: 500;
  }

  .password-requirements {
    margin-top: 0.5rem;
    font-size: 0.875rem;
  }

  .requirement {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    color: #6b7280;
    margin-bottom: 0.25rem;
  }

  .requirement.met {
    color: #10b981;
  }

  .requirement::before {
    content: "○";
    font-size: 1rem;
  }

  .requirement.met::before {
    content: "✓";
  }
</style>

<div class="form-group">
  <label for="password">Password</label>
  <input type="password" id="password" name="password" required />

  <div class="password-strength">
    <div class="strength-meter">
      <div class="strength-meter-fill" id="strengthMeterFill"></div>
    </div>
    <div class="strength-text" id="strengthText"></div>

    <div class="password-requirements">
      <div class="requirement" id="req-length">At least 8 characters</div>
      <div class="requirement" id="req-uppercase">One uppercase letter</div>
      <div class="requirement" id="req-lowercase">One lowercase letter</div>
      <div class="requirement" id="req-number">One number</div>
      <div class="requirement" id="req-special">One special character</div>
    </div>
  </div>
</div>

<script>
  class PasswordStrengthMeter {
    constructor(inputSelector) {
      this.input = document.querySelector(inputSelector);
      this.meterFill = document.getElementById("strengthMeterFill");
      this.strengthText = document.getElementById("strengthText");

      this.requirements = {
        length: {
          element: document.getElementById("req-length"),
          test: (pw) => pw.length >= 8,
        },
        uppercase: {
          element: document.getElementById("req-uppercase"),
          test: (pw) => /[A-Z]/.test(pw),
        },
        lowercase: {
          element: document.getElementById("req-lowercase"),
          test: (pw) => /[a-z]/.test(pw),
        },
        number: {
          element: document.getElementById("req-number"),
          test: (pw) => /[0-9]/.test(pw),
        },
        special: {
          element: document.getElementById("req-special"),
          test: (pw) => /[!@#$%^&*(),.?":{}|<>]/.test(pw),
        },
      };

      this.input.addEventListener("input", () => this.updateStrength());
    }

    updateStrength() {
      const password = this.input.value;
      let metCount = 0;

      // Check each requirement
      Object.values(this.requirements).forEach((req) => {
        const met = req.test(password);
        req.element.classList.toggle("met", met);
        if (met) metCount++;
      });

      // Calculate strength
      const strength = password.length === 0 ? 0 : metCount;
      const percentage = (strength / 5) * 100;

      // Update meter
      this.meterFill.style.width = `${percentage}%`;
      this.meterFill.className = "strength-meter-fill";

      // Update styling and text
      if (strength === 0) {
        this.strengthText.textContent = "";
      } else if (strength <= 2) {
        this.meterFill.classList.add("strength-weak");
        this.strengthText.textContent = "Weak";
        this.strengthText.style.color = "#ef4444";
      } else if (strength === 3) {
        this.meterFill.classList.add("strength-fair");
        this.strengthText.textContent = "Fair";
        this.strengthText.style.color = "#f59e0b";
      } else if (strength === 4) {
        this.meterFill.classList.add("strength-good");
        this.strengthText.textContent = "Good";
        this.strengthText.style.color = "#3b82f6";
      } else {
        this.meterFill.classList.add("strength-strong");
        this.strengthText.textContent = "Strong";
        this.strengthText.style.color = "#10b981";
      }

      // Set custom validity
      if (metCount < 5) {
        const missing = Object.entries(this.requirements)
          .filter(([_, req]) => !req.test(password))
          .map(([key]) => key);
        this.input.setCustomValidity(`Password must meet all requirements`);
      } else {
        this.input.setCustomValidity("");
      }
    }
  }

  // Initialize
  new PasswordStrengthMeter("#password");
</script>
```

### Example 8: Accessible Error Messages with ARIA

```html
<form class="accessible-form">
  <div class="form-group">
    <label for="username">
      Username
      <span aria-label="required">*</span>
    </label>
    <input
      type="text"
      id="username"
      name="username"
      aria-required="true"
      aria-invalid="false"
      aria-describedby="username-error username-hint"
      required
      pattern="[a-zA-Z0-9_]{3,15}"
    />
    <small id="username-hint"
      >3-15 characters, letters, numbers, and underscores only</small
    >
    <div
      id="username-error"
      class="error-message"
      role="alert"
      aria-live="polite"
      aria-atomic="true"
    ></div>
  </div>

  <div class="form-group">
    <label for="email">
      Email
      <span aria-label="required">*</span>
    </label>
    <input
      type="email"
      id="email"
      name="email"
      aria-required="true"
      aria-invalid="false"
      aria-describedby="email-error email-hint"
      required
    />
    <small id="email-hint">We'll never share your email</small>
    <div
      id="email-error"
      class="error-message"
      role="alert"
      aria-live="polite"
      aria-atomic="true"
    ></div>
  </div>

  <button type="submit">Submit</button>
</form>

<script>
  class AccessibleFormValidator {
    constructor(formSelector) {
      this.form = document.querySelector(formSelector);
      this.inputs = this.form.querySelectorAll("input[aria-invalid]");
      this.attachListeners();
    }

    attachListeners() {
      this.inputs.forEach((input) => {
        input.addEventListener("blur", () => this.validateField(input));
        input.addEventListener("input", () => this.clearError(input));
      });

      this.form.addEventListener("submit", (e) => {
        e.preventDefault();
        this.validateForm();
      });
    }

    validateField(input) {
      const errorElement = document.getElementById(`${input.id}-error`);

      if (!input.checkValidity()) {
        const message = this.getErrorMessage(input);
        this.showError(input, errorElement, message);
        return false;
      } else {
        this.clearError(input);
        return true;
      }
    }

    getErrorMessage(input) {
      if (input.validity.valueMissing) {
        return `${input.labels[0].textContent} is required`;
      }
      if (input.validity.typeMismatch) {
        return `Please enter a valid ${input.type}`;
      }
      if (input.validity.patternMismatch) {
        return input.title || `Please match the requested format`;
      }
      if (input.validity.tooShort) {
        return `Minimum length is ${input.minLength} characters`;
      }
      if (input.validity.tooLong) {
        return `Maximum length is ${input.maxLength} characters`;
      }
      return input.validationMessage;
    }

    showError(input, errorElement, message) {
      input.setAttribute("aria-invalid", "true");
      errorElement.textContent = message;
      errorElement.style.display = "block";

      // Announce error to screen readers
      errorElement.setAttribute("role", "alert");
    }

    clearError(input) {
      const errorElement = document.getElementById(`${input.id}-error`);
      input.setAttribute("aria-invalid", "false");
      errorElement.textContent = "";
      errorElement.style.display = "none";
      input.setCustomValidity("");
    }

    validateForm() {
      let isValid = true;
      const invalidInputs = [];

      this.inputs.forEach((input) => {
        if (!this.validateField(input)) {
          isValid = false;
          invalidInputs.push(input);
        }
      });

      if (!isValid && invalidInputs.length > 0) {
        // Focus first invalid input
        invalidInputs[0].focus();

        // Announce summary to screen readers
        const summary = `Form has ${invalidInputs.length} error${invalidInputs.length > 1 ? "s" : ""}. Please correct and resubmit.`;
        this.announceToScreenReader(summary);
      } else {
        console.log("Form is valid, submitting...");
        // Submit form
      }
    }

    announceToScreenReader(message) {
      const announcement = document.createElement("div");
      announcement.setAttribute("role", "status");
      announcement.setAttribute("aria-live", "polite");
      announcement.className = "sr-only";
      announcement.textContent = message;
      document.body.appendChild(announcement);

      setTimeout(() => announcement.remove(), 1000);
    }
  }

  // Initialize
  new AccessibleFormValidator(".accessible-form");
</script>

<style>
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }

  .error-message {
    display: none;
    color: #ef4444;
    font-size: 0.875rem;
    margin-top: 0.25rem;
    padding: 0.5rem;
    background: #fef2f2;
    border-left: 3px solid #ef4444;
    border-radius: 0.25rem;
  }

  [aria-invalid="true"] {
    border-color: #ef4444;
  }

  [aria-invalid="false"]:focus {
    border-color: #3b82f6;
  }
</style>
```

### Example 9: Multi-Step Form Validation

```typescript
interface StepValidation {
  stepIndex: number;
  isValid: boolean;
  errors: string[];
}

class MultiStepFormValidator {
  private form: HTMLFormElement;
  private steps: NodeListOf<HTMLElement>;
  private currentStep: number = 0;
  private nextBtn: HTMLButtonElement;
  private prevBtn: HTMLButtonElement;
  private submitBtn: HTMLButtonElement;
  private stepIndicators: NodeListOf<HTMLElement>;

  constructor(formSelector: string) {
    this.form = document.querySelector(formSelector)!;
    this.steps = this.form.querySelectorAll(".form-step");
    this.nextBtn = this.form.querySelector('[data-action="next"]')!;
    this.prevBtn = this.form.querySelector('[data-action="prev"]')!;
    this.submitBtn = this.form.querySelector('[data-action="submit"]')!;
    this.stepIndicators = this.form.querySelectorAll(".step-indicator");

    this.init();
  }

  private init(): void {
    this.showStep(0);
    this.attachListeners();
  }

  private attachListeners(): void {
    this.nextBtn.addEventListener("click", () => this.nextStep());
    this.prevBtn.addEventListener("click", () => this.prevStep());
    this.submitBtn.addEventListener("click", (e) => {
      e.preventDefault();
      this.submitForm();
    });
  }

  private showStep(stepIndex: number): void {
    this.steps.forEach((step, index) => {
      step.classList.toggle("active", index === stepIndex);
      step.setAttribute("aria-hidden", String(index !== stepIndex));
    });

    this.stepIndicators.forEach((indicator, index) => {
      indicator.classList.toggle("active", index === stepIndex);
      indicator.classList.toggle("completed", index < stepIndex);
    });

    this.prevBtn.style.display = stepIndex === 0 ? "none" : "block";
    this.nextBtn.style.display =
      stepIndex === this.steps.length - 1 ? "none" : "block";
    this.submitBtn.style.display =
      stepIndex === this.steps.length - 1 ? "block" : "none";

    this.currentStep = stepIndex;
  }

  private validateStep(stepIndex: number): StepValidation {
    const step = this.steps[stepIndex];
    const inputs = step.querySelectorAll<HTMLInputElement>(
      "input, select, textarea",
    );
    const validation: StepValidation = { stepIndex, isValid: true, errors: [] };

    inputs.forEach((input) => {
      if (!input.checkValidity()) {
        validation.isValid = false;
        validation.errors.push(input.validationMessage);

        // Show error
        const errorElement = step.querySelector(`#${input.id}-error`);
        if (errorElement) {
          errorElement.textContent = input.validationMessage;
          errorElement.classList.add("visible");
        }
        input.setAttribute("aria-invalid", "true");
      }
    });

    return validation;
  }

  private nextStep(): void {
    const validation = this.validateStep(this.currentStep);

    if (validation.isValid) {
      if (this.currentStep < this.steps.length - 1) {
        this.showStep(this.currentStep + 1);
      }
    } else {
      // Focus first invalid input
      const step = this.steps[this.currentStep];
      const firstInvalid = step.querySelector<HTMLInputElement>(
        '[aria-invalid="true"]',
      );
      firstInvalid?.focus();
    }
  }

  private prevStep(): void {
    if (this.currentStep > 0) {
      this.showStep(this.currentStep - 1);
    }
  }

  private submitForm(): void {
    // Validate all steps
    const allValidations = Array.from(this.steps).map((_, index) =>
      this.validateStep(index),
    );

    const allValid = allValidations.every((v) => v.isValid);

    if (allValid) {
      console.log("Form is valid, submitting...");
      // Submit form or gather data
      const formData = new FormData(this.form);
      console.log("Form data:", Object.fromEntries(formData));
    } else {
      // Find first invalid step
      const firstInvalidStep = allValidations.findIndex((v) => !v.isValid);
      this.showStep(firstInvalidStep);

      alert(`Please complete step ${firstInvalidStep + 1} correctly`);
    }
  }

  public reset(): void {
    this.form.reset();
    this.showStep(0);

    // Clear all error messages
    this.form.querySelectorAll(".error-message").forEach((el) => {
      el.textContent = "";
      el.classList.remove("visible");
    });

    this.form.querySelectorAll("[aria-invalid]").forEach((el) => {
      el.setAttribute("aria-invalid", "false");
    });
  }
}

// Usage
const multiStepForm = new MultiStepFormValidator("#registrationForm");
```

```html
<!-- HTML Structure for Multi-Step Form -->
<form id="registrationForm" class="multi-step-form">
  <div class="step-indicators">
    <div class="step-indicator active" data-step="1">
      <span class="step-number">1</span>
      <span class="step-label">Personal Info</span>
    </div>
    <div class="step-indicator" data-step="2">
      <span class="step-number">2</span>
      <span class="step-label">Account</span>
    </div>
    <div class="step-indicator" data-step="3">
      <span class="step-number">3</span>
      <span class="step-label">Preferences</span>
    </div>
  </div>

  <!-- Step 1: Personal Information -->
  <div class="form-step active">
    <h2>Personal Information</h2>

    <div class="form-group">
      <label for="firstName">First Name</label>
      <input type="text" id="firstName" name="firstName" required />
      <div id="firstName-error" class="error-message"></div>
    </div>

    <div class="form-group">
      <label for="lastName">Last Name</label>
      <input type="text" id="lastName" name="lastName" required />
      <div id="lastName-error" class="error-message"></div>
    </div>

    <div class="form-group">
      <label for="birthdate">Date of Birth</label>
      <input
        type="date"
        id="birthdate"
        name="birthdate"
        required
        max="2008-01-01"
      />
      <div id="birthdate-error" class="error-message"></div>
    </div>
  </div>

  <!-- Step 2: Account Details -->
  <div class="form-step">
    <h2>Account Details</h2>

    <div class="form-group">
      <label for="email">Email</label>
      <input type="email" id="email" name="email" required />
      <div id="email-error" class="error-message"></div>
    </div>

    <div class="form-group">
      <label for="password">Password</label>
      <input
        type="password"
        id="password"
        name="password"
        required
        minlength="8"
      />
      <div id="password-error" class="error-message"></div>
    </div>

    <div class="form-group">
      <label for="confirmPassword">Confirm Password</label>
      <input
        type="password"
        id="confirmPassword"
        name="confirmPassword"
        required
      />
      <div id="confirmPassword-error" class="error-message"></div>
    </div>
  </div>

  <!-- Step 3: Preferences -->
  <div class="form-step">
    <h2>Preferences</h2>

    <div class="form-group">
      <label for="country">Country</label>
      <select id="country" name="country" required>
        <option value="">Select country</option>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
      </select>
      <div id="country-error" class="error-message"></div>
    </div>

    <div class="form-group">
      <label>
        <input type="checkbox" id="terms" name="terms" required />
        I agree to the terms and conditions
      </label>
      <div id="terms-error" class="error-message"></div>
    </div>
  </div>

  <div class="form-actions">
    <button type="button" class="btn btn-secondary" data-action="prev">
      Previous
    </button>
    <button type="button" class="btn btn-primary" data-action="next">
      Next
    </button>
    <button
      type="button"
      class="btn btn-success"
      data-action="submit"
      style="display: none;"
    >
      Submit
    </button>
  </div>
</form>
```

### Example 10: Async Validation (Email/Username Availability)

```typescript
interface ValidationResponse {
  isAvailable: boolean;
  message: string;
}

class AsyncValidator {
  private input: HTMLInputElement;
  private errorElement: HTMLElement;
  private loadingElement: HTMLElement;
  private validationTimeout: number | null = null;
  private debounceDelay: number = 500;
  private apiEndpoint: string;

  constructor(
    inputSelector: string,
    apiEndpoint: string,
    errorSelector?: string,
  ) {
    this.input = document.querySelector(inputSelector)!;
    this.apiEndpoint = apiEndpoint;
    this.errorElement =
      document.querySelector(errorSelector!) || this.createErrorElement();
    this.loadingElement = this.createLoadingElement();

    this.attachListeners();
  }

  private createErrorElement(): HTMLElement {
    const div = document.createElement("div");
    div.className = "validation-error";
    div.style.cssText =
      "color: #ef4444; font-size: 0.875rem; margin-top: 0.25rem;";
    this.input.parentElement?.appendChild(div);
    return div;
  }

  private createLoadingElement(): HTMLElement {
    const div = document.createElement("div");
    div.className = "validation-loading";
    div.style.cssText =
      "color: #6b7280; font-size: 0.875rem; margin-top: 0.25rem; display: none;";
    div.innerHTML = "<span>Checking availability...</span>";
    this.input.parentElement?.appendChild(div);
    return div;
  }

  private attachListeners(): void {
    this.input.addEventListener("input", () => {
      if (this.validationTimeout) {
        clearTimeout(this.validationTimeout);
      }

      // Clear previous state
      this.clearValidation();

      // Skip validation if input is empty or fails basic validation
      if (!this.input.value || !this.input.checkValidity()) {
        return;
      }

      // Show loading state
      this.showLoading();

      // Debounced async validation
      this.validationTimeout = window.setTimeout(async () => {
        await this.validateAsync();
      }, this.debounceDelay);
    });
  }

  private async validateAsync(): Promise<void> {
    try {
      const response = await this.checkAvailability(this.input.value);

      this.hideLoading();

      if (!response.isAvailable) {
        this.showError(response.message);
        this.input.setCustomValidity(response.message);
      } else {
        this.showSuccess(response.message);
        this.input.setCustomValidity("");
      }
    } catch (error) {
      this.hideLoading();
      this.showError("Unable to verify availability. Please try again.");
      console.error("Async validation error:", error);
    }
  }

  private async checkAvailability(value: string): Promise<ValidationResponse> {
    // Simulate API call
    return new Promise((resolve) => {
      setTimeout(() => {
        // Simulate checking against database
        const unavailable = ["admin", "test", "user", "demo"];
        const isAvailable = !unavailable.includes(value.toLowerCase());

        resolve({
          isAvailable,
          message: isAvailable
            ? `${value} is available!`
            : `${value} is already taken`,
        });
      }, 1000);
    });

    // Real implementation would be:
    // const response = await fetch(`${this.apiEndpoint}?value=${encodeURIComponent(value)}`);
    // return await response.json();
  }

  private showLoading(): void {
    this.loadingElement.style.display = "block";
  }

  private hideLoading(): void {
    this.loadingElement.style.display = "none";
  }

  private showError(message: string): void {
    this.errorElement.textContent = message;
    this.errorElement.style.color = "#ef4444";
    this.errorElement.style.display = "block";
    this.input.setAttribute("aria-invalid", "true");
  }

  private showSuccess(message: string): void {
    this.errorElement.textContent = message;
    this.errorElement.style.color = "#10b981";
    this.errorElement.style.display = "block";
    this.input.setAttribute("aria-invalid", "false");
  }

  private clearValidation(): void {
    this.errorElement.style.display = "none";
    this.input.setCustomValidity("");
  }
}

// Usage
const usernameValidator = new AsyncValidator(
  "#username",
  "/api/check-username",
  "#username-error",
);

const emailValidator = new AsyncValidator(
  "#email",
  "/api/check-email",
  "#email-error",
);
```

---

## Real-World Usage

### E-commerce Checkout Form

E-commerce checkout requires robust validation for payment information, shipping addresses, and customer data. Implement real-time validation for credit card numbers (using Luhn algorithm), postal codes (format validation by country), and email verification.

### Registration Forms with Password Confirmation

User registration demands password strength validation, matching confirmation fields, and username availability checking. Use async validation for uniqueness checks while providing immediate feedback for password strength and format requirements.

### Survey Forms with Conditional Logic

Surveys often have conditional questions based on previous answers. Implement dynamic validation rules that activate/deactivate based on user selections, ensuring only relevant fields are validated.

### File Upload Forms

File upload validation requires checking file types (MIME type validation), file sizes, and image dimensions. Use FileReader API for client-side validation before upload, providing immediate feedback without server round-trips.

---

## Production Patterns

### Form Validation Manager Pattern

```typescript
interface ValidationRule {
  validate: (value: string) => boolean;
  message: string;
}

class FormValidationManager {
  private forms: Map<string, HTMLFormElement> = new Map();
  private customRules: Map<string, ValidationRule[]> = new Map();

  registerForm(formId: string): void {
    const form = document.getElementById(formId) as HTMLFormElement;
    if (!form) return;

    this.forms.set(formId, form);
    this.setupFormValidation(form);
  }

  addCustomRule(fieldId: string, rule: ValidationRule): void {
    if (!this.customRules.has(fieldId)) {
      this.customRules.set(fieldId, []);
    }
    this.customRules.get(fieldId)!.push(rule);
  }

  private setupFormValidation(form: HTMLFormElement): void {
    form.addEventListener("submit", (e) => {
      if (!form.checkValidity()) {
        e.preventDefault();
        this.showValidationErrors(form);
      }
    });
  }

  private showValidationErrors(form: HTMLFormElement): void {
    const invalidInputs = form.querySelectorAll<HTMLInputElement>(":invalid");
    invalidInputs.forEach((input) => {
      const errorElement = form.querySelector(`#${input.id}-error`);
      if (errorElement) {
        errorElement.textContent = input.validationMessage;
      }
    });
  }
}
```

### Progressive Enhancement Pattern

Start with HTML5 validation, enhance with JavaScript for better UX:

```html
<form novalidate>
  <!-- novalidate disables browser UI, we handle it with JS -->
  <!-- Form fields -->
</form>
```

```typescript
// If JavaScript fails, HTML5 validation still works by removing novalidate
if ("noValidate" in HTMLFormElement.prototype) {
  document.querySelectorAll("form").forEach((form) => {
    form.setAttribute("novalidate", "true");
    // Add custom validation
  });
}
```

---

## Best Practices

### 1. **Always Use Required Attribute**

Mark required fields with both `required` attribute and visual indicator (\*). Include `aria-required="true"` for screen readers.

### 2. **Provide Helpful Error Messages**

Default browser error messages are generic. Use `setCustomValidity()` or `title` attribute to provide context-specific, actionable error messages.

### 3. **Validate on Blur, Not on Every Keystroke**

Real-time validation on every keystroke is annoying. Validate when user leaves field (blur event) or after a debounce period.

### 4. **Show Success States**

Don't just show errors - show when fields are valid with green checkmarks or success messages. Positive feedback improves user confidence.

### 5. **Use Pattern Attribute Wisely**

The `pattern` attribute is powerful but complex. Always include `title` attribute to explain the expected format in plain language.

### 6. **Disable Submit Until Valid**

For critical forms (payment, registration), disable submit button until all required fields are valid. Prevent frustration from failed submissions.

### 7. **Group Related Validations**

For multi-field validations (password confirmation, date ranges), validate the group together and show errors clearly associated with the related fields.

### 8. **Implement Async Validation Carefully**

Debounce async validations (username/email availability) to reduce server load. Show loading states clearly. Don't block form submission while checking.

### 9. **Test Accessibility**

Always test forms with keyboard navigation and screen readers. Ensure error messages are announced, and invalid fields can be identified and reached.

### 10. **Consider Mobile UX**

Use appropriate input types (`email`, `tel`, `number`) to trigger correct mobile keyboards. Ensure error messages are visible above the keyboard.

### 11. **Validate Server-Side**

Client-side validation is for UX, not security. Always validate on the server. Never trust client-side validation alone.

### 12. **Provide Format Examples**

Use `placeholder` attributes or helper text to show expected formats (phone numbers, dates, etc.). Reduces validation errors.

---

## 10 Key Takeaways

1. **HTML5 Validation Works Without JavaScript**: Native browser validation provides immediate feedback with zero code, making it essential for progressive enhancement and accessibility.

2. **Constraint Validation API Bridges HTML and JavaScript**: The `validity` object, `checkValidity()`, and `setCustomValidity()` methods give programmatic control while maintaining native browser behavior.

3. **CSS Pseudo-Classes Provide Free Visual Feedback**: Using `:valid`, `:invalid`, `:in-range`, and `:out-of-range` styles form elements automatically without JavaScript event listeners.

4. **Custom Validation Messages Are Non-Negotiable**: Default browser error messages are generic and unhelpful - always customize with `setCustomValidity()` or `title` attributes.

5. **Real-Time Validation Needs Debouncing**: Validating on every keystroke is annoying - debounce input events (300-500ms) and validate on blur for better UX.

6. **Pattern Attribute Is Powerful but Complex**: Regular expressions in the `pattern` attribute enable sophisticated validation, but always include a `title` explaining the format in plain language.

7. **Accessible Forms Require ARIA Attributes**: Use `aria-invalid`, `aria-describedby`, `aria-live`, and `role="alert"` to ensure error messages are announced to screen readers.

8. **Async Validation Must Show Loading States**: When checking username/email availability, display "Checking..." indicators and debounce requests to prevent server overload.

9. **Multi-Step Forms Need Step-Level Validation**: Validate each step before allowing progression, store validation state, and highlight invalid steps in step indicators.

10. **Server-Side Validation Is Mandatory**: Client-side validation improves UX but never provides security - always validate data on the server to prevent malicious submissions.
