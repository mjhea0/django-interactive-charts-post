# Adding Charts to the Django Admin with Chart.js

In this tutorial, we'll look at how to add interactive charts to the Django admin with [Chart.js](https://www.chartjs.org/). We'll use Django to model and prepare the data and then fetch it asynchronously from our template using AJAX.

## What is Chart.js?

[Chart.js](https://www.chartjs.org/) is a free, open-source JavaScript library for data visualization. It supports eight different chart types: bar, line, area, pie, bubble, radar, polar, and scatter. It's flexible and highly customizable. It supports animations. Add best of all, it's easy to use.

To get started, you just need to include Chart.js script along with a `<canvas>` node:

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@2.9.4"></script>

<canvas id="chart" width="500" height="500"></canvas>
```

Then, you can create a chart like so:

```javascript
let ctx = document.getElementById("chart").getContext("2d");

let chart = new Chart(ctx, {
  type: "bar",
  data: {
	 labels: ["2020/Q1", "2020/Q2", "2020/Q3", "2020/Q4"],
	 datasets: [
		{
		  label: "Gross volume ($)",
		  backgroundColor: "#79AEC8",
		  borderColor: "#417690",
		  data: [26900, 28700, 27300, 29200]
		}
	 ]
  },
  options: {
	 title: {
		text: "Gross Volume in 2020",
		display: true
	 }
  }
});
```

This code creates the following chart:

<p class="codepen" data-height="500" data-theme-id="0" data-default-tab="result" data-user="mjhea0" data-slug-hash="wvogmJy" style="height: 501px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="wvogmJy">
  <span>See the Pen <a href="https://codepen.io/mjhea0/pen/wvogmJy/">
  wvogmJy</a> by Michael Herman (<a href="https://codepen.io/mjhea0">@mjhea0</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

<p></p>

[CodePen link](https://codepen.io/mjhea0/pen/wvogmJy)

> To learn more about Chart.js check out the [official documentation](https://www.chartjs.org/docs/latest/).

With that, let's look at how to customize the Django admin with Chart.js.

## Project Setup

Let's create a simple shop application. We'll generate sample data using a Django command and then visualize it with Chart.js in the Django administration dashboard.

> You can swap [Chart.js](https://www.chartjs.org/) for any other JavaScript chart library like [D3.js](https://d3js.org/) or [morris.js](http://morrisjs.github.io/morris.js/). However, you will have to adjust the data format in your application's endpoints.

Workflow:

1. Fetch data using Django ORM queries
1. Format the data and return it via a protected endpoint
1. Request the data from a template using AJAX
1. Initialize Chart.js and load in the data

Start by creating a new directory and setting up a new Django project:

```sh
$ mkdir django-interactive-charts && cd django-interactive-charts
$ python3.9 -m venv env
$ source env/bin/activate

(env)$ pip install django==3.1.6
(env)$ django-admin.py startproject core .
```

After that, create a new app called `shop`:

```sh
(env)$ python manage.py startapp shop
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
    'shop.apps.ShopConfig', # new
]
```

### Create Database Models

Next, create `Item` and `Purchase` models:

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

1. `Item` represents an item in our store
1. `Purchase` represents a purchase (that's linked to an `Item`)

Make migrations and then apply them:

```sh
(env)$ python manage.py makemigrations
(env)$ python manage.py migrate
```

Register the models in *shop/admin.py*:

```python
# shop/admin.py

from django.contrib import admin

from shop.models import Item, Purchase


admin.site.register(Item)
admin.site.register(Purchase)
```

### Populate the Database

Before creating any charts, we need some data to work with. I've created a simple command we can use to populate the database.

Create a new folder in "shop" called "management", and then inside that folder create another folder called "commands". Inside of the "commands" folder create a new file called *populate_db.py*.

```sh
management
└── commands
    └── populate_db.py
```

*populate_db.py*:

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
(venv)$ python manage.py populate_db --amount 1000
```

If everything went well you should see a `Successfully populated the database.` message in the console.

> You can specify a custom amount; the amount represents the number of purchases.

## Prepare and Serve the Data

Our app will have the following endpoints:

1. `chart/filter-options/` lists all the years we have records for
1. `chart/sales/<YEAR>/` fetches monthly gross volume data by year
1. `chart/spend-per-customer/<YEAR>/` fetches monthly spend per customer by year
1. `chart/payment-success/YEAR/` fetches yearly payment success data
1. `chart/payment-method/YEAR/` fetches yearly payment method data

Next, let's create a few utility functions that will make it easier to create the charts. Add a new folder in the project root called "utils". Then, add a new file to that folder called *charts.py*:

