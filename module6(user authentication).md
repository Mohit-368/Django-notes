# 🎓 Module 6 — User Authentication (Complete Guide)

## 🎯 The Interview Questions

> *"How does Django's authentication system work?"*
> *"What's the difference between authentication and authorization?"*
> *"How do you create a custom user model?"*
> *"How does Django store passwords?"*

---

## First — Two Concepts You Must Know

| Concept | Question It Answers | Examples |
|---------|---------------------|---------|
| **Authentication** | WHO are you? | Login, logout |
| **Authorization** | WHAT can you do? | Permissions, groups |

> 📒 **Interview note:** Interviewers ask this difference constantly!

---

## 1️⃣ Django's Built-in User Model

Django gives you a `User` model out of the box:

```python
from django.contrib.auth.models import User

# Default fields:
user.username       # 'john_doe'
user.email          # 'john@email.com'
user.password       # hashed password (never plain text!)
user.first_name     # 'John'
user.last_name      # 'Doe'
user.is_active      # True/False — can user login?
user.is_staff       # True/False — can access admin?
user.is_superuser   # True/False — has all permissions?
user.date_joined    # when account was created
user.last_login     # last login timestamp

# Useful methods:
user.is_authenticated           # True if logged in
user.check_password('mypass')   # verify password
user.set_password('newpass')    # set new password
user.get_full_name()            # 'John Doe'
```

---

## 2️⃣ How Django Stores Passwords — Never Plain Text

> 📒 **Interview note:** This is the most asked security question in interviews!

```python
# User sets password: "mypassword123"
# What's actually stored in the database:
pbkdf2_sha256$260000$abc123$hashedvalue==
#    👆            👆        👆       👆
#  algorithm    iterations   salt    hash
```

### Storing a Password

```
User types: "mypassword123"
                ↓
Django adds random SALT: "mypassword123" + "randomsalt"
                ↓
Runs through hashing algorithm (PBKDF2 + SHA256)
                ↓
Stores: "pbkdf2_sha256$260000$randomsalt$hashedresult"
```

### Verifying a Password at Login

```
User types: "mypassword123"
                ↓
Django takes stored salt
                ↓
Hashes: "mypassword123" + stored_salt
                ↓
Compares result with stored hash
                ↓
Match? ✅ Login success    |    No match? ❌ Wrong password
```

**Why salt?** Two users with the same password get different hashes. Attackers can't use pre-computed lookup tables.

> 📒 **Interview answer:** *"Django uses PBKDF2 with SHA256 hashing and a random salt. Passwords are never stored in plain text. Even Django can't reverse the hash — it only verifies by re-hashing."*

---

## 3️⃣ Setting Up Authentication URLs

Django gives you built-in auth views — just wire the URLs:

```python
# myproject/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),
    # 👆 one line gives you ALL of these for free:
    # accounts/login/
    # accounts/logout/
    # accounts/password_change/
    # accounts/password_reset/
    # accounts/password_reset/done/
    # accounts/reset/<uidb64>/<token>/
    path('accounts/', include('accounts.urls')),  # your custom views
]
```

---

## 4️⃣ Login — Complete Implementation

### Built-in Way (Simplest)

```python
# settings.py
LOGIN_URL = '/accounts/login/'            # redirect here if not logged in
LOGIN_REDIRECT_URL = '/blog/'             # redirect here after login
LOGOUT_REDIRECT_URL = '/accounts/login/'  # redirect here after logout
```

Just create the template — Django handles everything else:

```html
<!-- templates/registration/login.html -->
{% extends 'base.html' %}

{% block content %}
<h1>Login</h1>

{% if form.errors %}
    <p style="color:red">Invalid username or password.</p>
{% endif %}

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Login</button>
</form>

<p>Don't have an account? <a href="{% url 'register' %}">Register</a></p>
{% endblock %}
```

### Custom Login View (More Control)

```python
# views.py
from django.contrib.auth import authenticate, login, logout
from django.shortcuts import render, redirect

def login_view(request):
    if request.user.is_authenticated:
        return redirect('blog:post-list')  # already logged in

    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')

        user = authenticate(request, username=username, password=password)

        if user is not None:
            login(request, user)  # creates session
            next_url = request.GET.get('next', 'blog:post-list')
            return redirect(next_url)
        else:
            return render(request, 'accounts/login.html', {
                'error': 'Invalid username or password'
            })

    return render(request, 'accounts/login.html')
```

### The Three Key Auth Functions

| Function | What It Does |
|----------|-------------|
| `authenticate(request, username=..., password=...)` | Checks if credentials are valid. Returns `User` or `None` |
| `login(request, user)` | Creates a session, stores user info in a cookie |
| `logout(request)` | Destroys the session, user is logged out |

