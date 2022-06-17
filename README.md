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




