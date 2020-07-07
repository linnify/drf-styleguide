
Django Rest Framework Styleguide used in Linnify projects.

This document is maintained by the Linnify community.

**Table of contents:**



1.  Overview
2.  Python Virtual Environment
3.  Multiple Environments - and settings.py (variable de env)
4.  Linting
5.  Models
6.  Managers 
7.  Custom Raw Queries
8.  Serializers
9.  Views
10. URLs
11. Exception Handling
12. Testing
13. Migration
14. Fixtures and load data
15. Translations
16. Enums and Types


### **Overview**


We at Linnify, understand that writing quality code is a craftsmanship that requires patience along with multiple try/failure blocks. This style guide represents the patterns and rules that we follow on our projects in order to keep everything organized, clean and fast. In this document you might find out new stuff, old stuff, or stuff that you don’t agree with, and that is ok. We don't say that this document is perfect, we just find it effective when maintaining multiple projects.


### **Python Virtual Environment**

We recommend starting every project with `python manage.py startproject projectname` and every app with `python manage.py startapp appname`.

In order to manage multiple environments we use `pipenv`. 

**Multiple Environments**

Instead of hardcoding strings in settings.py please make use of the environment variables and by any means do not store passwords or keys in plain text in settings.py.  A use-case would be to store the host and the port of the application in order to be deployed on multiple services.


```
   API_HOST = os.environ.get("API_HOST")
   API_PORT = os.environ.get("API_PORT")
```


**Linting**

We recommend performing linting checks prior to every push and formatting code prior to every commit. 

This is accomplished by running the following scripts: **pre-commit** and **pre-push.** The scripts are placed in the project root, under a directory called git-hooks.

In the **pre-commit** file run **black** for python applications.

In the **pre-push** file run **flake8** for python applications.


### **Models**

Let's take a look of an example model:


```
class MeasurementValue(models.Model):
    DEFAULT_PRIORITY = 1
    id = models.UUIDField(primary_key=True, default=uuid4, editable=False)
    measurement = models.ForeignKey(
        Measurement, on_delete=models.CASCADE, null=True, related_name="values"
    )
    measured = models.DateTimeField(
        editable=True, auto_now_add=False, default=timezone.now
    )
    validation_end_time = models.DateTimeField(null=True, blank=True)
    priority = models.PositiveIntegerField(default=DEFAULT_PRIORITY)
    objects = MeasurementValueManager()

    class Meta:
        ordering = ["-measured"]

    def __str__(self):
        measurement_id = (
            "No measurement" if not self.measurement else self.measurement.id
        )
        return f"Value {self.amount} - {measurement_id}"


    def save(self, *args, **kwargs):
        ## Custom changes
        super().save(*args, **kwargs)


    @property
    def valid(self):
        if self.validation_end_time is None:
            return True

        return self.validation_end_time >= timezone.now()


    def complete(self):
        self.validation_end_time = timezone.now()
        self.save()
```

Few things to spot here:

**The order of the inner classes and standard methods should be as follows:**


*   All database fields
*   Custom manager attributes
*   class Meta
*   def `__str__()`
*   def `save()`
*   `property` attributes
*   Any custom methods

**Properties:**

*   The valid property is working directly on the model fields.
*   Use UUIDField for models with more than 2.147.483.647 (4bytes) records.


##### **Custom validation**

