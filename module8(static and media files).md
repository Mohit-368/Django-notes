# 🎓 Module 8 — Static Files & Media Files

> This trips up almost every beginner in real projects. Understanding the difference between static and media files is **crucial** before deployment.

---

## 🎯 The Interview Questions

> *"What's the difference between static files and media files?"*
> *"How do you handle file uploads in Django?"*
> *"How do static files work in production?"*

---

## 1️⃣ The Big Difference — Static vs Media

| | Static Files | Media Files |
|-|-------------|-------------|
| **Source** | You provide them | Users upload them |
| **Examples** | CSS, JavaScript, images | Profile pictures, documents, post images |
| **Changes** | Don't change | Change constantly |
| **Location** | Part of your codebase | Stored separately |

They are handled **completely differently** in Django.

---

## 2️⃣ Static Files — Complete Setup

### Settings

```python
# settings.py

STATIC_URL = '/static/'                        # URL to access static files

STATICFILES_DIRS = [
    BASE_DIR / 'static',                       # your project-level static folder
]

STATIC_ROOT = BASE_DIR / 'staticfiles'         # where collectstatic puts files
```

### Folder Structure

```
myproject/
├── static/                  ← project-level static files
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── app.js
│   └── images/
│       └── logo.png
│
└── blog/
    └── static/              ← app-level static files
        └── blog/            ← always nest in app name folder!
            ├── css/
            │   └── blog.css
            └── js/
                └── blog.js
```

### Using Static Files in Templates

```html
{% load static %}  <!-- must load first! -->

<link rel="stylesheet" href="{% static 'css/style.css' %}">
<script src="{% static 'js/app.js' %}"></script>
<img src="{% static 'images/logo.png' %}" alt="Logo">

<!-- App-level static: -->
<link rel="stylesheet" href="{% static 'blog/css/blog.css' %}">
```

---

## 3️⃣ Media Files — Complete Setup

### Settings

```python
# settings.py

MEDIA_URL = '/media/'               # URL to access uploaded files
MEDIA_ROOT = BASE_DIR / 'media'     # where uploaded files are stored
```

### URLs — Serve Media in Development

```python
# urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
# 👆 serves media files in development only!
# In production — your web server (Nginx) handles this
```

### Model with File Upload

```python
# models.py
class Post(models.Model):
    title   = models.CharField(max_length=200)
    content = models.TextField()

    image = models.ImageField(
        upload_to='posts/images/',    # saved to media/posts/images/
        blank=True,
        null=True
    )
    document = models.FileField(
        upload_to='posts/documents/', # saved to media/posts/documents/
        blank=True,
        null=True
    )
```

### Dynamic Upload Paths — Organise by Date

```python
import os
from datetime import datetime

def post_image_path(instance, filename):
    date = datetime.now()
    return os.path.join('posts', str(date.year), str(date.month),
                        str(date.day), filename)
    # generates: posts/2024/03/09/filename.jpg

class Post(models.Model):
    image = models.ImageField(upload_to=post_image_path)
```

---

## 4️⃣ Handling File Uploads in Views

```python
# views.py
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)
        #                              👆 must include request.FILES!
        if form.is_valid():
            form.save()
            return redirect('blog:post-list')
    else:
        form = PostForm()
    return render(request, 'blog/form.html', {'form': form})
```

### Template — Must Add `enctype`

```html
<!-- enctype is REQUIRED for file uploads -->
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Save</button>
</form>
```

### 🎯 Most Asked: What if you forget `enctype="multipart/form-data"`?

> Without `enctype`, the browser sends the form as plain text. File data is not included in the request, `request.FILES` is empty, and the upload silently fails — **no error is shown.** Always add `enctype` to any form with file uploads!

---

## 5️⃣ Displaying Uploaded Files in Templates

```html
<!-- Always check if file exists first -->
{% if post.image %}
    <img src="{{ post.image.url }}" alt="{{ post.title }}">
{% else %}
    <img src="{% static 'images/default.jpg' %}" alt="Default">
{% endif %}

<!-- File download link -->
{% if post.document %}
    <a href="{{ post.document.url }}" download>Download Document</a>
{% endif %}
```

### File Object Properties

| Property | Returns |
|----------|---------|
| `post.image.url` | URL to access the file |
| `post.image.name` | Filename with path |
| `post.image.size` | File size in bytes |

---

## 6️⃣ Validating Uploads in Forms

```python
# forms.py
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'image']

    def clean_image(self):
        image = self.cleaned_data.get('image')

        if image:
            # Check file size — max 2MB
            if image.size > 2 * 1024 * 1024:
                raise forms.ValidationError("Image size must be under 2MB.")

            # Check file extension
            allowed = ['jpg', 'jpeg', 'png', 'gif']
            ext = image.name.split('.')[-1].lower()
            if ext not in allowed:
                raise forms.ValidationError("Only JPG, PNG and GIF files allowed.")

        return image
```