---

## 5️⃣ What Is a Session? — Explained Deeply

> 📒 **Interview note:** Interviewers love this question!

HTTP is **stateless** — every request is independent. The server doesn't remember you between requests:

```
Request 1: "Hi I'm John, show me my posts"  →  "Here are posts"
Request 2: "Show me my profile"              →  "Who are you??" 😕
```

Sessions solve this:

```
Login request → Django creates session:
  session_id   = "abc123xyz"
  session_data = {'user_id': 5, 'username': 'john'}
  stored in database (django_session table)

Browser receives cookie:
  sessionid=abc123xyz  ← stored in browser

Every future request sends this cookie automatically:
  Request: "Show me my profile" + cookie: sessionid=abc123xyz
  Django: checks cookie → finds session → knows it's John ✅
```

```python
login(request, user)      # creates session automatically
logout(request)           # destroys it

# Store custom data in session:
request.session['last_visited'] = '/posts/'
request.session['cart_items'] = 3

# Read session data:
last = request.session.get('last_visited', '/')
```

---

## 6️⃣ Registration — Complete Implementation

Django doesn't provide a registration view — you build it:

```python
# forms.py
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm

class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)
    first_name = forms.CharField(max_length=50)
    last_name = forms.CharField(max_length=50)

    class Meta:
        model = User
        fields = ['username', 'first_name', 'last_name',
                  'email', 'password1', 'password2']

    def clean_email(self):
        email = self.cleaned_data['email']
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError("This email is already registered.")
        return email
```

```python
# views.py
from django.contrib.auth import login
from .forms import RegisterForm

def register_view(request):
    if request.user.is_authenticated:
        return redirect('blog:post-list')

    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()    # creates the user
            login(request, user)  # log them in immediately
            return redirect('blog:post-list')
    else:
        form = RegisterForm()

    return render(request, 'accounts/register.html', {'form': form})
```

```html
<!-- templates/accounts/register.html -->
{% extends 'base.html' %}

{% block content %}
<h1>Register</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Create Account</button>
</form>

<p>Already have an account? <a href="{% url 'login' %}">Login</a></p>
{% endblock %}
```

---

## 7️⃣ Logout

```python
# views.py
from django.contrib.auth import logout

def logout_view(request):
    logout(request)
    return redirect('login')
```

```html
<!-- Simple link -->
<a href="{% url 'logout' %}">Logout</a>

<!-- As a form (more secure — uses POST) -->
<form method="post" action="{% url 'logout' %}">
    {% csrf_token %}
    <button type="submit">Logout</button>
</form>
```

---

## 8️⃣ Protecting Views — 3 Ways

### `@login_required` decorator (FBV)

```python
from django.contrib.auth.decorators import login_required

@login_required(login_url='/accounts/login/')
def post_create(request):
    ...
# Unauthenticated user → redirected to login
# After login → redirected back to original page ✅
```

### `LoginRequiredMixin` (CBV)

```python
from django.contrib.auth.mixins import LoginRequiredMixin

class PostCreateView(LoginRequiredMixin, CreateView):
    login_url = '/accounts/login/'
    ...
```

### Manual Check (Most Control)

```python
def post_create(request):
    if not request.user.is_authenticated:
        return redirect('login')
    ...
```

---

## 9️⃣ Custom User Model — The Right Way

> 📒 **Interview note:** This is the most important interview topic in this module!

Django's default User model might not have everything you need. In real projects you **always** create a custom user model. Adding fields to the default User later is very painful — do it from the start.

### Method 1 — `AbstractUser` (Recommended)

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    # Keep all default fields + add your own:
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    phone = models.CharField(max_length=15, blank=True)
    is_verified = models.BooleanField(default=False)

    def __str__(self):
        return self.username
```

```python
# settings.py
AUTH_USER_MODEL = 'accounts.CustomUser'
# Must be set BEFORE first migration!
```

```python
# Always import User this way in real projects:
from django.contrib.auth import get_user_model
User = get_user_model()  # gets CustomUser automatically

# Never do this:
from django.contrib.auth.models import User  # ❌ hardcoded
```

### Method 2 — `AbstractBaseUser` (Full Control)

Use only when you need completely custom auth (e.g. email login instead of username):

```python
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager

class CustomUserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('Email is required')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save()
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)

class CustomUser(AbstractBaseUser):
    email = models.EmailField(unique=True)
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    USERNAME_FIELD = 'email'  # login with email instead of username
    REQUIRED_FIELDS = ['first_name', 'last_name']

    objects = CustomUserManager()
