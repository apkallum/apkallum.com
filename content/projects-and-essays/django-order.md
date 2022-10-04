+++
title = "Ordering Objects with Respect to a Foreign Key in Django: Challenges and Solutions"
+++
## 

Originally written for [JWP Consulting](jwpconsulting.net).

A common requirement in Django (and other) applications is to retrieve objects in a certain order. This order can be global, where all objects of a given model are ordered with respect to a certain attribute, for example their modification date. But what if we would like to retrieve objects with respect to a certain foreign key?

In Projectify, we want our tasks to retain their ordering with respect to the section they belong to, such that the order of tasks in a To-Do or Done section is guaranteed to be consistent. We also want to be able to insert a task into another section at a specified order.

We have tried two different approaches, and settled on using Django's meta option [order_with_respect_to](https://docs.djangoproject.com/en/3.2/ref/models/options/#order-with-respect-to). We combined `order_with_respect_to` with additional guarantees to satisfy our data integrity requirements.

### First Approach: django-ordered-model

We first reached for the package [django-ordered-model](https://pypi.org/project/django-ordered-model/) to achieve our aims. It comes with a clean interface and excellent Django admin integration. The API provided by the package offers simple methods to change the order of an object, and all that is required is inheriting the methods from the OrderedModel. From their documentation:

{{< highlight python3 >}}
from django.db import models
from ordered_model.models import OrderedModel

class Item(OrderedModel):
    name = models.CharField(max_length=100)
{{< /highlight >}}

Objects created are ordered incrementally when created:

{{< highlight python3 >}}

foo = Item.objects.create(name="Foo")
bar = Item.objects.create(name="Bar")
{{< /highlight >}}

The code above allows you to arbitrarily move the objects:

{{< highlight python3 >}}
# move to arbitrary position n
foo.to(12)
bar.to(13)

# move an object up or down
foo.up()
foo.down()

# Move object to the top
foo.top()
{{< /highlight >}}

You get the idea. One caveat is that these methods are called by hooking up to the `update()` method, not the `save()` method. Therefore if you override the `save()` method as a substitute for Django signals, or to simply update fields, you will have to pass the fields to update and the new values, like so:

{{< highlight python3 >}}
foo.to(12, extra_update={'modified': now()})
{{< /highlight >}}

The primary issue we ran into the package is the transient duplication of the order field, that sometimes didn't resolve on its own. We wanted to have a guarantee at the database level that no two child objects can have the same order number. [We opened an issue](https://github.com/django-ordered-model/django-ordered-model/issues/260) detailing the situation, and the package maintainers replied that the code is structured in such a way that supports neither guarantees nor even standard database transactions.

### Second Approach: Using order_with_respect_to meta option from Django

We settled on using the native Django implementation `order_with_respect_to` option. We added our own guarantees on top of it. This meta option can be set on any model that has a parent-child relationship. From the Django docs:

{{< highlight python3 >}}
from django.db import models

class Question(models.Model):
    text = models.TextField()
    # ...

class Answer(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    # ...

    class Meta:
        order_with_respect_to = 'question'
{{< /highlight >}}


How do we get and manipulate the order of Answer objects? The `order_with_respect_to` option provides the parent (Question) with two methods, one to retrieve and the other to set the order:

```
>>> question = Question.objects.get(id=1)`
>>> question.get_answer_order()`
[1, 2, 3]
```

The numbers in the array `[1, 2, 3]` represent the primary keys of each Answer object. To reorder the Answer objects, we call the method `set_answer_order()`:

```python3
>>> question.set_answer_order([3, 1, 2])
```

The order is implicitly set on object creation. Additionally, when an object is deleted, the order list is automatically updated by Django. There are two additional methods that are available for the child, ordered objects. These methods are `get_next_in_order()` and `get_previous_in_order()`.

What we need to implement ourselves is a method to change the order of the objects. In our case, we needed a method to move an object to position ~n~ safely.

To safely change the order of the objects, we used two features from the Django ORM. The first is standard (database transactions)[https://docs.djangoproject.com/en/4.0/topics/db/transactions/#django.db.transaction.atomic], which ensures that either the whole operation in the context block is successfully committed to the database, or none of the operations in the context block are committed.

The second feature we used is [select_for_update](https://docs.djangoproject.com/en/4.0/ref/models/querysets/#django.db.models.query.QuerySet.select_for_update). The purpose of this method is to lock certain rows in the database for the duration of the transaction. This is necessary since changing the order of one object necessitates changing the order of all related items.

Here is the implementation from our codebase:
{{< highlight python3 >}}
    def move_to(self, order):
        """
        Move to specified order n within workspace board.
        No save required.
        """
        neighbor_sections = (
            self.workspace_board.workspaceboardsection_set.select_for_update()
        )
        with transaction.atomic():
            # Force queryset to be evaluated to lock them for the time of
            # this transaction
            len(neighbor_sections)
            current_workspace = self.workspace_board
            # Django docs wrong, need to cast to list
            order_list = list(
                current_workspace.get_workspaceboardsection_order()
            )
            # The list is ordered by pk, which is not uuid for us
            current_object_index = order_list.index(self.pk)
            # Mutate to perform move operation
            order_list.insert(order, order_list.pop(current_object_index))
            # Set new order
            current_workspace.set_workspaceboardsection_order(order_list)
            current_workspace.save()
{{< /highlight >}}


The code builds on the aforementioned features of database transactions and row locking. We use standard list manipulation tools in Python, but you can come up with the new order in any manner you prefer. However, since the orders are relative to a foreign key, we don’t expect the order list to be very large.

In conclusion, we recommend using native Django ORM features to implement ordering of objects with respect to a foreign key. Future work on this could include the implementation of Django admin integration, or factoring out the ordering code as a mixin or Abstract Model.

### Note: Migrating from django-ordered-model to our own implementation

At Projectify, we care deeply about the data of our users. Despite our product being in Alpha stage, we ensure that all our database operations do not compromise data integrity.

Since we had already been using django-ordered-model, and since migrations are by default executed in a transaction, we cannot simply change the order field values in the same operation where we are modifying the schema.

The schema is stored in the database. Below is a photo of the schema table:

![django-order-db.png](/img/django-order.png)

We found it useful to have a database dump restored to a certain state so we can test our migration code. We use the following command on our Postgresql database:
```
pg_restore --clean --dbname {db_name} --host localhost --port 5432 --username {db_user} --no-owner database_backup
```