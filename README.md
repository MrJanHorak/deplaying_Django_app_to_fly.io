# Deploying a Django app to [fly.io](fly.io)

## Overview

This post has two main sections:

1. <b>Setting up</b>: steps on how to make your local Django app ready for going to production on any hosting platform.
2. <b>Deploying to [fly.io](fly.io)</b>: deploying your production-ready Django app to [fly.io](fly.io).

## Setting Up

You will need to create a virtual environment.

```shell
# Linux/Unix/macOS
python3 -m venv .venv
source .venv/bin/activate

#once activated you should see this prompt:
(.venv) $

# Windows
$ python -m venv .venv
$ .venv\Scripts\activate
#once activated you should see this prompt:
(.venv) $
```

The great thing is that Fly.io provides a single-node/high availability PostgreSQL cluster out of the box for us to use. It's also easy to set it up when configuring your deployment. We'll go over how in the next steps.

You can also use [neon.tech](neon.tech) to host your PostgreSQL database for free. 

The config updates required to change our database are explained in the next steps.

## Environment Variables

First of all, we want to store the configuration separate from our code and load them at runtime. This allow us to keep one settings.py file and still have multiple environments (i.e. local/staging/production).

One popular options is the usage of the environs package. Another option is the python-decouple package, which was originally designed for Django, but it's recommended to be used with dj-database-url to configure the database. For this guide we'll use django-environ to configure our Django application, which allows us to use the 12factor approach.

Make sure your Python virtual environment is activated and let's install the django-environ:

```shell
python -m pip install django-environ==0.9.0
```

In our settings.py file, we can define the casting and default values for specific variables, for example, setting DEBUG to False by default.

```python
# settings.py
from pathlib import Path
import environ  # <-- Updated!

env = environ.Env(  # <-- Updated!
    # set casting, default value
    DEBUG=(bool, False),
)
```

django-environ (and also the other mentioned packages) can take environment variables from the .env file. Go ahead and create this file in your root directory and also don't forget to add it to your .gitignore in case you are using Git for version control (you should!). We don't want those variables to be public, this will be used for the local environment and be set separately in our production environment.

Make sure you are taking the environment variables from your .env file:

```python
# settings.py
# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# Take environment variables from .env file
environ.Env.read_env(BASE_DIR / '.env')  # <-- Updated!
```
We can now set the specific environment variables in the .env file:

```.env
# .env
SECRET_KEY=justhaveyourcatdanceonyourkeyboard
DEBUG=True
```
Check that there are no quotations around strings neither spaces around the =.

Coming back to our settings.py, we can then read SECRET_KEY and DEBUG from our environment variables:

```python
# settings.py
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')  # <-- Updated!

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')  # <-- Updated!
```

Here we can define our local database, adding it to the .env file:

```.env
# .env
SECRET_KEY=3ohiu^m1su%906rf#mws)xt=1u#!xdj+l_ahdh0r#$(k_=e7lb
DEBUG=True
DATABASE_URL=postgres://postgres:postgres@localhost:5432/catcollector  # <-- Updated!
```

## Psycopg

To interact with our Postgres database, we'll use the most popular PostgreSQL database adapter for Python, the psycopg package. With your virtual environment activated, go ahead and installed it:

```shell
python -m pip install psycopg2-binary
```

⚠️ For production, [it's advised to use the source distribution](https://psycopg.org/docs/install.html?highlight=binary#psycopg-vs-psycopg-binary) instead of the binary package (psycopg2-binary). ```python -m pip install psycopg2==2.9.5``` however fly.o causes problems for some users and runs fine with the binary package.

## Gunicorn

When starting a Django project (with startproject management command), we get a minimal WSGI configuration set up out of the box. However, this default webserver is not recommended for production. Gunicorn (Green Unicorn) is a Python WSGI HTTP Server for Unix and one of the easiest to start with. It can be installed using pip:

```shell
python -m pip install gunicorn==20.1.0
```

## Static Files

Handling static files in production is a bit more complex than in development. One of the easiest and most popular ways to serve our static files in production is using the WhiteNoise package, which serves them directly from our WSGI Server (Gunicorn). Install it with:


```shell
python -m pip install whitenoise==6.3.0
```

A few changes to our settings.py are necessary.

* Add the WhiteNoise to the MIDDLEWARE list right after the SecurityMiddleware.
* Set the STATIC_ROOT to the directory where the collectstatic management command will collect the static files for deployment.
* (Optional) Set STATICFILES_STORAGE to CompressedManifestStaticFilesStorage to have compression and caching support.

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # <-- Updated!
    ...
]

