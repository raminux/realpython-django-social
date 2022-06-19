# Realpython.com Django Social(Dweet)

## Create a virtual environment and install Django
```bash
$> mkdir django-social-platform
$> cd django-social-platform
$> pipenv shell
$> pip install django==3.2.5
```

## Create a Django project and an app
```bash
$> django-admin startproject config .
$> python managen.py startapp dwitter
```
Also, register the newly installed app (dwitter) to `INSTALLED_APP` in `config/setting.py` as
```python
# config/settings.py

# ...

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "dwitter",
]
```

## Customize the Django admin interface
```bash
$> python manage.py migrte
$> python manage.py createsuperuser
username: admin
email: admin@dwitter.com
password: admin
```
Add the following codes to the `dwitter/admin.py`
```python
# dwitter/admin.py

from django.contrib.auth.models import User, Group

class UserAdmin(admin.ModelAdmin):
    model = User
    # Only display the "username" field
    fields = ["username"]

admin.site.unregister(User)
admin.site.register(User, UserAdmin)
admin.site.unregister(Group)
```

## Create users for the app via admin interface
```bash
$> python manage.py runserver
```
Now, define a Profile model as below:
```python
# dwitter/models.py

from django.db import models
from django.contrib.auth.models import User

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    follows = models.ManyToManyField(
        "self",
        related_name="followed_by",
        symmetrical=False,
        blank=True
    )
```
Now, migrate the new model
```bash
$> python manage.py makemigrations
$> python manage.py migrate
```

It's time to register the Profile model with the admin interface:
```python
# dwitter/admin.py

# ...
from .models import Profile

# ...
admin.site.register(Profile)
```
## Display Profile Information Within the User Admin Page
As the User and Profile models are highly connected, it is a good idea to see them in one place.
```python
# dwitter/admin.py

# ...

class ProfileInline(admin.StackedInline):
    model = Profile

class UserAdmin(admin.ModelAdmin):
    model = User
    fields = ["username"]
    inlines = [ProfileInline]

admin.site.unregister(User)
admin.site.register(User, UserAdmin)
admin.site.unregister(Group)
# Remove: admin.site.register(Profile)
```
Also, to see profiles clearly, add the following function to the Profile model:
```python
def __str__(self):
    return self.user.username
```
## Coordinate Users and Profiles With a Signal
Since there is a one to one relationship between the Profile and the User model, it is necessary to create a profile for each user. This task can be done automatically usign **==Django Signals==**.
```python
# dwitter/models.py

from django.db.models.signals import post_save

# ...

def create_profile(sender, instance, created, **kwargs):
    if created:
        user_profile = Profile(user=instance)
        user_profile.save()

# Create a Profile for each new user.
post_save.connect(create_profile, sender=User)
```
Also, each user must be able to see their own dweets. So, we need to add each user's profile to the follows column. So let's do it:
```python 
def create_profile(sender, instance, created, **kwargs):
    user_profile = Profile(user=instance)
    user_profile.save()
    user_profile.follows.set([instance.profile.id])
    # user_profile.follows.add(instance.profile)
    user_profile.save()
```

## Refactor Your Code Using a Decorator
It is possible to use decorators for signaling as below:
```python
# dwitter/models.py

from django.db.models.signals import post_save
from django.dispatch import receiver

# ...

@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        user_profile = Profile(user=instance)
        user_profile.save()
        user_profile.follows.add(instance.profile)
        user_profile.save()

# Remove: post_save.connect(create_profile, sender=User)
```

## Create a Base Template With Bulma CSS Framework
First create a directory named `templates` and add `base.html` in it as
```bash
$> mkdir templates
$> touch templates/base.html
```
The content of `base.html` is as
```html
<!-- templates/base.html -->

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <link rel="stylesheet"
          href="https://cdn.jsdelivr.net/npm/bulma@0.9.3/css/bulma.min.css">
    <title>Dwitter</title>
</head>
<body>
    <section class="hero is-small is-success mb-4">
        <div class="hero-body">
            <h1 class="title is-1">Dwitter</h1>
            <p class="subtitle is-4">
                Your tiny social network built with Django
            </p>
        </div>
    </section>

    <div class="container">
        <div class="columns">

            {% block content %}

            {% endblock content %}

        </div>
    </div>
</body>
</html>
```

