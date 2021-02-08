# Django Interactive Charts

## Intro

what we are building, code flow

## What is chart.js

brief description of chart.js

> You can swap [chart.js]() for any other JavaScript chart library.

## Project Setup

start djangoproject, create app

### Create Database Models

Item, Purchase

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
