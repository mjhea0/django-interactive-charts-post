# Django Interactive Charts

what we are building, code flow

## What is chart.js

[Chart.js](https://www.chartjs.org/) is a free open-source JavaScript library for data visualization. It supports eight different chart types: bar, line, area, pie, bubble, radar, polar, and scatter. It is flexible, highly customizable, has built-in animations and is easy to use.

> You can swap [chart.js]() for any other JavaScript chart library.

## Project Setup

xyz

Start by setting up a new Django project:

```sh
$ mkdir django-with-pydantic && cd django-with-pydantic
$ python3.9 -m venv env
$ source env/bin/activate

(env)$ pip install django==3.1.5
(env)$ django-admin.py startproject core .
```

After that, create a new app called `blog`:

```sh
(env)$ python manage.py startapp blog
```

Register the app in *core/settings.py* under `INSTALLED_APPS`:

```python
# core/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog.apps.BlogConfig', # new
]
```

### Create Database Models

Next, let's create the `Item` and the `Purchase` model.

```python
# shop/models.py

from django.db import models


class Item(models.Model):
    name = models.CharField(max_length=255)
    description = models.TextField(null=True)
    price = models.FloatField(default=0)

    def __str__(self):
        return f'{self.name} (${self.price})'


class Purchase(models.Model):
    customer_full_name = models.CharField(max_length=64)
    item = models.ForeignKey(to=Item, on_delete=models.CASCADE)
    PAYMENT_METHODS = [
        ('CC', 'Credit card'),
        ('DC', 'Debit card'),
        ('ET', 'Ethereum'),
        ('BC', 'Bitcoin'),
    ]
    payment_method = models.CharField(max_length=2, default='CC', choices=PAYMENT_METHODS)
    time = models.DateTimeField(auto_now_add=True)
    successful = models.BooleanField(default=False)

    class Meta:
        ordering = ['-time']

    def __str__(self):
        return f'{self.customer_full_name}, {self.payment_method} ({self.item.name})'
```

Create then apply the migrations:

```python 
(env)$ python manage.py makemigrations
(env)$ python manage.py migrate
```

Register the model in *blog/admin.py*:

```python
# shop/admin.py

from django.contrib import admin

from shop.models import Item, Purchase


admin.site.register(Item)
admin.site.register(Purchase)
```

### Populate the Database

for this step i created a django command which allows us to populate the database and get some sample data we can work with

```ssh
(venv)$ python manage.py populate_db --amount 2500
```

## Prepare and serve the data

describe each endpoint, django ORM, prepairing data for chart.js

## Create charts using chart.js

add chart.js, initialize chart, ajax request for fetching data

## Add charts to Django admin

we can use two different approaches:

1. creating a new admin view
1. overriding an admin django template

### Adding new admin views

### Overriding Django admin templates

## Conclusion