## View Your Base Template
To veiw a specific page, we need to define routes to them. Start by adding routes of the dwitter application to the main routes of the project as below
```python
# config/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("", include("dwitter.urls")),
    path("admin/", admin.site.urls),
]
```
Now, create `dwitter/urls.py` and add the routes of the application inside it
```python
# dwitter/urls.py

from django.urls import path
from .views import dashboard

app_name = "dwitter"

urlpatterns = [
    path("", dashboard, name="dashboard"),
]
```
Now, add the `dashboard` method to the `dwitter/views.py` file
```python
# dwitter/views.py

from django.shortcuts import render

def dashboard(request):
    return render(request, "base.html")
```

## List All User Profiles on the Front End of Your Django App
First, define the route to see all user profiles
```python
# dwitter/urls.py

from django.urls import path
from .views import dashboard, profile_list

app_name = "dwitter"

urlpatterns = [
    path("", dashboard, name="dashboard"),
    path("profile_list/", profile_list, name="profile_list"),
]
```
Then, write the code to find profiles
```python
# dwitter/views.py

from django.shortcuts import render
from .models import Profile


def profile_list(request):
    profiles = Profile.objects.exclude(user=request.user)
    return render(request, "dwitter/profile_list.html", {"profiles": profiles})
```
Next, code the `.html` file
```html

{% extends 'base.html' %}

{% block content %}

<div class="column">

{% for profile in profiles %}

    <div class="block">
      <div class="card">
        <a href="#">
          <div class="card-content">
            <div class="media">
              <div class="media-left">
                <figure class="image is-48x48">
                  <img src="https://bulma.io/images/placeholders/96x96.png"
                       alt="Placeholder image">
                </figure>
              </div>
              <div class="media-content">
                <p class="title is-4">
                  {{ profile.user.username }}
                </p>
                <p class="subtitle is-6">
                  @{{ profile.user.username|lower }}
                </p>
              </div>
            </div>
          </div>
        </a>
      </div>
    </div>

{% endfor %}

</div>

{% endblock content %}
```

## Access Individual Profile Pages
As before, we start by adding the route
```python
# dwitter/urls.py
from .views import dashboard, profile_list, profile


urlpatterns = [
    ...
    path("profile/<int:pk>", profile, name="profile"),
]
```
Then, write the profile view as
```python
# dwitter/views.py

# ...

def profile(request, pk):
    profile = Profile.objects.get(pk=pk)
    return render(request, "dwitter/profile.html", {"profile": profile})
```
and of course, we need to create a template for profile representation
```html
{% extends 'base.html' %}

{% block content %}

<div class="column">

    <div class="block">
    <h1 class="title is-1">
        {{profile.user.username|upper}}'s Dweets
    </h1>
    </div>

</div>

<div class="column is-one-third">

    <div class="block">
        <a href="{% url 'dwitter:profile_list' %}">
            <button class="button is-dark is-outlined is-fullwidth">
                All Profiles
            </button>
        </a>
    </div>

    <div class="block">
        <h3 class="title is-4">
            {{profile.user.username}} follows:
        </h3>
        <div class="content">
            <ul>
            {% for following in profile.follows.all %}
                <li>
                    <a href="{% url 'dwitter:profile' following.id %}">
                        {{ following }}
                    </a>
                </li>
            {% endfor %}
            </ul>
        </div>
    </div>

    <div class="block">
        <h3 class="title is-4">
            {{profile.user.username}} is followed by:
        </h3>
        <div class="content">
            <ul>
            {% for follower in profile.followed_by.all %}
                <li>
                    <a href="{% url 'dwitter:profile' follower.id %}">
                        {{ follower }}
                    </a>
                </li>
            {% endfor %}
            </ul>
        </div>
    </div>

</div>

{% endblock content %}
```

