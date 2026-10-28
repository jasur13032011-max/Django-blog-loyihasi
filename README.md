# Django-blog-loyihasi
1. settings.py Sozlamalari
Loyihangizda avtorizatsiyadan so'ng qayerga yo'naltirilishini ko'rsatib qo'ying:

Python
# settings.py oxiriga qo'shing:
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
2. models.py — Modellar
Muallif (User ga ForeignKey), teglar (Tag ga ManyToManyField), sarlavha, matn va yaratilgan vaqt maydonlarini o'z ichiga olgan modellar:

Python
# blog/models.py
from django.contrib.auth.models import User
from django.db import models


class Tag(models.Model):
  name = models.CharField(max_length=50, unique=True)

  def __str__(self):
    return self.name


class Post(models.Model):
  title = models.CharField(max_length=200, verbose_name="Sarlavha")
  body = models.TextField(verbose_name="Matn")
  author = models.ForeignKey(
      User, on_delete=models.CASCADE, verbose_name="Muallif"
  )
  tags = models.ManyToManyField(Tag, blank=True, verbose_name="Teglar")
  created_at = models.DateTimeField(
      auto_now_add=True, verbose_name="Yaratilgan vaqt"
  )

  def __str__(self):
    return self.title
3. admin.py — Admin panel sozlamalari
list_display va search_fields talablarini bajarish:

Python
# blog/admin.py
from django.contrib import admin
from .models import Post, Tag

admin.site.register(Tag)


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
  list_display = ("title", "author", "created_at")
  search_fields = ("title", "body")
4. forms.py — Forma (Ixtiyoriy, lekin CBV bilan qulaylik uchun)
Python
# blog/forms.py
from django import forms
from .models import Post


class PostForm(forms.ModelForm):

  class Meta:
    model = Post
    fields = ["title", "body", "tags"]
5. views.py — CBV (Class-Based Views) va form_valid()
ListView, DetailView va LoginRequiredMixin bilan CreateView (muallifni avtomatik biriktiruvchi form_valid bilan):

Python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DetailView, ListView
from .models import Post


class PostListView(ListView):
  model = Post
  template_name = "blog/post_list.html"
  context_object_name = "posts"
  ordering = ["-created_at"]


class PostDetailView(DetailView):
  model = Post
  template_name = "blog/post_detail.html"
  context_object_name = "post"


class PostCreateView(LoginRequiredMixin, CreateView):
  model = Post
  fields = ["title", "body", "tags"]
  template_name = "blog/post_form.html"
  success_url = reverse_lazy("post-list")

  def form_valid(self, form):
    # Muallifni avtomatik ravishda hozirgi kirgan foydalanuvchiga tenglaymiz
    form.instance.author = self.request.user
    return super().form_valid(form)
6. urls.py — URL manzillar
Loyiha va ilova URL'larini bog'lash:

Python
# blog/urls.py
from django.urls import path
from .views import PostCreateView, PostDetailView, PostListView

urlpatterns = [
    path("", PostListView.as_view(), name="post-list"),
    path("post/<int:pk>/", PostDetailView.as_view(), name="post-detail"),
    path("post/new/", PostCreateView.as_view(), name="post-create"),
]
Va asosiy urls.py (loyiha papkasidagi):

Python
# core/urls.py (yoki loyiha nomi)
from django.contrib import admin
from django.contrib.auth import views as auth_views
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    # Built-in autentifikatsiya uchun (login/logout)
    path(
        "accounts/login/",
        auth_views.LoginView.as_view(template_name="blog/login.html"),
        name="login",
    ),
    path(
        "accounts/logout/",
        auth_views.LogoutView.as_view(),
        name="logout",
    ),
    path("", include("blog.urls")),
]
7. Shablonlar (Templates)
Fayllar strukturasi quyidagicha bo'lishi kerak:

Plaintext
templates/
    base.html
    blog/
        post_list.html
        post_detail.html
        post_form.html
        login.html
templates/base.html (Asosiy shablon)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Mening Blogim{% endblock %}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f9f9f9; color: #333; }
        nav { background: #333; color: #fff; padding: 10px; margin-bottom: 20px; }
        nav a { color: #fff; margin-right: 15px; text-decoration: none; }
        .container { background: #fff; padding: 20px; border-radius: 5px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        .btn { display: inline-block; background: #007BFF; color: white; padding: 8px 12px; text-decoration: none; border-radius: 4px; }
    </style>
</head>
<body>
    <nav>
        <a href="{% url 'post-list' %}">Bosh sahifa</a>
        {% if user.is_authenticated %}
            <a href="{% url 'post-create' %}">Yangi post yozish</a>
            <span>Xush kelibsiz, <b>{{ user.username }}</b>!</span>
            <form action="{% url 'logout' %}" method="post" style="display:inline;">
                {% csrf_token %}
                <button type="submit" style="background:none; border:none; color:white; cursor:pointer; text-decoration:underline;">Chiqish</button>
            </form>
        {% else %}
            <a href="{% url 'login' %}">Kirish</a>
        {% endif %}
    </nav>

    <div class="container">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
templates/blog/post_list.html
HTML
{% extends 'base.html' %}
{% block content %}
<h1>Maqolalar ro'yxati</h1>
<ul>
    {% for post in posts %}
        <li>
            <h2><a href="{% url 'post-detail' post.pk %}">{{ post.title }}</a></h2>
            <p>Muallif: {{ post.author.username }} | Sana: {{ post.created_at|date:"Y-m-d H:i" }}</p>
        </li>
    {% empty %}
        <p>Hozircha maqolalar mavjud emas.</p>
    {% endfor %}
</ul>
{% endblock %}
templates/blog/post_detail.html
HTML
{% extends 'base.html' %}
{% block content %}
<h1>{{ post.title }}</h1>
<p><small>Muallif: {{ post.author.username }} | Yaratilgan vaqt: {{ post.created_at }}</small></p>
<div>
    {{ post.body|linebreaks }}
</div>
<p>
    <strong>Teglar:</strong> 
    {% for tag in post.tags.all %}
        <span>#{{ tag.name }}</span>
    {% empty %}
        Teglar yo'q
    {% endfor %}
</p>
<br>
<a href="{% url 'post-list' %}" class="btn">Ortga qaytish</a>
{% endblock %}
templates/blog/post_form.html (Post yaratish sahifasi)
HTML
{% extends 'base.html' %}
{% block content %}
<h1>Yangi post yaratish</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit" class="btn">Saqlash</button>
</form>
{% endblock %}
templates/blog/login.html (Kirish sahifasi)
HTML
{% extends 'base.html' %}
{% block content %}
<h1>Tizimga kirish</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit" class="btn">Kirish</button>
</form>
{% endblock %}
Yakuniy sinovni ishga tushirish:
Migratsiyalarni yarating va qo'llang:

Bash
python manage.py makemigrations
python manage.py migrate
Superuser (admin) yarating:

Bash
python manage.py createsuperuser
Serverni ishga tushiring:

Bash
python manage.py runserver
Brauzerda [http://127.0.0.1:8000/](http://127.0.0.1:8000/) manziliga o'tib, barcha talablar (admin panel, CBV orqali post yaratish, form_valid orqali muallif avtomatik birikishi, avtorizatsiya va shablonlar merosi) to'g'ri ishlashini tekshirishingiz mumkin!