```

### Which to Choose?

| Option | When to Use |
|--------|-------------|
| `AbstractUser` | Keep all default fields, just add more. Recommended for most projects. |
| `AbstractBaseUser` | Build auth from scratch — e.g. email login, completely custom flow. |

---

## 🔟 User Profile — Extending User Data

A common pattern in real projects — a separate profile model:

```python
# models.py
class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE,
                                related_name='profile')
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    website = models.URLField(blank=True)
    followers = models.ManyToManyField(User, related_name='following', blank=True)

    def __str__(self):
        return f'{self.user.username} Profile'
```

### Auto-Create Profile on Registration — Using Signals

```python
# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth import get_user_model
from .models import Profile

User = get_user_model()

@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_profile(sender, instance, **kwargs):
    instance.profile.save()
```

```python
# apps.py — register signals
class AccountsConfig(AppConfig):
    name = 'accounts'

    def ready(self):
        import accounts.signals  # 👈 import signals here
```

Now access profile anywhere:

```python
request.user.profile.bio
request.user.profile.avatar
```

---

## 1️⃣1️⃣ Password Reset — Built-in Flow

Django handles the entire password reset flow automatically:

```
User clicks "Forgot Password"
            ↓
Enters email address
            ↓
Django sends email with reset link
            ↓
User clicks link in email
            ↓
Django shows "enter new password" form
            ↓
Password updated → redirect to login
```

```python
# settings.py

# Production:
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'your@gmail.com'
EMAIL_HOST_PASSWORD = 'your-app-password'

# Development — prints emails to console instead of sending:
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

Templates Django needs (create these files):

```
templates/registration/
    password_reset_form.html        ← enter email
    password_reset_done.html        ← "check your email"
    password_reset_confirm.html     ← enter new password
    password_reset_complete.html    ← "password changed!"
```

---

## 1️⃣2️⃣ Permissions & Groups

```python
# Check permission in a view:
def delete_post(request, pk):
    if not request.user.has_perm('blog.delete_post'):
        return HttpResponseForbidden()
    ...

# Check permission in a template:
{% if perms.blog.delete_post %}
    <a href="{% url 'post-delete' post.pk %}">Delete</a>
{% endif %}

# Assign users to groups:
from django.contrib.auth.models import Group

editors = Group.objects.create(name='Editors')
user.groups.add(editors)
user.has_perm('blog.change_post')  # True if editors group has this perm
```

---

## 🎯 Common Interview Questions

**Q1. What's the difference between authentication and authorization?**
> Authentication = verifying WHO you are (login). Authorization = what you're ALLOWED to do (permissions).

**Q2. How does Django store passwords?**
> Uses PBKDF2 with SHA256 hashing + a random salt. Never stores plain text. Can't be reversed — only verified by re-hashing.

**Q3. What does `login()` do?**
> Creates a session for the user, stores user ID in that session, and sets a session cookie in the browser.

**Q4. Why should you always create a custom user model?**
> The default User model is hard to extend after the first migration. Creating a custom model from the start (using `AbstractUser`) lets you add fields without painful migrations later.

**Q5. What's the difference between `AbstractUser` and `AbstractBaseUser`?**
> `AbstractUser` keeps all default fields and lets you add more. `AbstractBaseUser` builds from scratch — use it when you need email login or completely custom authentication.

**Q6. What is `get_user_model()` and why use it?**
> Returns the currently active User model (custom or default). Always use it instead of importing `User` directly — makes code work regardless of which User model is configured.

---

## 📒 Notebook Summary

```
Module 6 — Authentication

Authentication → WHO are you (login/logout)
Authorization  → WHAT can you do (permissions)

Password storage:
  PBKDF2 + SHA256 + random salt → stored hash
  Never plain text, can't be reversed

3 key functions:
  authenticate() → checks username/password → User or None
  login()        → creates session
  logout()       → destroys session

Session → stores user data between requests
        → session_id in cookie, data in DB

Custom User Model:
  AbstractUser     → add fields to default (recommended)
  AbstractBaseUser → build from scratch (email login)
  AUTH_USER_MODEL = 'accounts.CustomUser'
  Always use get_user_model(), not User directly

Protecting views:
  @login_required    → FBV
  LoginRequiredMixin → CBV
  is_authenticated   → manual check

Interview:
  - auth vs authorization
  - how passwords are stored
  - why custom user model
  - AbstractUser vs AbstractBaseUser
```

---

## ✏️ Quick Check

**Q1.** What's wrong with this code?

```python
from django.contrib.auth.models import User

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
```

**Q2.** A user logs in — walk through exactly what happens step by step from `authenticate()` to the browser receiving the session cookie.

**Q3.** What's the difference between these two?

```python
user.is_authenticated
user.is_active
```

---

*Answer when ready — then move on to **Module 7 — Django Admin!** 🚀*
