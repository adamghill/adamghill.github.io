---
layout: post
title: "Optimize the Django ORM"
date:   2018-12-01 22:33:16 -0400
categories: django python
---
One of Django's main benefits is the built-in models and ORM (object-relational mapper). It provides a quick to use, common interface for data operations for your models and can handle most queries pretty easily. It can also do some tricky SQL once you understand the syntax.

However, one drawback is because the SQL calls _are_ abstracted behind a simple API, it's easy to end up making more SQL calls than you realize.

# Show me the sql

If your code is only in a management command, then determining the exact sql queries is a little tricky. You can retrieve a close approximation with the `query` attribute on a QuerySet, but heed the warning about it being an "[opaque representation](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet)". You can also add `django.db.logging` to the loggers to see SQL get printed out to the console.

```python
"loggers": {
    "django.db.backends": {
        "level": "DEBUG",
        "handlers': ["console", ],
    }
```

# The one Toolbar to rule them all

If your code is called from a view, the easiest way to start deciphering what SQL is generated is installing [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/en/latest/). DDT provides an unbelievably helpful diagnostic tool which shows all of the SQL queries being run, how many are similar to each other and how many are duplicated. You can also look at the query plan for each SQL query and dig into why it might be slow.

# Select and prefectch _all_ the relateds

One thing to realize is that Django's ORM is pretty lazy by default. It will not run queries until the result has been asked for (either in code or directly in a view). It also won't join models by their ForeignKeys until needed. Those are beneficial optimizations, however they can bite you if you don't realize. For example:

```python
# models.py
class Author(models.Model):
	name = models.CharField(max_length=50)


class Blog(models.Model):
	author = models.ForeignKey(Author, related_name="blogs", on_delete=models.PROTECT)
	text = models.TextField()
    url = models.URLField()
```

```python
# views.py
def index(request):
	blogs = Blog.objects.all()
	
	return render(request, { "blogs": blogs })
```

```django
<!-- index.html -->{% raw %}
{% for blog in blogs %}
Author: {{ blog.author.name }}
{% endfor %}{% endraw %}
```

In the code above, each blog in the `for loop` in `index.html` will call the database again for the author's name. So, there would be 1 database call for all of the blogs, and then an additional database call for each blog to get the author's name.

The way to prevent the extra database calls is to use [`select_related`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#select-related) to force Django to join to the other model once and prevent subsequent calls if that relation is used.

Updating the view code to use a `select_related` would reduce the total sql calls to only 1 for the same Django template.

```python
# views.py
def index(request):
	blogs = Blog.objects.select_related("author").all()
	
	return render(request, { "blogs": blogs })
```

In some cases `select_related` won't work, but `prefetch_related` will. The Django documentation has [lots more details](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#prefetch-related) about when to use `prefetch_related`.

# Beware the instantiating of models

When the Django ORM creates a [QuerySet]() it takes the data retrieved from the database and populates the models. However, if you don't need a model, there are a few ways to skip constructing them unnecessarily.

`values_list` will return a list of tuples for all of the columns specified. Particularly useful is the `flat=True` keyword argument which returns a regular list if only one field is specified.

```python
# get a list of blog ids to use later
blog_ids = Blog.objects.all().values_list("id", flat=True)
```

Another pattern that I have done in the past is to create a dictionary with pair of data that is required. For example, if I was going to need blog ids and their urls:

```python
# get a dictionary of blog id->url
blog_ids_to_urls = {b.get("id"): b.get("url") for b in Blog.objects.all().values("id", "url")}
```

Then, if I need all of the ids `blog_ids_to_urls.keys()` or if I need the list of urls `blog_ids_to_urls.values()`.

Somewhat related, [`bidict`](https://bidict.readthedocs.io/en/master/) is a easy library to use if you need an easy way to access a dictionary's key and value at different places, as opposed to keeping around 2 dictionaries.

# Filtering on ids makes the world go 'round

`filter` on a Django model translates to a `WHERE` clause in SQL, and searching for an integer will [almost always be faster than searching on a string](https://stackoverflow.com/questions/2346920/sql-select-speed-int-vs-varchar) in Postgres. So, `Blog.objects.filter(id__in=blog_ids)` will be slightly more performant than `Blog.objects.filter(url__in=blog_urls)`.

# Only and defer to your hearts content

`Only` and `defer` are mirror opposite methods to acheive the same goal of only retrieving particular fields for your model. [`Only`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet.only) works by SELECTing the specified database fields, but not filling in any non-specified fields. [`Defer`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet.defer) works the opposite way, so the fields you specify aren't included in the SELECT statement in SQL.

However, this note in the Django documentation is telling:

>They provide an optimization for when you have analyzed your queries closely and understand *exactly* what information you need and have measured that the difference

# Annotate and carry on

For some code, I was getting a count for each model in a list in a loop.

```python
for author in Author.objects.all():
    blog_count = author.blogs.count()
    print(f"{blog_count} blogs by {author.name}")
```

This will create one SQL `SELECT` statement for every author. Instead, using an annotation will create 1 SQL `SELECT` with a `COUNT` and `GROUP BY`.

```python
author_counts = (
    Author.objects
    .annotate(blog_count=Count("blog__id"))
    .values("author__id", "blog_count")
)
```

[`Aggregation`](https://docs.djangoproject.com/en/2.1/topics/db/aggregation/) is the simpler version of [`annotation`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet.annotate) if you want calculate a value for all objects ina list (e.g. get the maximum id from a list of models). [`Annotation`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet.annotate) is useful if you want to calculate values over each model in a list and get the output.

# Bulk _smash_! Errr, create

Creating multiple objects with one query is possible with [`bulk_create`](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#bulk-create). There are some caveats to using `bulk_create`, and unfortunately you don't get a list of ids created after the insert which would be pretty useful.

# We want to bulk _you_ up

In Django `update` is a method on `QuerySets`, so you are able to filter a set of objects and update a column to a particular value. However, if you want to update a bunch of models with different values for the columns [`django-bulk-update`](https://github.com/aykut/django-bulk-update) is package that lets you create one SQL statement for a set of model updates with differing values.

# Gonna make you sweat (Everybody Raw Sql now)

If you really can't figure out a way to get the Django ORM to generate performant SQL, [`raw sql`](https://docs.djangoproject.com/en/2.1/ref/models/expressions/#django.db.models.expressions.RawSQL) is always available, although it's not generally advised to do it unless you have to.

# Putting on the ritz

The Django documentation is generally really helpful and will give you more in-depth details about each technique above. If you know of any other approaches to squeezing the most performance out of Django, I would love to hear about them on [@adamghill](https://twitter.com/adamghill).