*   Do validation using [Django’s constraints](https://docs.djangoproject.com/en/2.2/ref/models/constraints/) or using the `validators` property of the field.
*   You can also use the clean method only on the non-relational model fields and call full_clean in `save`.
*   If the custom validation is more complex & spans relationships, do it in the serializer that creates the model.


##### **Methods**

*   Try to keep the `save` method as clean as possible
*   Use `action` methods that are called usually from the view in order to modify certain fields on the instance, like `complete()`.


### **Managers** 

There are two reasons you might want to customize a **Manager**:



1.  to add extra **Manager** methods like creating, filtering or triggering Signals 
2.  to modify the initial **QuerySet** the **Manager** returns

A customized Manager class should not contain information regarding:


1. Calls to external services
2. Serialization and other components from the rest_framework application

We recommend not having multiple managers on a model, as much as possible make use of inheritance between custom managers.

The following code examples could be used as a reference when adding extra methods to the Manager.

```
class MeasurementManager(models.Manager):
   def create_new_measurement(self, container, sample=None):
       if sample is None:
           sample = container.create_sample()
       return self.create(
           sample=sample,
       )


    def active_measurements(self):
      return self.filter(active=True)
```


###  **Custom Raw Queries**

Depending on the use-case one can encounter a situation when the custom query is mapped to a particular model and in that case, we recommend defining the query inside a custom manager class.


```
class OrderManager(models.Manager):

    def orders(self):
        return self.raw(
            "SELECT * from order_order"
        )
```


Sometimes even **Managers.raw()** isn’t quite enough: you might need to perform queries that don’t map cleanly to models, or directly execute **UPDATE**, **INSERT**, or **DELETE** queries.

In these cases, you can always access the database directly, routing around the model layer entirely. For this a new file called **queries.py** should be created inside the application where the query is run. In this file, a function shall be created for each custom query using a cursor. For retrieving the results in a array of dictionaries and not just raw results we use the following 2 functions:


```
def fetch_all(cursor):
    columns = [col[0] for col in cursor.description]
    return [dict(zip(columns, row)) for row in cursor.fetchall()]




def fetch_one(cursor):
    columns = [col[0] for col in cursor.description]
    row = cursor.fetchone()


    if not row:
        return None


    return dict(zip(columns, row))


"""Now that we know how we can map the query results to python dictionaries, we can see how a function looks like in the following example."""

def get_in_progress_tasks(limit):

    params = {
        "in_progress_status": Status.IN_PROGRESS.value,
        "limit": limit,
    }


    with connection.cursor() as cursor:
        cursor.execute(
            """
        SELECT * FROM workflow_task
        WHERE status = %(in_progress_status)s
        LIMIT %(limit)s
        """, 
            params,
        )
        return fetch_all(cursor)
```


The parameters in the query are put between **%( )s.** We avoid putting them inside the query using formatting to prevent **SQL INJECTION.** Bad example: WHERE user.id = {user}.

Serializers

The **ModelSerializer** should be used for **creating**/**updating** operations. In case the instance that needs to be **created/updated** has a relation with another table, expect the client to specify the identifier of that relation instead of specifying the entire object. If the entire object is required on the response make use of the to_representation in order to add more information. 


If it is required to create multiple instances of the same type define the **list_serializer_class** of the **ModelSerializer** by creating a **ListSerializer** in order to override **create** and **update**. Based on the circumstances the **bulk_create()** and **bulk_update()** could be used. This will highly improve the performance of the serializer.

``` 
class BulkCreateMeasurementsSerializer(serializers.ListSerializer):

    def update(self, instance, validated_data):
        pass

    def create(self, validated_data):
        return MeasurementValue.objects.bulk_create(
            [ MeasurementValue(**measurement_value) for measurement_value in validated_data ]
        )

class MeasurementValueBaseSerializer(serializers.ModelSerializer):

    class Meta:
        model = MeasurementValue
        fields = (
            "id",
            "amount",
            "metric",
            "measurement"
        )
        list_serializer_class = BulkCreateMeasurementsSerializer
```


For **reading** operations, we recommend using a ReadOnlySerializer which the fields specified as **read_only.** This will improve the speed of your serializer because the validation of the fields will not be performed.


```
class ReadOnlyMeasurementValueSerializer(serializers.Serializer):
    id = serializers.UUIDField(read_only=True)
    amount = serializers.FloatField(read_only=True)
    measurement = serializers.ReadOnlyField(source="measurement_id")
    metric = serializers.ReadOnlyField(source="metric_id")
```


**Important!** Do not keep business logic inside the serializers. The serializers should be used just in order to create, update, or retrieve information about records.

We recommend returning the same information from the serializers of the following actions: **create/update/partial_update/retrieve** and to have a different serializer for the list. In practice the list action does not require all the fields and relations for that model.

Use the validate method of the serializer when the validation depends on multiple fields. If the validation is made on only one field define the **validate_&lt;field_name>** for that field.

### **Views**

The views should serve only one purpose. Make use of the Mixin classes in case not all the actions are required on a route. 

When implementing pagination on an endpoint, always check with the team-lead first (not every endpoint requires pagination). 

When using the **@action** decorator it is recommended to specify the permissions using the **@permission_classes** decorator instead of overriding the **get_permissions** method of the viewset. The **get_permissions** method should be overridden only for the default actions (list, retrieve, update, delete, partial_update).

When it is required to create a child with a nested route, make use of the parent_id specified in the route and set it in the request.data, instead of letting the client specify it in the body.


```
class OrderItemsModelViewSet(ModelViewSet):
    serializer_class = OrderItemSerializer
    queryset = OrderItem.objects.all()
    permission_classes = (IsOwnerUser, IsAuthenticated)

    def get_queryset(self):
        return super().get_queryset().filter(order_id=self.kwargs.get("order_pk"))

    def create(self, request, *args, **kwargs):
        request.data["order"] = kwargs.get("order_pk")
        return super().create(request, *args, **kwargs)
```


When working with nested views always check if the user has permission to access the parent. 


```
class IsOwner(permissions.BasePermission):
    def has_permission(self, request, view):
        return Order.objects.filter(
            id=view.kwargs.get("order_pk"),
            user=request.user
        ).exists()

    def has_object_permission(self, request, view, obj):
        return obj.user == request.user
```


Make use of django_filters in order to implement out of the box filtering and override the FilterSet class for more complex filters. The FilterSet class should be placed in a file called filters.py in the same directory with the app for which was created. See the example below.


```
class MeasurementFilter(FilterSet):
    measurement_types = CharInFilter(
        field_name="measurement_type__name", lookup_expr="in"
    )

    class Meta:
        model = Measurement
        fields = ("measurement_types",)
```


  

For common functionalities, it is recommended to create a custom **Mixin** that would be inherited by the view in order to obtain the functionalities. 

See the above example where the mixin is responsible to add the history action to a viewset.

```
class HistoryMixin(object):
    history_queryset = None
    history_serializer = None

    @action(methods=["GET"], detail=False, url_path="history")
    def history(self, request):
        assert self.history_queryset is not None
        assert self.history_serializer is not None

        data = self.history_serializer(instance=self.history_queryset).data

        return Response(data=data)
```


### **URLS**

We are using the **DefaultRouter()** without the trailing slash and the **path()** function instead of the **url()**, according to Django documentation the **url()** function will be deprecated in the future release. The following implementation will also work for nested **Mixins** or **ModelViewSets** if the **include()** function is used with the **urls**.


```
from django.urls import include, path
from rest_framework.routers import DefaultRouter

from order import views

router = DefaultRouter(trailing_slash=False)
router.register("orders", views.OrdersViewSet, basename="orders")

orders_router = DefaultRouter(trailing_slash=False)
orders_router.register(
    r"items", views.OrderItemsModelViewSet, basename="items"
)

urlpatterns = [
    path(r"^", include(router.urls)),
    path(r"orders/<int:order_pk>/", include(orders_router.urls))
]
```


### **Exception Handling**

In case on an exception the backend will return in the payload of the response the following information:


```
 {
        "errors": [
            {
                "message": "Error message",
                "code": "code",
                "field": "field_name"
            },
            {
                "message": "Error message",
                "code": "code",
                "field": "field_name"
            },
            ...
        ]
    }
```


The following format was obtained by specifying a **custom exception handler** in the **rest_framework** object from **settings.py.** The handler is responsible for formatting the data and converting a **django.core.exception** into a **rest_framework.exception**.

The files related to the handler are accessible on this link.  (The link might not be provided yet).

### **Testing**

There is no fast way, only the right way. We believe in writing tests before starting the implementation. We write tests usually for **views**, **models**, **managers** and **serializers**. We aim to keep every test as isolated as possible, which sometimes breaks the **DRY** principle.

The tests directory for each app has the following structure: 

*   **app_name/tests/models** 
*   **app_name/tests/view**
*   **app_name/tests/serializers**, 
*   **app_name/tests/managers**. 

**Factory-boy** is used usually for mocking the database and all the factory instances of an application should be placed inside of a **factories.py** file from the **tests** directory. 


### **Migration**

Limit the number of new migrations to 1 migration per application in a Merge Request.

Make use of custom migrations in order to modify or not lose existing records. Check the example of a custom migration that is updating existing records in order to not have a null value on a field that was newly added.


```
def update_existing_materials(apps, schema):
    model_names = {
        "MixedMaterial": MaterialTypeEnum.MIXED_MATERIAL.value,
        "IntermediateMaterial": MaterialTypeEnum.INTERMEDIATE_MATERIAL.value,
        "SourceMaterial": MaterialTypeEnum.SOURCE_MATERIAL.value,
        "MaterialType": MaterialTypeEnum.MATERIAL_TYPE.value
    }

    for key in model_names.keys():
        Model = apps.get_model("materials", key)
        Model.objects.filter(subclass_type__isnull=True).update(subclass_type=model_names.get(key))

class Migration(migrations.Migration):
    dependencies = [
        ('materials', '0020_materialtype_type'),
    ]
    operations = [
        migrations.RunPython(update_existing_materials),
        migrations.AlterField(
            model_name='materialtype',
            name='subclass_type',
            field=models.CharField(choices=[('material-type', 'material-type'),  ('source-material', 'source-material'), ('mixed-material', 'mixed-material'), ('intermediate-material', 'intermediate-material')], max_length=255),
        ),
    ]
```


### **Fixtures**

For us every application has a fixture directory with two files: **initial_data.yaml** and **test_data.yaml.**

**initial_data.yaml** would store sensitive records that are required to populate an empty database. 

**test_data.yaml** would store data that is required to test features of that particular application.

### **Translations**

It is always better to assume that the application would be used in different languages, in that case, every hardcoded string from the backend should be translatable. Django provides out of the box translation for hardcoded strings on the backend by using the **gettext()** function. Always import **gettext** as **_**. 

It might be required in some cases for the actual fields of the models to be translated. For that situation, we recommend using **django-modeltranslation**. This library will add a new column on the table for every language requested. E.g. The required languages are German(de) and English(en) and the **name** field needs to be translated. The result would be a table with the following attributes **(name, name_en, name_de).**

### **Enums and Types**

The recommendation is to place all the enums of an application inside a directory called types. Besides enums, custom data types could be defined here. Based on the use-case the data type could be grouped with other data types or could be placed in a single file. 

The following example would be a proper use-case for an enum in favor of a choices attribute defined  on the Process model.

```
from enum import Enum
from django.utils.translation import ugettext_lazy as _

class ProcessStateEnum(Enum):
    COMPLETED = "completed"
    SCHEDULED = "scheduled"
    ABORTED = "aborted"
    IN_PROGRESS = "in_progress"

    @classmethod
    def choices(cls):
        return (
            (str(cls.COMPLETED), _('Completed')),
            (str(cls.SCHEDULED), _('Scheduled')),
            (str(cls.ABORTED), _('Aborted')),
            (str(cls.IN_PROGRESS), _('In Progress')),
        )

In models.py

from django.db.models import Model
from types.enums import ProcessStateEnum

class Process(Model):
    state = models.CharField(
        choices=ProcessStateEnum.choices(),
        default=ProcessStateEnum.SCHEDULED,
        max_length=255,
    )
