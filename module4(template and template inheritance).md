# 🎓 Module 4 — Templates & Template Inheritance

> Tutorials teach you basic templates. This module teaches you how **real projects** structure templates — the way that makes your code clean, maintainable, and interview-ready.

---

## 🎯 The Interview Questions

> *"What is template inheritance in Django?"*
> *"What's the difference between `include` and `extends`?"*
> *"What are template filters and tags?"*

---

## 1️⃣ The Problem — Without Template Inheritance

Imagine every page of your blog needs this:

```html
<!-- Every single page repeats this! -->
<!DOCTYPE html>
<html>
<head>
    <title>My Blog</title>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <nav>
        <a href="/blog/">Home</a>
        <a href="/about/">About</a>
    </nav>

    <!-- actual page content here -->

    <footer>© 2024 My Blog</footer>
</body>
</html>
```

If you copy this into every template, changing the navbar means editing 20 files. Fixing a footer typo means editing 20 files. **Template inheritance solves this.**

---

## 2️⃣ Template Inheritance — `extends` & `block`

### Step 1 — Create ONE base template

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}My Blog{% endblock %}</title>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <nav>
        <a href="{% url 'blog:post-list' %}">Home</a>
        <a href="{% url 'about' %}">About</a>
        {% if user.is_authenticated %}
            <a href="{% url 'logout' %}">Logout</a>
        {% else %}
            <a href="{% url 'login' %}">Login</a>
        {% endif %}
    </nav>

    <main>
        {% block content %}
        <!-- child templates fill this -->
        {% endblock %}
    </main>

    {% block extra_js %}
    <!-- child templates can add JS here -->
    {% endblock %}

    <footer>© 2024 My Blog</footer>
</body>
</html>
```

### Step 2 — Child templates extend it

```html
<!-- templates/blog/list.html -->
{% extends 'base.html' %}

{% block title %}All Posts{% endblock %}

{% block content %}
    <h1>All Posts</h1>
    {% for post in posts %}
        <div>
            <h2>{{ post.title }}</h2>
            <p>{{ post.content }}</p>
        </div>
    {% endfor %}
{% endblock %}
```

```html
<!-- templates/blog/detail.html -->
{% extends 'base.html' %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p>by {{ post.author }} on {{ post.created_at }}</p>
    <p>{{ post.content }}</p>
{% endblock %}
```

### How it works:

```
base.html has the skeleton 🦴
child fills in the blocks  🧩
Django combines them → full HTML page ✅
```

---

## 3️⃣ `block` — How It Works

```html
<!-- base.html defines blocks — like empty slots: -->
{% block content %}{% endblock %}

<!-- Set DEFAULT content in a block: -->
{% block title %}My Blog{% endblock %}
<!-- if child doesn't override this, 'My Blog' is used -->

<!-- Child can EXTEND parent block content with block.super: -->
{% block content %}
    {{ block.super }}  <!-- includes parent's content first -->
    <p>Extra content added by child</p>
{% endblock %}
```

---

## 4️⃣ `include` — Reusable Template Pieces

`include` is different from `extends`:

| Tag | Purpose |
|-----|---------|
| `extends` | Child inherits an **entire parent layout** |
| `include` | Inserts a **small reusable snippet** into a template |

**Real example — a post card used in multiple places:**

```html
<!-- templates/blog/components/post_card.html -->
<div class="post-card">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content|truncatewords:20 }}</p>
    <a href="{% url 'blog:post-detail' post.pk %}">Read more</a>
</div>
```

```html
<!-- templates/blog/list.html -->
{% extends 'base.html' %}

{% block content %}
    {% for post in posts %}
        {% include 'blog/components/post_card.html' %}
    {% endfor %}
{% endblock %}
```

You can also pass extra data to an included template:

```html
{% include 'blog/components/post_card.html' with show_author=True %}
```

---

## 5️⃣ Template Tags — Logic in Templates

```html
<!-- IF / ELIF / ELSE -->
{% if user.is_authenticated %}
    <p>Welcome, {{ user.username }}!</p>
{% elif user.is_staff %}
    <p>Welcome, admin!</p>
{% else %}
    <p>Please login</p>
{% endif %}

<!-- FOR loop with empty fallback -->
{% for post in posts %}
    <h2>{{ post.title }}</h2>
{% empty %}
    <p>No posts found.</p>
{% endfor %}

<!-- FOR loop variables -->
{% for post in posts %}
    {{ forloop.counter }}   <!-- 1, 2, 3...       -->
    {{ forloop.counter0 }}  <!-- 0, 1, 2...        -->
    {{ forloop.first }}     <!-- True on first iteration -->
    {{ forloop.last }}      <!-- True on last iteration  -->
{% endfor %}