---

## 7️⃣ Deleting Files — Important Gotcha!

```python
# ⚠️ Django does NOT delete files when you delete objects!
post = Post.objects.get(pk=1)
post.delete()
# post deleted from DB ✅
# but media/posts/images/myimage.jpg still exists on disk! 😱
```

Fix — delete files manually using signals:

```python
# signals.py
from django.db.models.signals import post_delete, pre_save
from django.dispatch import receiver
from .models import Post
import os

# Delete file when post is deleted
@receiver(post_delete, sender=Post)
def delete_post_image(sender, instance, **kwargs):
    if instance.image and os.path.isfile(instance.image.path):
        os.remove(instance.image.path)

# Delete OLD file when image is updated
@receiver(pre_save, sender=Post)
def delete_old_image(sender, instance, **kwargs):
    if not instance.pk:
        return  # new object, nothing to delete

    try:
        old_image = Post.objects.get(pk=instance.pk).image
    except Post.DoesNotExist:
        return

    if old_image and old_image != instance.image:
        if os.path.isfile(old_image.path):
            os.remove(old_image.path)
```

---

## 8️⃣ Pillow — Required for `ImageField`

```bash
pip install Pillow
```

Without Pillow you get:

```
Cannot use ImageField because Pillow is not installed.
```

---

## 9️⃣ Static Files in Production — `collectstatic`

In **development** — Django serves static files automatically.

In **production** — Django does NOT serve static files. Your web server does.

```bash
# Copies ALL static files from all apps into STATIC_ROOT:
python manage.py collectstatic
```

```
myproject/
└── staticfiles/          ← everything ends up here
    ├── admin/            ← Django admin static files
    │   ├── css/
    │   └── js/
    ├── css/              ← your static files
    │   └── style.css
    └── blog/
        └── css/
            └── blog.css
```

Your web server (Nginx) then serves files directly from `staticfiles/`.

---

## 🔟 Complete Settings Reference

```python
# settings.py

# ── STATIC FILES ──────────────────────────────────────
STATIC_URL      = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static']  # your project static files
STATIC_ROOT     = BASE_DIR / 'staticfiles' # collectstatic destination

# ── MEDIA FILES ───────────────────────────────────────
MEDIA_URL  = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'            # uploaded files stored here
```

```python
# urls.py — development only
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT) \
  + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

---

## 🎯 Common Interview Questions

**Q1. What's the difference between static and media files?**
> Static files are project files like CSS/JS that don't change. Media files are user-uploaded files. They use different settings and are handled differently.

**Q2. What does `collectstatic` do?**
> Copies all static files from all apps into the `STATIC_ROOT` folder so the web server can serve them in production.

**Q3. Why do we add `request.FILES` in the view?**
> File uploads are sent separately from form text data. `request.POST` has text data, `request.FILES` has file data. Both are needed for file upload forms.

**Q4. What happens if you forget `enctype="multipart/form-data"`?**
> Files are not sent with the request. `request.FILES` is empty. The upload silently fails with no error.

**Q5. Does Django delete files when you delete a model instance?**
> No. Files remain on disk. You must manually delete them using signals or custom delete logic.

---

## 📒 Notebook Summary

```
Module 8 — Static & Media Files

STATIC (your files — CSS, JS, images):
  STATIC_URL       = '/static/'
  STATICFILES_DIRS = [BASE_DIR / 'static']
  STATIC_ROOT      = BASE_DIR / 'staticfiles'
  collectstatic    → copies all to STATIC_ROOT
  {% load static %} then {% static 'file.css' %}

MEDIA (user uploaded files):
  MEDIA_URL  = '/media/'
  MEDIA_ROOT = BASE_DIR / 'media'
  add to urls.py in development
  ImageField needs Pillow installed

File upload checklist:
  ✅ request.FILES in view
  ✅ enctype="multipart/form-data" in template
  ✅ MEDIA_URL and MEDIA_ROOT in settings
  ✅ media URL added in urls.py
  ✅ Pillow installed

Django does NOT auto-delete files on object delete!
Use signals to clean up orphaned files.

Interview:
  static vs media
  collectstatic purpose
  enctype missing → silent fail
  Pillow required for ImageField
```

---

## ✏️ Quick Check

**Q1.** A user uploads a profile picture but it's not saving. What are the 3 things you'd check first?

**Q2.** What's wrong with this template?

```html
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button>Upload</button>
</form>
```

**Q3.** What's the difference between `STATIC_ROOT` and `STATICFILES_DIRS`?

---

*Answer when ready — then move on to **Module 9 — Django Messages & Redirects!** 🚀*
