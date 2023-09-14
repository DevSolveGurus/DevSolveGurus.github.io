---
title : How to craft a bespoke filter with graphene-django and Relay harnessing multiple fields of diverse data types 
date : 2023-09-12 01:35:53 +0000
published: true  
---
# How to craft a bespoke filter with graphene-django and Relay, harnessing multiple fields of diverse data types

I'm currently working with Graphene Django and Graphene Relay in a Python 3.7 environment. My objective is to create a specialized GraphQL filter for handling requests, leveraging the capabilities of Django filters.

Here's an overview of my table structure:

```
| id(type: int) | flow(type: varchar) | datetime(type: datetime) | duration(type: int) |
|---------------|---------------------|--------------------------|---------------------|
| 1             | aaa                 | 2019-07-06 08:59:00      | 113095465           |
| 2             | xxx                 | 2019-07-06 08:59:00      | 113095465           |
| 3             | bbb                 | 2019-07-06 08:59:00      | 113095465           |

```
I aim to execute SQL queries of this nature via GraphQL:




```
SELECT * FROM tablename WHERE datetime <= "2019-07-06 09:00:00" AND DATE_ADD(datetime, INTERVAL duration / 1000 SECOND) >= "2019-07-06 08:59:00";
```
Using a GraphQL query like the one below (with the dates formatted in Python DateTime):

```json
{
  allTable(startDate: "2019-07-06 08:59:00", endDate: "2019-07-06 09:00:00") {
    edges {
      node {
        id
        name
      }
    }
  }
}```

This format is acceptable as long as it functions effectively:

```json
{
  allTable(timeRange: "{'start':'2019-07-06 08:59:00', 'end': '2019-07-06 09:00:00'}") {
    edges {
      node {
        id
        name
      }
    }
  }
}
```

the code
``` python 
#models.py

from django.db import models

class TableName(models.Model):
    name = models.CharField()
    datetime = models.DateTimeField()
    duration = models.BigIntegerField()
    class Meta:
        managed = False
        db_table = 'tablename'


```
```python
#schema.py
from graphene import relay, ObjectType
from graphene_django import DjangoObjectType
from graphene_django.filter import DjangoFilterConnectionField
import django_filters

from .models import TableName
from .fieldList import fields_table

class TableFilter(django_filters.FilterSet):
    class Meta:
        model = TableName
        fields = fields_table

class TableType(DjangoObjectType):
    class Meta:
        model = TableName
        filterset_class = TableFilter
        interfaces = (relay.Node,)

class Query(ObjectType):
    table = relay.Node.Field(TableType)
    all_Table = DjangoFilterConnectionField(TableType)

```
I've already succeeded in adding a custom filter to a field and accessing this filter within GraphQL:

```python
class TableFilter(django_filters.FilterSet):
    yolo = django_filters.NumberFilter(field_name="duration", lookup_expr='gt')
    ...

```
However, I'm uncertain about the procedure for implementing this particular custom filter.


I've reviewed the documentation for django_filters, Graphene, and Django querysets, but I'm still struggling to grasp the complete process, and I haven't come across an example that precisely matches my requirements. Thank you for your assistance! :)

## The Answer

To create a custom filter using Graphene Django and Relay that uses multiple fields of different types, you can follow these steps:

#### Step 1: Define a Custom Filter Class

In this step, you'll create a custom filter class that inherits from '**django_filters.FilterSet**'. This class will specify the fields you want to filter on and define custom filters for datetime and duration.

```python
# filters.py

import django_filters

class TableFilter(django_filters.FilterSet):
    # Define a custom filter for the datetime field
    datetime_range = django_filters.DateFromToRangeFilter(field_name="datetime")

    # Define a custom filter for the duration field
    duration_range = django_filters.NumberFilter(
        field_name="duration",
        method="filter_duration_range"
    )

    def filter_duration_range(self, queryset, name, value):
        if value:
            # Filter records where datetime is within the given range
            queryset = queryset.filter(
                datetime__lte=value[1],  # Upper bound of datetime range
                datetime__gte=value[0] - models.F("duration") / 1000  # Lower bound adjusted by duration
            )
        return queryset

    class Meta:
        model = TableName
        fields = []
```

#### Step 2: Update Your GraphQL Schema
Now, let's update your GraphQL schema to include the custom filters in the connection field.

```python
# schema.py

from graphene import relay, ObjectType
from graphene_django import DjangoObjectType
from graphene_django.filter import DjangoFilterConnectionField
import django_filters

from .models import TableName
from .filters import TableFilter  # Import the custom filter class

class TableType(DjangoObjectType):
    class Meta:
        model = TableName
        filterset_class = TableFilter  # Set the custom filter class

class Query(ObjectType):
    table = relay.Node.Field(TableType)
    
    # Update the all_table field to use the DjangoFilterConnectionField with custom filters
    all_table = DjangoFilterConnectionField(TableType, filterset_class=TableFilter)
```
#### Step 3: Use the Custom Filters in GraphQL Queries

Now you can use the custom filters in your GraphQL queries. Here's an example query that filters '**TableName**' records based on both datetime and duration:

```graphql
{
  allTable(
    datetimeRange: { after: "2019-07-06T08:59:00", before: "2019-07-06T09:00:00" }
    durationRange: 113095465
  ) {
    edges {
      node {
        id
        name
        datetime
        duration
      }
    }
  }
}

```

In this query, we use the '**datetimeRange**' filter to specify a datetime range and the '**durationRange**' filter to specify a duration value. The custom filter logic in the '**TableFilter**' class will handle the filtering based on your specified criteria.

This approach provides a more organized and maintainable way to implement custom filters in your GraphQL schema.