<!-- URL tag -->
<a href="{% url 'blog:post-detail' post.pk %}">Read</a>

<!-- CSRF token — always required in forms! -->
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
</form>

<!-- Load static files -->
{% load static %}
<img src="{% static 'images/logo.png' %}">
```

---

## 6️⃣ Template Filters — Transform Variables

Filters modify how variables are displayed using the `|` pipe:

### Text Filters

```html
{{ post.title|upper }}              <!-- HELLO WORLD          -->
{{ post.title|lower }}              <!-- hello world          -->
{{ post.title|title }}              <!-- Hello World          -->
{{ post.title|truncatewords:10 }}   <!-- first 10 words...    -->
{{ post.title|truncatechars:50 }}   <!-- first 50 chars...    -->
{{ post.content|wordcount }}        <!-- number of words      -->
{{ post.content|linebreaks }}       <!-- converts \n to <br>  -->
{{ post.title|slugify }}            <!-- hello-world          -->
{{ post.title|length }}             <!-- 11                   -->
```

### Date Filters

```html
{{ post.created_at|date:'d M Y' }}      <!-- 09 Mar 2026        -->
{{ post.created_at|date:'D, d b Y' }}   <!-- Mon, 09 Mar 2026   -->
{{ post.created_at|timesince }}         <!-- 3 days ago         -->
{{ post.created_at|timeuntil }}         <!-- 2 days from now    -->
```

### Other Useful Filters

```html
{{ post.views|floatformat:2 }}      <!-- 1234.56                          -->
{{ post.bio|default:'No bio yet' }} <!-- fallback if value is empty        -->
{{ post.content|safe }}             <!-- renders HTML instead of escaping  -->
```

> 📒 **Interview note:** Never use `|safe` on user-submitted content — it bypasses HTML escaping and can lead to XSS attacks.

---

## 7️⃣ Template Structure in Real Projects

```
templates/
│
├── base.html                    ← main layout
├── base_auth.html               ← layout for login/register pages
│
├── blog/
│   ├── list.html                ← post list
│   ├── detail.html              ← post detail
│   ├── form.html                ← create/update form
│   ├── confirm_delete.html      ← delete confirmation
│   └── components/
│       ├── post_card.html       ← reusable post card
│       └── pagination.html      ← reusable pagination
│
├── accounts/
│   ├── login.html
│   ├── register.html
│   └── profile.html
│
└── errors/
    ├── 404.html                 ← custom 404 page
    └── 500.html                 ← custom 500 page
```

---

## 8️⃣ Custom 404 and 500 Pages

```python
# settings.py
DEBUG = False        # must be False to show custom error pages
ALLOWED_HOSTS = ['*']
```

```html
<!-- templates/404.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>404 — Page Not Found</h1>
    <p>The page you're looking for doesn't exist.</p>
    <a href="{% url 'blog:post-list' %}">Go Home</a>
{% endblock %}
```

Django automatically uses `404.html` and `500.html` from your templates folder when `DEBUG=False`.

---

## 🎯 Common Interview Questions

**Q1. What's the difference between `extends` and `include`?**
> `extends` inherits an entire layout — the child fills in blocks. `include` inserts a small reusable snippet inside a template.

**Q2. What is `block.super`?**
> Includes the parent block's content inside the child block. Lets you add to parent content instead of replacing it entirely.

**Q3. Why do we use `{% csrf_token %}` in forms?**
> CSRF token protects against Cross-Site Request Forgery attacks. Django validates this token on every POST request to ensure the form was submitted from your own site.

**Q4. What does the `safe` filter do? When should you be careful with it?**
> Renders raw HTML instead of escaping it. Never use `safe` on user-submitted content — it can lead to XSS attacks.

---

## 📒 Notebook Summary

```
Module 4 — Templates

extends     → inherit full layout from base.html
block       → define slots that child templates fill
include     → insert reusable snippets
block.super → keep parent content + add more

Key tags:
  if/elif/else, for/empty,
  url, csrf_token, load static

Key filters:
  truncatewords, date, timesince,
  upper/lower, default, safe

forloop.counter, forloop.first, forloop.last

Interview:
  - extends vs include
  - why csrf_token
  - safe filter risks
```

---

## ✏️ Quick Check

**Q1.** What's wrong with this template?

```html
{% block content %}
    <h1>All Posts</h1>
{% endblock %}
```

**Q2.** How would you show `"No posts available"` when the posts list is empty — without using an `{% if %}` tag?

**Q3.** What's the difference between these two?

```html
{{ post.created_at|date:'d M Y' }}
{{ post.created_at|timesince }}
```

---

*Answer when ready — then move on to **Module 5 — Forms & Validation!** 🚀*
