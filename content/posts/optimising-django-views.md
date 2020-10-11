---
title: "Optimising Django views"
date: 2019-07-18T17:18:05+01:00
draft: false
---

In the early stages at Nuabee, we had little concern for optimising pages. Our main focus was to develop features and build up the functionality of Atlas. While this worked well in the beginning, some of our pages started to grow complex and page loading times were getting out of control.

To get an idea of just how bad it was, I downloaded [Django Silk](https://github.com/jazzband/django-silk) which is an open source profiler for Django. The cool thing about Django Silk is that it allows you to see each HTTP request, the time it took, and the amount of database queries that were made.

![silk](silk.png)

From there I made a spreadsheet to be able to rank the different pages and understand what needed to be optimised.

![excel](excel.png)

While some pages are quite complex, we can see that there are pages that have thousands and thousands of database requests. Obviously, this is inefficient and can be done better.

When searching how to optimise pages with Django, I came across 3 main ideas.

1. Using select_related and prefetch_related
2. Using cache
3. Loading data into memory.

## Using select_related and prefetch_related

One of the great things about Django is its ORM. With a simple command we can fetch all the items in the database and access any of their foreign keys. In this example we have a Machine model has a Site as a foreign key:

```python
machines = Machine.objects.all()
```

We can then iterate over all the machines and print the machine's site:

```python
for machine in machines:
	print(machine.site.name)
```

Even though we haven't done much, this is a terrible way to access an object's foreign keys. The problem here is that Django will query the database each time it has to obtain the machine's site. This also applies when using models in Django templates. If you try and access a models foreign key within a template, Django will query the database every single time.

The much better approach is to use `select_related`:

```python
machines = Machine.objects.select_related("site")
```

`select_related` is going to join the Machine and the Site tables so that there will only be one database query. We can access all the site's properties as Django has done a join at the time of getting the machines.

Now, imagine that our Machines have many ports, and for each machine we want to print each of its ports.

```python
machines = Machine.objects.all()
for machine in machines:
	for port in machine.ports:
		print(port)
```

We now encounter the same problem as before, Django will make a new database query each time we need to get a port. But this time, seeing as it's a many-to-many relationship we cannot use `select_related`, we must use `prefetch_related`

```python
machines = Machine.objects.prefetch_related("ports")
```

Now every time we access the machines ports, they will already be stored in memory and Django won't make a database query for each port.

## Using cache

While this is pretty simple, we can use cache to greatly improve the loading page speed.

Imagine that we have a page that is loaded frequently and there is a complex calculation that needs to be done. If you know that the data won't change for a certain period of time, we can put it into the cache.

To set cache we can use:

```python
data = complex_calculation()

# Stores data for 24hrs
cache.set("complex_data", 60 * 60 * 24)
```

To retrieve data:

```python
data = cache.get("complex_data")
```

Otherwise, we can combine the two:

```python
data = cache.get_or_set("complex_data", complex_calculation(), 60 * 60 * 24)
```

One of the considerations with caching is that at some point a user will be hit with the initial loading time. If this is unacceptable, you can run regular external tasks to load the data into cache. This way anytime a user tries to access the page they will only get cached responses rather than making the server do all the calculations.

## Loading models into memory

This last approach is the idea of loading data into memory rather than relying on database queries. It should only really be used as a last resort and you have to take into consideration how large the dataset is and how much it improves performance.

Imagine that you have a long list of items and for each of these items you need to call a function. Within this function, you need to gather information from the database. For example:

```python
long_list = [1,2,3,4,...]

****for item in long_list:
	complex_calculation(item)

def complex_calculation(item):
	data = Example.objects.filter(x_lt=item, x_gt=item)
	print(data)
```

The problem with this function is that Django will make requests for each item. Depending on the size of the list, this can dramatically slow down the page.

One of the ways to get around this problem is to load the Example model into memory and then use python to filter the list.

```python
long_list = [1,2,3,4,...]

examples = list(Example.objects.values("x"))

****for item in long_list:
	complex_calculation(examples, item)

def complex_calculation(examples, item):
	data = [e for e in examples if e["x"] > item ]
	print(data)
```

There is a few things going on within the code:

1. We get all the Example objects and only keep the x field.
2. Convert the QuerySet into a list so that Django makes the database query.
3. Pass the examples list into each function.
4. Use Python list comprehension to filter the list.

This will drastically reduce the amount of database requests and most likely improve loading times depending on the size of Example table.

## Conclusion

From using these three methods, we have managed to drastically reduce the loading times of some of our pages. If anyone has any other ways to improve loading times, I'd love to hear them.
