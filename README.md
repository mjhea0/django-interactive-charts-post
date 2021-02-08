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

Now let's populate our database so we 

Create the folder management and within the folder create another folder called commands. Then put the following inside *populate_db.py*:

```python
# shop/management/commands/populate_db.py

  
import random
from datetime import datetime, timedelta

import pytz
from django.core.management.base import BaseCommand

from shop.models import Item, Purchase


class Command(BaseCommand):
    help = 'Populates the database with random generated data.'

    def add_arguments(self, parser):
        parser.add_argument('--amount', type=int, help='The number of purchases that should be created.')

    def handle(self, *args, **options):
        names = ['James', 'John', 'Robert', 'Michael', 'William', 'David', 'Richard', 'Joseph', 'Thomas', 'Charles']
        surname = ['Smith', 'Jones', 'Taylor', 'Brown', 'Williams', 'Wilson', 'Johnson', 'Davies', 'Patel', 'Wright']
        items = [
            Item.objects.get_or_create(name='Socks', price=6.5), Item.objects.get_or_create(name='Pants', price=12),
            Item.objects.get_or_create(name='T-Shirt', price=8), Item.objects.get_or_create(name='Boots', price=9),
            Item.objects.get_or_create(name='Sweater', price=3), Item.objects.get_or_create(name='Underwear', price=9),
            Item.objects.get_or_create(name='Leggings', price=7), Item.objects.get_or_create(name='Cap', price=5),
        ]
        amount = options['amount'] if options['amount'] else 2500
        for i in range(0, amount):
            dt = pytz.utc.localize(datetime.now() - timedelta(days=random.randint(0, 1825)))
            purchase = Purchase.objects.create(
                customer_full_name=random.choice(names) + ' ' + random.choice(surname),
                item=random.choice(items)[0],
                payment_method=random.choice(Purchase.PAYMENT_METHODS)[0],
                successful=True if random.randint(1, 2) == 1 else False,
            )
            purchase.time = dt
            purchase.save()

        self.stdout.write(self.style.SUCCESS('Successfully populated the database.'))
```

Run the following command to populate the DB:

```ssh
(venv)$ python manage.py populate_db --amount 2500
```

## Prepare and serve the data

Our app is going to have the following endpoints:

1. `chart/filter-options/` description
1. `chart/sales/<YEAR>/` description
1. `chart/spend-per-customer/<YEAR>/` description
1. `chart/payment-success/YEAR/` description
1. `chart/payment-method/YEAR/` description

Create the views:

```python
# shop/views.py

from django.contrib.admin.views.decorators import staff_member_required
from django.db.models import Count, F, Sum, Avg
from django.db.models.functions import ExtractYear, ExtractMonth
from django.http import JsonResponse

from shop.models import Purchase
from util.charts import months, colorPrimary, colorSuccess, colorDanger, generate_color_palette, get_year_dict


@staff_member_required
def get_filter_options(request):
    grouped_purchases = Purchase.objects.annotate(year=ExtractYear('time')).values('year').order_by('-year').distinct()
    options = [purchase['year'] for purchase in grouped_purchases]

    return JsonResponse({
        'options': options,
    })


@staff_member_required
def get_sales_chart(request, year):
    purchases = Purchase.objects.filter(time__year=year)
    grouped_purchases = purchases.annotate(price=F('item__price')).annotate(month=ExtractMonth('time'))\
        .values('month').annotate(average=Sum('item__price')).values('month', 'average').order_by('month')

    sales_dict = get_year_dict()

    for group in grouped_purchases:
        sales_dict[months[group['month']-1]] = round(group['average'], 2)

    return JsonResponse({
        'title': f'Sales in {year}',
        'data': {
            'labels': list(sales_dict.keys()),
            'datasets': [{
                'label': 'Amount ($)',
                'backgroundColor': colorPrimary,
                'borderColor': colorPrimary,
                'data': list(sales_dict.values()),
            }]
        },
    })


@staff_member_required
def spend_per_customer_chart(request, year):
    purchases = Purchase.objects.filter(time__year=year)
    grouped_purchases = purchases.annotate(price=F('item__price')).annotate(month=ExtractMonth('time'))\
        .values('month').annotate(average=Avg('item__price')).values('month', 'average').order_by('month')

    spend_per_customer_dict = get_year_dict()

    for group in grouped_purchases:
        spend_per_customer_dict[months[group['month']-1]] = round(group['average'], 2)

    return JsonResponse({
        'title': f'Spend per customer in {year}',
        'data': {
            'labels': list(spend_per_customer_dict.keys()),
            'datasets': [{
                'label': 'Amount ($)',
                'backgroundColor': colorPrimary,
                'borderColor': colorPrimary,
                'data': list(spend_per_customer_dict.values()),
            }]
        },
    })


@staff_member_required
def payment_success_chart(request, year):
    purchases = Purchase.objects.filter(time__year=year)

    return JsonResponse({
        'title': f'Payment success rate in {year}',
        'data': {
            'labels': ['Successful', 'Unsuccessful'],
            'datasets': [{
                'label': 'Amount ($)',
                'backgroundColor': [colorSuccess, colorDanger],
                'borderColor': [colorSuccess, colorDanger],
                'data': [
                    purchases.filter(successful=True).count(),
                    purchases.filter(successful=False).count(),
                ],
            }]
        },
    })


@staff_member_required
def payment_method_chart(request, year):
    purchases = Purchase.objects.filter(time__year=year)
    grouped_purchases = purchases.values('payment_method').annotate(count=Count('id'))\
        .values('payment_method', 'count').order_by('payment_method')

    payment_method_dict = dict()

    for payment_method in Purchase.PAYMENT_METHODS:
        payment_method_dict[payment_method[1]] = 0

    for group in grouped_purchases:
        payment_method_dict[dict(Purchase.PAYMENT_METHODS)[group['payment_method']]] = group['count']

    return JsonResponse({
        'title': f'Payment method rate in {year}',
        'data': {
            'labels': list(payment_method_dict.keys()),
            'datasets': [{
                'label': 'Amount ($)',
                'backgroundColor': generate_color_palette(len(payment_method_dict)),
                'borderColor': generate_color_palette(len(payment_method_dict)),
                'data': list(payment_method_dict.values()),
            }]
        },
    })
```

