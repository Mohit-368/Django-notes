# 🎓 Module 1 — How Django Actually Works

> This is the **most important module** in the entire roadmap. Every other concept builds on this.

Most tutorial followers can *use* Django but can't explain *how* it works. After this module — you can. 💪

---

## 🎯 The Interview Question

> *"What happens when a user types a URL in the browser and hits enter in a Django app?"*

This single question tests your entire Django understanding. Let's build the perfect answer.

---

## 🔄 The Big Picture — Django's Request/Response Cycle

```
Browser types: http://myblog.com/posts/

        ↓  1. HTTP Request sent to server

    Web Server (Nginx/Gunicorn)

        ↓  2. Passes request to Django

    Django's WSGIHandler

        ↓  3. Request goes through Middleware stack

    URL Resolver (urls.py)

        ↓  4. Matches URL pattern → finds the view

    View (views.py)

        ↓  5. View talks to Model if needed

    Model (models.py) ←→ Database

        ↓  6. View passes data to Template

    Template (template.html)

        ↓  7. Template renders HTML

    HTTP Response

        ↓  8. Goes back through Middleware

    Browser receives HTML and displays it
```

Every single step matters. Let's understand each one deeply.

---

## Step 1 — Browser Sends an HTTP Request

When you type `http://myblog.com/posts/` the browser sends:

```http
GET /posts/ HTTP/1.1
Host: myblog.com
```

This is just a **text message** sent over the internet. It contains:

| Part | Description |
|------|-------------|
| **Method** | GET, POST, PUT, DELETE |
| **Path** | `/posts/` |
| **Headers** | Browser info, cookies, etc. |
| **Body** | Form data (only in POST requests) |

---

## Step 2 — WSGI / ASGI

Django receives the request through **WSGI** (Web Server Gateway Interface).

Think of it as a **translator** between the web server and Django:

```
Web Server speaks "HTTP"
Django speaks "Python"
WSGI translates between them
```

> 📒 **Interview note:** WSGI is synchronous (traditional Django). ASGI is asynchronous (modern Django with WebSockets). Interviewers ask this!

---

## Step 3 — Middleware Stack ⭐

This is where most beginners have zero knowledge — and interviewers know it.

Middleware is like a **series of security checkpoints** every request passes through:

```
Request →  SecurityMiddleware
        →  SessionMiddleware
        →  AuthenticationMiddleware  ← attaches user to request
        →  CsrfViewMiddleware        ← checks CSRF token
        →  View gets called
        ←  Response passes back through all middleware
        ←  Browser receives final response
```

**Real example — what `AuthenticationMiddleware` does automatically:**

```python
# This happens behind the scenes on every request:
request.user = get_user_from_session(request)
# That's why you can access request.user in every view!
```

> 📒 **Interview note:** Middleware runs on **every single request** — both incoming AND outgoing. Order matters: top-to-bottom on request, bottom-to-top on response.

---

## Step 4 — URL Resolver

Django opens your `urls.py` and tries to match the URL pattern:

```python
# urls.py
urlpatterns = [
    path('posts/', PostListView.as_view(), name='post-list'),       # ✅ matches /posts/
    path('posts/<int:pk>/', PostDetailView.as_view()),              # matches /posts/5/
    path('admin/', admin.site.urls),
]
```

Django checks patterns **one by one, top to bottom** until it finds a match.

**If no match is found** → Django returns **404 Not Found**

> 📒 **Interview note:** URL patterns are checked in order. Put more specific URLs before generic ones!

```python
# ✅ Correct order — specific first
path('posts/new/', PostCreateView.as_view()),
path('posts/<int:pk>/', PostDetailView.as_view())

# ❌ Wrong order — 'new' would be treated as a pk!
path('posts/<int:pk>/', PostDetailView.as_view()),
path('posts/new/', PostCreateView.as_view()),
```

---

## Step 5 — The View

The matched view gets called with the request:

```python
# Django calls this:
PostListView.as_view()(request)

# Your view runs:
class PostListView(ListView):
    model = Post
    template_name = 'posts/list.html'

    # Internally Django calls get()
    def get(self, request):
        posts = Post.objects.all()  # talks to database
        return render(request, 'posts/list.html', {'posts': posts})
```

The view is the **brain** — it decides what data to fetch and what to return.

---

## Step 6 & 7 — Model → Template

**View asks the Model for data:**

```python
posts = Post.objects.all()
# Django translates this to SQL:
# SELECT * FROM blog_post;
# Returns Python objects
```

**View passes data to Template:**

```python
return render(request, 'posts/list.html', {'posts': posts})
```

**Template turns it into HTML:**

```html
{% for post in posts %}
    <h2>{{ post.title }}</h2>
{% endfor %}

<!-- Renders as: -->
<h2>How to learn Django</h2>
<h2>Top 10 Python tips</h2>
```

---

## Step 8 — HTTP Response

Django wraps the HTML in an **HTTP Response:**

```http
HTTP/1.1 200 OK
Content-Type: text/html

<html>
  <h2>How to learn Django</h2>
  ...
</html>
```

### Status Codes You Must Know

> 📒 **Interview note:** Interviewers ask status codes constantly, especially in DRF!

| Code | Meaning |
|------|---------|
| `200` | OK — success |
| `201` | Created — something was created |
| `301` | Moved Permanently — permanent redirect |
| `302` | Found — temporary redirect |
| `400` | Bad Request — client sent bad data |
| `401` | Unauthorized — not logged in |
| `403` | Forbidden — logged in but no permission |
| `404` | Not Found — URL doesn't exist |
| `500` | Server Error — something crashed in Django |

---

## 🗂️ Django Project Structure

```
myproject/
│
├── manage.py           ← command line tool (runserver, migrate, etc.)
│
├── myproject/
│   ├── settings.py     ← all configuration (database, apps, middleware)
│   ├── urls.py         ← main URL file, includes app URLs
│   ├── wsgi.py         ← WSGI entry point for deployment
│   └── asgi.py         ← ASGI entry point for async
│
└── blog/               ← your app
    ├── models.py       ← database tables
    ├── views.py        ← business logic
    ├── urls.py         ← app-specific URLs
    ├── forms.py        ← form classes
    ├── admin.py        ← admin configuration
    ├── apps.py         ← app configuration
    └── templates/      ← HTML files
```

---

## 🎯 Common Interview Questions

**Q1. Explain Django's request/response cycle.**
> Walk through all 8 steps above confidently.

**Q2. What is middleware in Django?**
> Middleware is code that runs on every request/response. It sits between the web server and your view. Used for authentication, security, sessions, etc.

**Q3. What is WSGI?**
> Interface between web server and Django application. Translates HTTP requests into Python objects Django can work with.

**Q4. What happens when a URL isn't found in Django?**
> URL resolver goes through all patterns, finds no match, returns a 404 response.

**Q5. What's the difference between 401 and 403?**
> `401` = not logged in. `403` = logged in but don't have permission.

---

## 📒 Notebook Summary

```
Module 1 — Django Request/Response Cycle

8 Steps:
  1. Browser sends HTTP request
  2. WSGI receives it
  3. Middleware processes it
  4. URL resolver matches pattern
  5. View is called
  6. View fetches data from Model
  7. Template renders HTML
  8. HTTP Response sent back

Key points:
  - Middleware runs on EVERY request, both ways
  - URLs matched top to bottom
  - WSGI = sync, ASGI = async

Status codes: 200, 201, 301, 302, 400, 401, 403, 404, 500

Interview: "Walk through request/response cycle"
```

---

## ✏️ Quick Check

Test yourself before moving on:

1. A user visits `/posts/999/` but post 999 doesn't exist. Which step in the cycle handles this — and what response code is returned?

2. `request.user` is available in every view automatically. Which middleware makes this possible?

---

*Answer these, then say **"next"** for Module 2 — Models & ORM Deep Dive! 🚀*