```python
# util/charts.py

months = [
    'January', 'February', 'March', 'April',
    'May', 'June', 'July', 'August',
    'September', 'October', 'November', 'December'
]
colorPalette = ['#55efc4', '#81ecec', '#a29bfe', '#ffeaa7', '#fab1a0', '#ff7675', '#fd79a8']
colorPrimary, colorSuccess, colorDanger = '#79aec8', colorPalette[0], colorPalette[5]


def get_year_dict():
    year_dict = dict()

    for month in months:
        year_dict[month] = 0

    return year_dict


def generate_color_palette(amount):
    palette = []
    i = 0
    while i < len(colorPalette) and len(palette) < amount:
        palette.append(colorPalette[i])
        i += 1
        if i == len(colorPalette) and len(palette) < amount:
            i = 0
    return palette
```

So, we defined our chart colors and created the following two methods:

1. `get_year_dict()` creates a dictionary of months and values, which we'll use to add the monthly data to.
1. `generate_color_palette(amount)` generates a repeating color palette that we'll pass to our charts.

### Views

Create the views:

```python
# shop/views.py

from django.contrib.admin.views.decorators import staff_member_required
from django.db.models import Count, F, Sum, Avg
from django.db.models.functions import ExtractYear, ExtractMonth
from django.http import JsonResponse

from shop.models import Purchase
from utils.charts import months, colorPrimary, colorSuccess, colorDanger, generate_color_palette, get_year_dict


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

1. `get_filter_options(request)` fetches all the purchases, groups them by year, extracts the year from the time field and returns them in array.
1. `get_sales_chart(request, year)` fetches all the purchases (in a specific year) and its prices, groups them by month and calculates the monthly sum of prices.
1. `spend_per_customer_chart(request, year)` fetches all the purchases (in a specific year) and its prices, groups them by month and calculates the average monthly price.
1. `payment_success_chart(request, year)` counts successful and unsuccessful purchases in a specific year.
1. `payment_method_chart(request, year)` fetches all the purchases in a specific year, groups them by payment method, counts the purchases and returns them in a dictionary.

> Note that all the views have `@staff_member_required` decorator.

### Urls

Create the app-level URLs for the views:

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

Then, wire up the app URLs to the project URLs:

```python
# core/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('shop/', include('shop.urls')),  # new
]
```

### Testing

Now that we registered the urls, let's test the endpoints to see if everything works correctly.

Create a superuser, and then tun the development server:

```sh
(env)$ python manage.py createsuperuser
(env)$ python manage.py runserver
```

Then, in your browser of choice, navigate to [http://localhost:8000/shop/chart/filter-options/](http://localhost:8000/shop/chart/filter-options/). You should see something like:

```json
{
    "options":[
        2021,
        2020,
        2019,
        2018,
        2017,
        2016
    ]
}
```

To view the monthly sales data, first pick a year. For example, [http://localhost:8000/shop/chart/sales/2020/](http://localhost:8000/shop/chart/sales/2020/) should return sales data for 2020:

```json
{
  "title":"Sales in 2020",
  "data":{
    "labels":[
      "January",
      "February",
      "March",
      "April",
      "May",
      "June",
      "July",
      "August",
      "September",
      "October",
      "November",
      "December"
    ],
    "datasets":[
      {
        "label":"Amount ($)",
        "backgroundColor":"#79aec8",
        "borderColor":"#79aec8",
        "data":[
          137.0,
          88.0,
          187.5,
          156.0,
          70.5,
          133.0,
          142.0,
          176.0,
          155.5,
          104.0,
          125.5,
          97.0
        ]
      }
    ]
  }
}
```

## Create Charts with Chart.js

Add a "templates" folder to "shop". Then, add a new file called *statistics.html*:

```html
<!-- shop/templates/statistics.html -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Statistics</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@2.8.0"></script>
    <script src="https://code.jquery.com/jquery-3.5.1.min.js" crossorigin="anonymous"></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-v4-grid-only@1.0.0/dist/bootstrap-grid.min.css">
  </head>
  <body>
    <div class="container">
      <form id="filterForm">
        <label for="year">Choose a year:</label>
        <select name="year" id="year"></select>
        <input type="submit" value="Load" name="_load">
      </form>
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
          }
        });
        let spendPerCustomerCtx = document.getElementById('spendPerCustomerChart').getContext('2d');
        let spendPerCustomerChart = new Chart(spendPerCustomerCtx, {
          type: 'line',
          options: {
            responsive: true,
          }
        });
        let paymentSuccessCtx = document.getElementById('paymentSuccessChart').getContext('2d');
        let paymentSuccessChart = new Chart(paymentSuccessCtx, {
          type: 'pie',
          options: {
            responsive: true,
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
    </div>
  </body>
</html>
```

This block of code creates the HTML canvases that Chart.js uses to initialize. We also passed `responsive` to each charts' options so it adjusts based on the window size.

Add the following script to your HTML file:

```html
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
      chart.options.title.display = true;
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
```

When the page loads this script sends an AJAX request to `/chart/filter-options/` to fetch all the active years and loads them into the form.

1. `loadChart` loads chart data from the Django endpoint into the chart
2. `loadAllCharts` loads all the charts

> Note that we used jQuery's AJAX function. Feel free to use plain JavaScript requests.

Inside *shop/urls.py* create a new view:

```python
@staff_member_required
def statistics_view(request):
    return render(request, 'statistics.html', {})
```

Don't forget the import:

```python
from django.shortcuts import render
```

Assign a URL to the view:

```python
# shop/urls.py

from django.urls import path

from . import views

urlpatterns = [
    path('statistics/', views.statistics_view, name='shop-statistics'),  # new
    path('chart/filter-options/', views.get_filter_options, name='chart-filter-options'),
    path('chart/sales/<int:year>/', views.get_sales_chart, name='chart-sales'),
    path('chart/spend-per-customer/<int:year>/', views.spend_per_customer_chart, name='chart-spend-per-customer'),
    path('chart/payment-success/<int:year>/', views.payment_success_chart, name='chart-payment-success'),
    path('chart/payment-method/<int:year>/', views.payment_method_chart, name='chart-payment-method'),
]
```

Your charts are now accessible at: [http://localhost:8000/shop/statistics/](http://localhost:8000/shop/statistics/).

## Add Charts to the Django Admin

WIth regard to integrating charts into the Django admin, we can:

1. Create a new Django admin view
1. Override an existing admin template
1. Use a third-party package (i.e., [django-admin-tools](https://github.com/django-admin-tools/django-admin-tools))

### Create a new Django admin view

Creating a new Django admin view is the cleanest and the most straight forward approach. In this approach, we'll create a new [AdminSite](https://docs.djangoproject.com/en/3.1/ref/contrib/admin/#django.contrib.admin.AdminSite) and change it in the *settings.py* file.

First, within "shop/templates", add an "admin" folder. Add a *statistics.html* template to it:

```html
<!-- shop/templates/admin/statistics.html -->

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
    loadAllCharts(year);
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
        chart.options.title.display = true;
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
    }
  });
  let spendPerCustomerCtx = document.getElementById('spendPerCustomerChart').getContext('2d');
  let spendPerCustomerChart = new Chart(spendPerCustomerCtx, {
    type: 'line',
    options: {
      responsive: true,
    }
  });
  let paymentSuccessCtx = document.getElementById('paymentSuccessChart').getContext('2d');
  let paymentSuccessChart = new Chart(paymentSuccessCtx, {
    type: 'pie',
    options: {
      responsive: true,
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

*core/admin.py*:

```python
# core/admin.py

from django.contrib import admin
from django.contrib.admin.views.decorators import staff_member_required
from django.shortcuts import render
from django.urls import path


@staff_member_required
def admin_statistics_view(request):
    return render(request, 'admin/statistics.html', {
        'title': 'Statistics'
    })


class CustomAdminSite(admin.AdminSite):
    def get_app_list(self, request):
        app_list = super().get_app_list(request)
        app_list += [
            {
                "name": "My Custom App",
                "app_label": "my_custom_app",
                "models": [
                    {
                        "name": "Statistics",
                        "object_name": "statistics",
                        "admin_url": "/admin/statistics",
                        "view_only": True,
                    }
                ],
            }
        ]
        return app_list

    def get_urls(self):
        urls = super().get_urls()
        urls += [
            path('statistics/', admin_statistics_view, name='admin-statistics'),
        ]
        return urls
```

Here we created a staff protected view called `admin_statistics_view`. Then we created a new `AdminSite` and overrode `get_app_list` to add our own custom application to it. We provided our application with an artificial view-only model called `Statistics`. Lastly, we overrode `get_urls` and assigned an URL to our new view.

Create an `AdminConfig` inside *core/apps.py*:

```python
# core/apps.py

from django.contrib.admin.apps import AdminConfig


class CustomAdminConfig(AdminConfig):
    default_site = 'core.admin.CustomAdminSite'
```

Here we created a new `AdminConfig` which loads our `CustomAdminSite` instead of Django's default one.

Replace the default `AdminConfig` with the new one in *core/settings.py*:

```python
INSTALLED_APPS = [
    'core.apps.CustomAdminConfig',  # replaced
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'shop.apps.ShopConfig',
]
```

Navigate to [http://localhost:8010/admin/statistics/](http://localhost:8010/admin/statistics/).

The final result:

![Django admin new app](https://i.ibb.co/XyXxj3V/new-app.png)

Click on 'View' to see the charts:

![Chart preview](https://i.ibb.co/WysdG50/app-details.png)

### Override an existing admin template

You can always extend admin templates and override the parts you want. You can copy some parts from the admin template and change them the way you want.

For example, if you wanted to put charts under your shop models, you can do that by overriding *django-interactive-charts/shop/templates/admin/shop/app_index.html* and put the following inside: [app_index.html](https://gist.github.com/duplxey/5c9d17ccf2bdd2bc904d0546c044c2b1)

Final result:

![Django admin template overriding](https://i.ibb.co/qNHFLCX/shop-dashboard.png)

## Conclusion

In this article, we learned how to serve data with Django and then visualize it using Chart.js. We also looked at three different approaches for integrating charts into your Django administration.

Grab the code from the [django-interactive-charts](https://github.com/duplxey/django-interactive-charts) repo on GitHub.
