# 🎓 Module 3 — URLs & Views Done Right

> You already know basic URLs and FBVs from tutorials. This module fills all the gaps that tutorials skip — the stuff interviewers actually ask.

---

## 🎯 The Interview Questions

> *"How does Django's URL routing work?"*
> *"What's the difference between `path()` and `re_path()`?"*
> *"How do you structure URLs in a large project?"*

---

## 1️⃣ How URL Routing Actually Works

When a request comes in for `/posts/5/`, Django does this:

```python
# Django reads urls.py top to bottom:
urlpatterns = [
    path('admin/', admin.site.urls),            # does /posts/5/ match? NO
    path('', HomeView.as_view()),               # does /posts/5/ match? NO
    path('posts/', include('blog.urls')),       # does /posts/ match start? YES!
]

# Django strips 'posts/' and passes '5/' to blog/urls.py:
urlpatterns = [
    path('', PostListView.as_view()),             # does 5/ match? NO
    path('<int:pk>/', PostDetailView.as_view()),  # does 5/ match? YES! ✅
]
# pk=5 is extracted and passed to the view
```

---

## 2️⃣ `path()` Converters — Capturing URL Parameters

```python
# int — captures whole numbers
path('posts/<int:pk>/', PostDetailView.as_view())
# matches /posts/5/, /posts/100/
# does NOT match /posts/abc/

# str — captures any non-empty string (default)
path('posts/<str:username>/', UserPostsView.as_view())
# matches /posts/john/, /posts/any-text/

# slug — captures slug strings
path('posts/<slug:slug>/', PostDetailView.as_view())
# matches /posts/how-to-learn-django/
# does NOT match /posts/hello world/

# uuid — captures UUID strings
path('orders/<uuid:order_id>/', OrderDetailView.as_view())
# matches /orders/123e4567-e89b-12d3-a456-426614174000/

# path — captures entire path including slashes
path('files/<path:filepath>/', FileView.as_view())
# matches /files/documents/2024/report.pdf/
```

---

## 3️⃣ `path()` vs `re_path()`

```python
from django.urls import path, re_path

# path() — simple, readable, uses converters
path('posts/<int:pk>/', PostDetailView.as_view())

# re_path() — uses regex, more powerful but complex
re_path(r'^posts/(?P<pk>[0-9]+)/$', PostDetailView.as_view())
# same result, but harder to read
```

> 📒 **Interview note:** Use `path()` for most cases — it's cleaner and more readable. Use `re_path()` only when you need complex pattern matching that converters can't handle, like matching specific formats.

---

## 4️⃣ `include()` — Organising URLs in Large Projects

Never put all URLs in one file. Split by app:

```python
# myproject/urls.py (main)
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls')),         # all blog URLs
    path('accounts/', include('accounts.urls')), # all auth URLs
    path('shop/', include('shop.urls')),         # all shop URLs
]

# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'  # 👈 namespace — very important!

urlpatterns = [
    path('', views.PostListView.as_view(), name='post-list'),
    path('<int:pk>/', views.PostDetailView.as_view(), name='post-detail'),
    path('new/', views.PostCreateView.as_view(), name='post-create'),
]
```

---

## 5️⃣ URL Namespacing — Avoiding Name Conflicts

Without namespacing, two apps can have views with the same name and conflict:

```python
# blog/urls.py
path('', PostListView.as_view(), name='list')    # name='list'

# shop/urls.py
path('', ProductListView.as_view(), name='list') # also name='list'!
# conflict! which 'list' do you mean?
```

With namespacing:

```python
# blog/urls.py
app_name = 'blog'
path('', PostListView.as_view(), name='list')

# shop/urls.py
app_name = 'shop'
path('', ProductListView.as_view(), name='list')

# Reference them clearly in Python:
reverse('blog:list')   # → /blog/
reverse('shop:list')   # → /shop/

# Reference them clearly in templates:
{% url 'blog:list' %}
{% url 'shop:list' %}
```

> 📒 **Always use namespacing in real projects!**

---

## 6️⃣ FBVs Done Right

You know FBVs, but here's what tutorials skip:

### Handling Multiple HTTP Methods Cleanly

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required

@login_required  # protect the view
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)  # request.FILES for images!
        if form.is_valid():
            post = form.save(commit=False)  # don't save to DB yet
            post.author = request.user      # set the author
            post.save()                     # now save
            return redirect('blog:post-list')
    else:
        form = PostForm()  # empty form for GET

    return render(request, 'blog/form.html', {'form': form})
```

### 🎯 Most Asked: What does `commit=False` do?

```python
post = form.save(commit=False)
# Creates the Post object in memory
# Does NOT save to database yet
# Lets you modify fields before saving