...
STATIC_ROOT = BASE_DIR / 'staticfiles'  # <-- Updated!

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'  # <-- Updated!
```

It's recommended to use WhiteNoise also in development to keep consistent behavior between development and production environments. The easiest way to do that is to add whitenoise.runserver_nostatic to our INSTALLED_APPS right before the built-in staticfiles app:

```python
# settings.py
INSTALLED_APPS = [
    ...
    'whitenoise.runserver_nostatic',  # <-- Updated!
    'django.contrib.staticfiles',
    ...
]
```

## ALLOWED_HOSTS and CSRF_TRUSTED_ORIGINS

As a security measure, we should set in ALLOWED_HOSTS, a list of host/domain names that our Django website can serve. For development we might include localhost and 127.0.0.1 and for our production we can start with .fly.dev (or the provider's subdomain you chose) and update for the dedicated URL once your app is deployed to the hosting platform.

CSRF_TRUSTED_ORIGINS should also be defined with a list of origins to perform unsafe requests (e.g. POST). We can set the subdomain https://*.fly.dev (or the provider's subdomain you chose) until our deployment is done and we have the proper domain for our website.

```python
# settings.py
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.fly.dev']  # <-- Updated!

CSRF_TRUSTED_ORIGINS = ['https://*.fly.dev']  # <-- Updated!
```
## Installed packages

Make sure you have all necessary installed packages tracked and listed in your requirements.txt by running:

```shell
pip freeze > requirements.txt
```

This command generates the requirements.txt file if it doesn't exist. (It is the Django equivalent to the package.json in NodeJS)

```python
# requirements.txt
asgiref==3.7.2
boto3==1.26.158
botocore==1.29.158
Django==4.2.2
django-environ==0.9.0
gunicorn==20.1.0
jmespath==1.0.1
psycopg2-binary==2.9.6
python-dateutil==2.8.2
s3transfer==0.6.1
six==1.16.0
sqlparse==0.4.4
typing_extensions==4.6.3
urllib3==1.26.16
whitenoise==6.3.0
```

With our Django application prepped and ready for production hosting, we'll take the next step and deploy our app to Fly.io!

# Deploying to Fly.io 🚀

[flyctl](https://fly.io/docs/hands-on/install-flyctl/) is the command-line utility provided by Fly.io.

If not installed yet, follow these [instructions](https://fly.io/docs/hands-on/install-flyctl/), [sign up](https://fly.io/docs/hands-on/sign-up/) and [log in](https://fly.io/docs/hands-on/sign-in/) to Fly.io.

## Launching our App

Fly.io allows us to deploy our Django app as long as it's packaged in a Docker image. However, we don't need to define our <span style="color:lightblue">Dockerfile</span>. manually. Fly.io detects our Django app and automatically generates all the necessary files for our deployment. Those are:

Dockerfile contain commands to build our image.
.dockerignore list of files or directories Docker will ignore during the build process.
fly.toml configuration for deployment on Fly.io.
All of those files are templates for a simple Django apps and can be modified according to your needs.

Before deploying our app, first we need to configure and launch our app to Fly.io by using the flyctl command fly launch. During the process, we will:

Choose an app name: this will be your dedicated fly.dev subdomain.
Select the organization: you can create a new organization or deploy to your personal account (connect to your Fly account, visible only to you).
Choose the region for deployment: Fly.io initially suggests the closest to you, you can choose another region if you prefer.
Set up a Postgres database cluster: flyctl offers a single node "Development" config that is designed so we can turn it into a high-availability cluster by adding a second instance in the same region. Fly Postgres is a regular app you deploy on Fly.io, not a managed database.
This is what it looks like when we run fly launch:



pip install psycopg2-binary

python manage.py collectstatic



# Credits:
* [This blog post](https://fly.io/django-beats/deploying-django-to-production/)
* https://fly.io/docs/django/getting-started/
* https://fly.io/docs/reference/secrets/#setting-secrets 