Hook up the urls:

```python
# shop/urls.py

from django.urls import path

from . import views

urlpatterns = [
    path('chart/filter-options/', views.get_filter_options, name='chart-filter-options'),
    path('chart/sales/<int:year>/', views.get_sales_chart, name='chart-sales'),
    path('chart/spend-per-customer/<int:year>/', views.spend_per_customer_chart, name='chart-spend-per-customer'),
    path('chart/payment-success/<int:year>/', views.payment_success_chart, name='chart-payment-success'),
    path('chart/payment-method/<int:year>/', views.payment_method_chart, name='chart-payment-method'),
]
```

## Create charts using chart.js

Add the following HTML:

```htmlembedded=
{% extends "admin/base_site.html" %}

{% block content %}
    <script src="https://cdn.jsdelivr.net/npm/chart.js@2.8.0"></script>
    <script src="https://code.jquery.com/jquery-3.5.1.min.js" crossorigin="anonymous"></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-v4-grid-only@1.0.0/dist/bootstrap-grid.min.css">
    <form id="filterForm">
        <label for="year">Choose a year:</label>
        <select name="year" id="year"></select>
        <input type="submit" value="Load" name="_load">
    </form>
    <script>
        $(document).ready(function() {
            $.ajax({
                url: "/shop/chart/filter-options/",
                type: "GET",
                dataType: "json",
                success: (jsonResponse) => {
                    // Load all the options
                    jsonResponse.options.forEach(option => {
                        $("#year").append(new Option(option, option));
                    });
                    // Load data for the first option
                    loadAllCharts($("#year").children().first().val());
                },
                error: () => console.log("Failed to fetch chart filter options!")
            });
        });

        $("#filterForm").on('submit', (event) => {
            event.preventDefault();

            const year = $("#year").val();
            loadAllCharts(year)
        });

        function loadChart(chart, endpoint) {
            $.ajax({
                url: endpoint,
                type: "GET",
                dataType: "json",
                success: (jsonResponse) => {
                    // Extract data from the response
                    const title = jsonResponse.title;
                    const labels = jsonResponse.data.labels;
                    const datasets = jsonResponse.data.datasets;

                    // Reset the current chart
                    chart.data.datasets = [];
                    chart.data.labels = [];

                    // Load new data into the chart
                    chart.options.title.text = title;
                    chart.data.labels = labels;
                    datasets.forEach(dataset => {
                        chart.data.datasets.push(dataset);
                    });
                    chart.update();
                },
                error: () => console.log("Failed to fetch chart data from " + endpoint + "!")
            });
        }

        function loadAllCharts(year) {
            loadChart(salesChart, `/shop/chart/sales/${year}/`);
            loadChart(spendPerCustomerChart, `/shop/chart/spend-per-customer/${year}/`);
            loadChart(paymentSuccessChart, `/shop/chart/payment-success/${year}/`);
            loadChart(paymentMethodChart, `/shop/chart/payment-method/${year}/`);
        }
    </script>
    <div class="row">
        <div class="col-6">
            <canvas id="salesChart"></canvas>
        </div>
        <div class="col-6">
            <canvas id="paymentSuccessChart"></canvas>
        </div>
        <div class="col-6">
            <canvas id="spendPerCustomerChart"></canvas>
        </div>
        <div class="col-6">
            <canvas id="paymentMethodChart"></canvas>
        </div>
    </div>
    <script>
        let salesCtx = document.getElementById('salesChart').getContext('2d');
        let salesChart = new Chart(salesCtx, {
            type: 'bar',
            options: {
                responsive: true,
                title: {
                    text: "",
                    display: true
                }
            }
        });
        let spendPerCustomerCtx = document.getElementById('spendPerCustomerChart').getContext('2d');
        let spendPerCustomerChart = new Chart(spendPerCustomerCtx, {
            type: 'line',
            options: {
                responsive: true,
                title: {
                    text: 'Spend per customer',
                    display: true
                }
            }
        });
        let paymentSuccessCtx = document.getElementById('paymentSuccessChart').getContext('2d');
        let paymentSuccessChart = new Chart(paymentSuccessCtx, {
            type: 'pie',
            options: {
                responsive: true,
                title: {
                    text: 'Spend per customer',
                    display: true
                },
                layout: {
                    padding: {
                        left: 0,
                        right: 0,
                        top: 0,
                        bottom: 25
                    }
                }
            }
        });
        let paymentMethodCtx = document.getElementById('paymentMethodChart').getContext('2d');
        let paymentMethodChart = new Chart(paymentMethodCtx, {
            type: 'pie',
            options: {
                responsive: true,
                title: {
                    text: 'Payment method',
                    display: true
                },
                layout: {
                    padding: {
                        left: 0,
                        right: 0,
                        top: 0,
                        bottom: 25
                    }
                }
            }
        });
    </script>
{% endblock %}
```

1. Explanation 1
1. Explanation 2
1. Explanation 3

## Add charts to Django admin

we can use two different approaches:

1. creating a new admin view
1. overriding an admin django template

### Adding new admin views

### Overriding Django admin templates

## Conclusion
