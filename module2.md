# 🎓 Module 2 — Models & ORM Deep Dive

> This is the **most interviewed Django topic.** Every company asks ORM questions. Let's master it completely.

---

## What is a Model?

A model is a **Python class that represents a database table.**

```python
# This Python class:
class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()

# Becomes this database table:
# ┌────┬──────────────────┬─────────────────┐
# │ id │ title            │ content         │
# ├────┼──────────────────┼─────────────────┤
# │ 1  │ Learn Django     │ Django is...    │
# │ 2  │ Python tips      │ Here are...     │
# └────┴──────────────────┴─────────────────┘
```

You write Python — Django handles the SQL. You never need to write `CREATE TABLE` manually.

---

## What is ORM?

**ORM = Object Relational Mapper**

It translates Python code into SQL queries:

```python
# You write Python:
Post.objects.all()
# Django translates to SQL:
# SELECT * FROM blog_post;

# You write Python:
Post.objects.filter(title__icontains='django')
# Django translates to SQL:
# SELECT * FROM blog_post WHERE title LIKE '%django%';
```

> 📒 **Interview note:** ORM lets you work with databases without writing SQL. But knowing what SQL it generates helps you write efficient queries — interviewers love this!

---

## 1️⃣ Field Types — The Complete Guide

```python
class Post(models.Model):
    # TEXT fields
    title = models.CharField(max_length=200)    # short text, max_length required
    content = models.TextField()                # long text, no max_length needed
    slug = models.SlugField(unique=True)        # URL-friendly text

    # NUMBER fields
    age = models.IntegerField()                 # whole numbers
    price = models.DecimalField(max_digits=10, decimal_places=2)  # money
    rating = models.FloatField()                # decimal numbers

    # BOOLEAN field
    is_published = models.BooleanField(default=False)

    # DATE/TIME fields
    created_at = models.DateTimeField(auto_now_add=True)  # set once on creation
    updated_at = models.DateTimeField(auto_now=True)      # updates every save
    publish_date = models.DateField()                     # date only, no time

    # FILE fields
    image = models.ImageField(upload_to='images/')
    document = models.FileField(upload_to='docs/')

    # EMAIL & URL
    email = models.EmailField()    # validates email format
    website = models.URLField()    # validates URL format
```

### 🎯 Most Asked: `auto_now` vs `auto_now_add`

```python
created_at = models.DateTimeField(auto_now_add=True)
# Sets timestamp ONCE when object is first created
# Never changes after that
# ✅ use for "created at"

updated_at = models.DateTimeField(auto_now=True)
# Updates timestamp EVERY time object is saved
# ✅ use for "last modified"
```

---

## 2️⃣ Field Options

Every field can accept these options:

```python
class Post(models.Model):
    title = models.CharField(
        max_length=200,
        null=True,       # allows NULL in database
        blank=True,      # allows empty in forms
        default='',      # default value
        unique=True,     # no duplicates allowed
        db_index=True,   # creates database index (faster queries)
        verbose_name='Post Title'  # human-readable name in admin
    )
```

### 🎯 Most Asked: `null=True` vs `blank=True`

```python
null=True   # database level — stores NULL in DB column
blank=True  # form/validation level — allows empty form submission
```

| Combination | Meaning |
|-------------|---------|
| `null=False, blank=False` | Required everywhere (default) |
| `null=True, blank=True` | Optional everywhere |
| `blank=True` only | Optional in forms, empty string stored in DB |
| `null=True` only | NULL in DB, but form still requires input ⚠️ |

> 📒 **Interview note:** For string fields, use `blank=True` alone — empty string `""` is better than `NULL` for text. For non-string fields (integers, dates), use `null=True, blank=True` together.

---

## 3️⃣ Relationships — The Big Three

### ForeignKey — Many to One

```python
# Many posts can belong to ONE user
class Post(models.Model):
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,  # if user deleted, delete their posts too
        related_name='posts'       # enables user.posts.all()
    )
```

### ManyToManyField — Many to Many

```python
# A post can have many tags; a tag can belong to many posts
class Post(models.Model):
    tags = models.ManyToManyField('Tag', blank=True)

class Tag(models.Model):
    name = models.CharField(max_length=50)
```

### OneToOneField — One to One

```python
# One user has exactly one profile
class Profile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/')
```

### 🎯 Most Asked: `on_delete` options

| Option | Behaviour |
|--------|-----------|
| `CASCADE` | Delete related objects too |
| `PROTECT` | Prevent deletion if related objects exist |
| `SET_NULL` | Set field to NULL (requires `null=True`) |
| `SET_DEFAULT` | Set field to its default value |

---

## 4️⃣ ORM Queries — The Complete Guide

### Basic Queries

```python
# GET ALL
Post.objects.all()                        # SELECT * FROM blog_post;

# GET ONE — raises exception if 0 or 2+ results
Post.objects.get(pk=1)                    # SELECT * FROM blog_post WHERE id=1;

# FILTER — returns queryset, never raises exception
Post.objects.filter(is_published=True)

# EXCLUDE — opposite of filter
Post.objects.exclude(is_published=True)

# GET OR 404 — always use this in views!
post = get_object_or_404(Post, pk=1)
```