## Follow and Unfollow Other Profiles
1. Add buttons to the template
```html
<!-- dwitter/profile.html -->

<form method="post">
    {% csrf_token %}
    <div class="buttons has-addons">
    {% if profile in user.profile.follows.all %}
        <button class="button is-success is-static">Follow</button>
        <button class="button is-danger" name="follow" value="unfollow">
            Unfollow
        </button>
    {% else %}
        <button class="button is-success" name="follow" value="follow">
            Follow
        </button>
        <button class="button is-danger is-static">Unfollow</button>
    {% endif %}
    </div>
</form>
```
2. Add the logic to the `views.py` file
```python
# dwitter/views.py

# ...

def profile(request, pk):
    if not hasattr(request.user, 'profile'):
        missing_profile = Profile(user=request.user)
        missing_profile.save()

    profile = Profile.objects.get(pk=pk)
    if request.method == "POST":
        current_user_profile = request.user.profile
        data = request.POST
        action = data.get("follow")
        if action == "follow":
            current_user_profile.follows.add(profile)
        elif action == "unfollow":
            current_user_profile.follows.remove(profile)
        current_user_profile.save()
    return render(request, "dwitter/profile.html", {"profile": profile})
```

## Create the Back-End Logic for Dweets
1. Make the model
```python
# dwitter/models.py

# ...

class Dweet(models.Model):
    user = models.ForeignKey(
        User, related_name="dweets", on_delete=models.DO_NOTHING
    )
    body = models.CharField(max_length=140)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return (
            f"{self.user} "
            f"({self.created_at:%Y-%m-%d %H:%M}): "
            f"{self.body[:30]}..."
        )
```
2. Migrate the Dweet model
```bash
$> python manage.py makemigrations
$> python manage.py migrate
```
3. Add Dweet to the Admin panel
```python
# dwitter/admin.py
from .models import Dweet, Profile

# ...

admin.site.register(Dweet)
```

## Display Dweets on the Front End
1. Show dweets on profile page
```html
<div class="content">
    {% for dweet in profile.user.dweets.all %}
        <div class="box">
            {{ dweet.body }}
            <span class="is-small has-text-grey-light">
                ({{ dweet.created_at }})
            </span>
        </div>
    {% endfor %}
</div>
```

## Create a Dashboard View
1. Add this code to the `dashboard.html`
```html
{% extends 'base.html' %}

{% block content %}

<div class="column">
    {% for followed in user.profile.follows.all %}
        {% for dweet in followed.user.dweets.all %}
            <div class="box">
                {{ dweet.body }}
                <span class="is-small has-text-grey-light">
                    ({{ dweet.created_at }} by {{ dweet.user.username }}
                </span>
            </div>
        {% endfor %}
    {% endfor %}
</div>

{% endblock content %}
```

## Submit Dweets Using Django Forms
1. Create a TextInput form
```python
# dwitter/forms.py

from django import forms
from .models import Dweet

class DweetForm(forms.ModelForm):
    body = forms.CharField(
        required=True, 
        widget=forms.widgets.Textarea(
            attrs={
                "placeholder": "Dweet something...",
                "class": "textarea is-success is-medium",
            }
        ),
        label="",
        )

    class Meta:
        model = Dweet
        exclude = ("user", )
```

2. Render the form in the template
```python
# dwitter/views.py

from django.shortcuts import render
from .forms import DweetForm
from .models import Profile

def dashboard(request):
    if request.method == "POST":
        form = DweetForm(request.POST or None)
        if form.is_valid():
            dweet = form.save(commit=False)
            dweet.user = request.user
            dweet.save()
            return redirect("dwitter:dashboard")
    return render(request, "dwitter/dashboard.html", {"form": form})
```

3. Add template codes
```html 
<!-- dwitter/dashboard.html -->
<div class="column is-one-third">
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button class="button is-success is-fullwidth is-medium mt-5"
                type="submit">Dweet
        </button>
    </form>
</div>
```

## Improve the Front-End User Experience
1. Add this to the `dashboard.html` file
```html
<div class="block">
    <a href="{% url 'dwitter:profile_list' %} ">
        <button class="button is-dark is-outlined is-fullwidth">
            All Profiles
        </button>
    </a>
</div>
<div class="block">
    <a href="{% url 'dwitter:profile' request.user.profile.id %} ">
        <button class="button is-success is-light is-outlined is-fullwidth">
            My Profile
        </button>
    </a>
</div>
```