post.author = request.user  # add extra data
post.save()                 # NOW save to database
```

---

## 7️⃣ The `request` Object — Everything It Contains

Tutorials only show `request.method` and `request.user`. Here's everything:

| Attribute | Description |
|-----------|-------------|
| `request.method` | `'GET'`, `'POST'`, `'PUT'`, `'DELETE'` |
| `request.user` | Logged-in user (or `AnonymousUser`) |
| `request.GET` | Query parameters dict → `/posts/?page=2` → `{'page': '2'}` |
| `request.POST` | Form data dict |
| `request.FILES` | Uploaded files |
| `request.path` | `'/posts/5/'` |
| `request.get_full_path()` | `'/posts/5/?page=2'` (includes query string) |
| `request.META` | Server/request metadata |
| `request.META['REMOTE_ADDR']` | User's IP address |
| `request.headers` | HTTP headers |
| `request.session` | Session data |
| `request.COOKIES` | Cookies |

### Reading Query Parameters with `request.GET`

```python
# URL: /posts/?page=2&category=python
def post_list(request):
    page = request.GET.get('page', 1)          # get 'page', default 1
    category = request.GET.get('category', '')  # get 'category', default ''

    posts = Post.objects.filter(category=category)
    return render(request, 'posts/list.html', {'posts': posts})
```

---

## 8️⃣ HttpResponse Types

```python
from django.shortcuts import render, redirect
from django.http import (
    HttpResponse,
    JsonResponse,
    HttpResponseNotFound,
    HttpResponseForbidden,
    HttpResponseBadRequest,
)

# Basic response
return HttpResponse("Hello World")                      # 200
return HttpResponse("Not found", status=404)            # any status code

# Render template
return render(request, 'template.html', {'key': val})  # 200 with template

# Redirect
return redirect('blog:post-list')                      # 302
return redirect('blog:post-detail', pk=5)              # 302 with args

# JSON response
return JsonResponse({'status': 'ok', 'count': 5})      # 200 with JSON
return JsonResponse({'error': 'not found'}, status=404)

# Error responses
return HttpResponseNotFound("Page not found")          # 404
return HttpResponseForbidden("Not allowed")            # 403
return HttpResponseBadRequest("Bad request")           # 400
```

> 📒 **Note `JsonResponse`** — this is your bridge to DRF. When you return JSON from a view, you're basically building an API manually. DRF automates this.

---

## 9️⃣ Decorators — Protecting FBVs

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_http_methods, require_GET, require_POST

# Only logged-in users
@login_required(login_url='/login/')
def my_view(request): ...

# Only users with a specific permission
@permission_required('blog.add_post')
def my_view(request): ...

# Only allow specific HTTP methods
@require_http_methods(['GET', 'POST'])
def my_view(request): ...

@require_GET   # only GET allowed
def my_view(request): ...

@require_POST  # only POST allowed
def my_view(request): ...
```

---

## 🎯 Common Interview Questions

**Q1. What's the difference between `path()` and `re_path()`?**
> `path()` uses simple converters like `<int:pk>`. `re_path()` uses regex for complex patterns. Use `path()` by default.

**Q2. What is URL namespacing and why use it?**
> Namespacing adds a prefix to URL names to avoid conflicts between apps. Set `app_name = 'blog'` and reference URLs as `blog:post-list`.

**Q3. What does `commit=False` do?**
> Creates a model instance in memory without saving to the database. Lets you add extra fields before saving.

**Q4. What's the difference between `request.GET` and `request.POST`?**
> `request.GET` contains URL query parameters. `request.POST` contains form submission data.

---

## 📒 Notebook Summary

```
Module 3 — URLs & Views

URL routing: top to bottom, first match wins
Converters:  int, str, slug, uuid, path
path() vs re_path() → use path() always
include()   → splits URLs by app
app_name    → namespacing → avoid conflicts

request object:
  method, user, GET, POST,
  FILES, path, session, COOKIES

commit=False  → creates object, doesn't save to DB
JsonResponse  → returns JSON (bridge to DRF)

Decorators: @login_required, @require_POST
```

---

## ✏️ Quick Check

**Q1.** What's wrong with this URL pattern?

```python
urlpatterns = [
    path('posts/<int:pk>/', PostDetailView.as_view()),
    path('posts/new/', PostCreateView.as_view()),
]
```

**Q2.** A user visits `/posts/?search=django` — how do you get the search value inside your view?

**Q3.** What does this return and what's the status code?

```python
return JsonResponse({'error': 'not found'}, status=404)
```

---

*Answer when ready — then move on to **Module 4 — Templates & Template Inheritance!** 🚀*