### Field Lookups

```python
Post.objects.filter(title='Django')           # exact match (default)
Post.objects.filter(title__iexact='django')   # case-insensitive exact

Post.objects.filter(title__contains='Django') # case-sensitive contains
Post.objects.filter(title__icontains='django')# case-insensitive contains

Post.objects.filter(views__gt=100)            # greater than
Post.objects.filter(views__gte=100)           # greater than or equal
Post.objects.filter(views__lt=100)            # less than
Post.objects.filter(views__lte=100)           # less than or equal

Post.objects.filter(id__in=[1, 2, 3])         # in a list

Post.objects.filter(title__startswith='How')
Post.objects.filter(title__endswith='Django')

Post.objects.filter(created_at__year=2024)
Post.objects.filter(created_at__month=1)
Post.objects.filter(created_at__date='2024-01-01')
```

### Chaining, Ordering & Limiting

```python
# Chain as many filters as needed
Post.objects.filter(is_published=True) \
            .filter(author=request.user) \
            .exclude(title__icontains='draft') \
            .order_by('-created_at')

# Ordering
Post.objects.order_by('title')                     # A → Z
Post.objects.order_by('-title')                    # Z → A
Post.objects.order_by('author', '-created_at')     # multiple fields

# Limiting
Post.objects.all()[:5]          # first 5  → LIMIT 5
Post.objects.all()[5:10]        # posts 6–10 → OFFSET 5 LIMIT 5
Post.objects.all().first()      # first object
Post.objects.all().last()       # last object
```

---

## 5️⃣ Creating, Updating & Deleting

### Create

```python
# Method 1 — create()
post = Post.objects.create(
    title='My Post',
    content='Content here',
    author=request.user
)

# Method 2 — save()
post = Post(title='My Post', content='Content here')
post.author = request.user
post.save()  # hits the database
```

### Update

```python
# Update one object
post = Post.objects.get(pk=1)
post.title = 'New Title'
post.save()

# Update many at once (more efficient!)
Post.objects.filter(is_published=False).update(is_published=True)
# UPDATE blog_post SET is_published=True WHERE is_published=False;
```

### Delete

```python
# Delete one
post = Post.objects.get(pk=1)
post.delete()

# Delete many
Post.objects.filter(is_published=False).delete()
```

---

## 6️⃣ Aggregations

```python
from django.db.models import Count, Sum, Avg, Max, Min

Post.objects.count()                          # total count

Post.objects.aggregate(Avg('views'))          # {'views__avg': 150.5}
Post.objects.aggregate(Sum('views'))          # {'views__sum': 15050}
Post.objects.aggregate(Max('views'))          # {'views__max': 5000}

# Count posts per author
Post.objects.values('author') \
            .annotate(post_count=Count('id'))
# [{'author': 1, 'post_count': 5}, {'author': 2, 'post_count': 3}]
```

---

## 7️⃣ Model Methods & Meta

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title  # shows title instead of "Post object (1)"

    def short_content(self):
        return self.content[:100] + '...'

    class Meta:
        ordering = ['-created_at']       # default ordering
        verbose_name = 'Post'            # singular name in admin
        verbose_name_plural = 'Posts'    # plural name in admin
        db_table = 'blog_posts'          # custom table name
```

---

## 🎯 Common Interview Questions

**Q1. What's the difference between `get()` and `filter()`?**
> `get()` returns a single object and raises an exception if none or multiple are found. `filter()` returns a queryset that can be empty — it never raises an exception.

**Q2. What's the difference between `null=True` and `blank=True`?**
> `null=True` is database-level (stores NULL). `blank=True` is form validation-level (allows empty input).

**Q3. What does `related_name` do in ForeignKey?**
> Lets you access related objects from the reverse side. `related_name='posts'` on an author ForeignKey lets you do `user.posts.all()`.

**Q4. What's the difference between `update()` and `save()`?**
> `save()` updates one object after loading it into Python first. `update()` updates many objects directly in the database — much more efficient for bulk updates.

---

## 📒 Notebook Summary

```
Module 2 — Models & ORM

Model = Python class = Database table
ORM   = translates Python to SQL

Field types:
  CharField, TextField, IntegerField, BooleanField,
  DateTimeField, ForeignKey, ManyToManyField, OneToOneField

Key differences:
  - auto_now vs auto_now_add
  - null=True vs blank=True
  - get() vs filter()
  - save() vs update()

Common lookups:
  __icontains, __gt, __lt, __in, __startswith, __year

on_delete options:
  CASCADE, PROTECT, SET_NULL, SET_DEFAULT

Interview: null vs blank, get vs filter, update vs save
```

---

## ✏️ Quick Check

**Q1.** What's wrong with this code?

```python
post = Post.objects.get(is_published=True)
```

**Q2.** Write a query to get all posts by `request.user` that contain `"django"` in the title, ordered by newest first.

**Q3.** What's the difference between these two approaches?

```python
# Approach A
Post.objects.filter(author=user).update(is_published=True)

# Approach B
posts = Post.objects.filter(author=user)
for post in posts:
    post.is_published = True
    post.save()
```

---

*Answer when ready — then move on to **Module 3 — URLs & Views Done Right!** 🚀*
