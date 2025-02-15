Serialization
=============

Serialization is the process of converting an object into a stream of
bytes to store the object in the memory, a file or database, or transmit
it through the network. Its main purpose is to save the state of an
object in order to be able to recreate it when needed. The reverse
process is called deserialization. Hazelcast offers you its own native
serialization methods. You will see these methods throughout this
chapter.

Hazelcast serializes all your objects before sending them to the server.
The ``bool``, ``int``, ``float``, ``str``, ``bytearray``, ``list``,
``datetime.date``, ``datetime.time``, ``datetime.datetime``, and
``decimal.Decimal`` types are serialized natively and you cannot override
this behavior. The following table is the conversion of types for the
Java server side.

================= ================================================
Python            Java
================= ================================================
bool              Boolean
int               Byte, Short, Integer, Long, java.math.BigInteger
float             Float, Double
str               String
bytearray         byte[]
list              java.util.ArrayList
datetime.date     java.time.LocalDate
datetime.time     java.time.LocalTime
datetime.datetime java.time.OffsetDateTime
decimal.Decimal   java.math.BigDecimal
================= ================================================


.. Note:: An ``int`` is serialized as ``Integer`` by
    default. You can configure this behavior using the
    ``default_int_type`` argument.

Arrays of the above types can be serialized as ``boolean[]``,
``byte[]``, ``short[]``, ``int[]``, ``float[]``, ``double[]``,
``long[]`` and ``string[]`` for the Java server side, respectively.

**Serialization Priority**

When Hazelcast Python client serializes an object:

1. It first checks whether the object is ``None``.

2. If the above check fails, then it checks if there is a
   ``CompactSerializer`` registered for the class of the object.

3. If the above check fails, then it checks if it is an instance of
   ``IdentifiedDataSerializable``.

4. If the above check fails, then it checks if it is an instance of
   ``Portable``.

5. If the above check fails, then it checks if it is an instance of one
   of the default types (see the default types above).

6. If the above check fails, then it looks for a user-specified
   :ref:`serialization:custom serialization`.

7. If the above check fails, it will use the registered
   :ref:`serialization:global serialization` if one exists.

8. If the above check fails, then the Python client uses ``pickle``
   by default.

However, ``cPickle/pickle Serialization`` is not the best way of
serialization in terms of performance and interoperability between the
clients in different languages. If you want the serialization to work
faster or you use the clients in different languages, Hazelcast offers
its own native serialization types, such as
:ref:`serialization:compact serialization`,
:ref:`serialization:identifieddataserializable serialization`, and
:ref:`serialization:portable serialization`.

On top of all, if you want to use your own serialization type, you can
use a :ref:`serialization:custom serialization`.

Compact Serialization
---------------------

.. warning::
    Compact Serialization feature is in the BETA status and any part of it
    might be changed without a prior notice, until it is promoted to the
    stable status.

As an enhancement to existing serialization methods, Hazelcast offers a beta
version of the compact serialization, with the following main features:

- Separates the schema from the data and stores it per type, not per object
  which results in less memory and bandwidth usage compared to other formats
- Does not require a class to extend another class or change the source code
  of the class in any way
- Supports schema evolution which permits adding or removing fields, or
  changing the types of fields
- Platform and language independent
- Supports partial deserialization of fields, without deserializing the whole
  objects during queries or indexing

Hazelcast achieves these features by having a well-known schema of objects and
replicating them across the cluster which enables members and clients to fetch
schemas they don’t have in their local registries. Each serialized object
carries just a schema identifier and relies on the schema distribution service
or configuration to match identifiers with the actual schema. Once the schemas
are fetched, they are cached locally on the members and clients so that the
next operations that use the schema do not incur extra costs.

Schemas help Hazelcast to identify the locations of the fields on the
serialized binary data. With this information, Hazelcast can deserialize
individual fields of the data, without reading the whole binary. This results
in a better query and indexing performance.

Schemas can evolve freely by adding or removing fields. Even, the types of the
fields can be changed. Multiple versions of the schema may live in the same
cluster and both the old and new readers may read the compatible parts of the
data. This feature is especially useful in rolling upgrade scenarios.

The Compact serialization does not require any changes in the user classes as
it doesn’t need a class to extend another class. Serializers might be
implemented and registered separately from the classes.

The underlying format of the compact serialized objects is platform and
language independent.

Using Compact Serialization
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Compact serialization can be used by writing a serializer that extends the
:class:`CompactSerializer <hazelcast.serialization.api.CompactSerializer>`
for a class and registering it in the client configuration.

For example, assume that you have the following ``Employee`` class:

.. code:: python

    class Employee:
        def __init__(self, name: str, age: int):
            self.name = name
            self.age = age


Then, a serializer for it can be implemented as below:

.. code:: python

    from hazelcast.serialization.api import CompactSerializer, CompactWriter, CompactReader

    class EmployeeSerializer(CompactSerializer[Employee]):
        def read(self, reader: CompactReader):
            name = reader.read_string("name")
            age = reader.read_int32("age")
            return Employee(name, age)

        def write(self, writer: CompactWriter, obj: Employee):
            writer.write_string("name", obj.name)
            writer.write_int32("age", obj.age)

        def get_type_name(self):
            return "employee"

        def get_class(self):
            return Employee

The last step is to register the serializer in the client configuration.

.. code:: python

    client = HazelcastClient(
        compact_serializers=[
            EmployeeSerializer(),
        ]
    )

A schema will be created from the serializer, and a unique schema identifier
will be assigned to it automatically.

From now on, Hazelcast will serialize instances of the ``Employee`` class
using the ``EmployeeSerializer``.

Schema Evolution
~~~~~~~~~~~~~~~~

Compact serialization permits schemas and classes to evolve by adding or
removing fields, or by changing the types of fields. More than one version of
a class may live in the same cluster and different clients or members might
use different versions of the class.

Hazelcast handles the versioning internally. So, you don’t have to change
anything in the classes or serializers apart from the added, removed, or
changed fields.

Hazelcast achieves this by identifying each version of the class by a unique
fingerprint. Any change in a class results in a different fingerprint.
Hazelcast uses a 64-bit
`Rabin Fingerprint <https://en.wikipedia.org/wiki/Rabin_fingerprint>`__ to
assign identifiers to schemas, which has an extremely low collision rate.

Different versions of the schema with different identifiers are replicated in
the cluster and can be fetched by clients or members internally. That allows
old readers to read fields of the classes they know when they try to read data
serialized by a new writer. Similarly, new readers might read fields of the
classes available in the data, when they try to read data serialized by an old
writer.

Assume that the two versions of the following ``Employee`` class lives in the
cluster.

.. code:: python

    class Employee:
        def __init__(self, name: str, age: int):
            self.name = name
            self.age = age

.. code:: python

    class Employee:
        def __init__(self, name: str, age: int, is_active: bool):
            self.name = name
            self.age = age
            self.is_active = is_active  # Newly added field

Then, when faced with binary data serialized by the new writer, old readers
will be able to read the following fields.

.. code:: python

    class EmployeeSerializer(CompactSerializer[Employee]):
        def read(self, reader: CompactReader) -> Employee:
            name = reader.read_string("name")
            age = reader.read_int32("age")
            # The new "is_active" field is there, but the old reader does not
            # know anything about it. Hence, it will simply ignore that field.
            return Employee(name, age)

        ...

Then, when faced with binary data serialized by the old writer, new readers
will be able to read the following fields. Also, Hazelcast provides convenient
APIs to read default values when there is no such field in the data.

.. code:: python

    class EmployeeSerializer(CompactSerializer[Employee]):
        def read(self, reader: CompactReader) -> Employee:
            name = reader.read_string("name")
            age = reader.read_int32("age")
            # Read the "is_active" if it exists, or the default value `False`.
            # reader.read_boolean("is_active") would throw if the "is_active"
            #field does not exist in data.
            is_active = reader.read_boolean_or("is_active", False)
            return Employee(name, age, is_active)

        ...

Note that, when an old reader reads data written by an old writer, or a new
reader reads a data written by a new writer, they will be able to read all
fields.

Limitations
~~~~~~~~~~~

Currently, the following APIs are not fully supported with the Compact
serialization format. They may or may not work, depending on whether the
schema is available on the client or not.

All of these APIs will work with the Compact serialization format, once it is
promoted to the stable status.

- Reading OBJECT columns of the SQL results
- Listening for :class:`hazelcast.proxy.reliable_topic.ReliableTopic` messages
- :func:`hazelcast.proxy.list.List.iterator`
- :func:`hazelcast.proxy.list.List.list_iterator`
- :func:`hazelcast.proxy.list.List.get_all`
- :func:`hazelcast.proxy.list.List.sub_list`
- :func:`hazelcast.proxy.map.Map.values`
- :func:`hazelcast.proxy.map.Map.entry_set`
- :func:`hazelcast.proxy.map.Map.execute_on_keys`
- :func:`hazelcast.proxy.map.Map.key_set`
- :func:`hazelcast.proxy.map.Map.project`
- :func:`hazelcast.proxy.map.Map.execute_on_entries`
- :func:`hazelcast.proxy.map.Map.get_all`
- :func:`hazelcast.proxy.multi_map.MultiMap.remove_all`
- :func:`hazelcast.proxy.multi_map.MultiMap.key_set`
- :func:`hazelcast.proxy.multi_map.MultiMap.values`
- :func:`hazelcast.proxy.multi_map.MultiMap.entry_set`
- :func:`hazelcast.proxy.multi_map.MultiMap.get`
- :func:`hazelcast.proxy.queue.Queue.iterator`
- :func:`hazelcast.proxy.replicated_map.ReplicatedMap.values`
- :func:`hazelcast.proxy.replicated_map.ReplicatedMap.entry_set`
- :func:`hazelcast.proxy.replicated_map.ReplicatedMap.key_set`
- :func:`hazelcast.proxy.set.Set.get_all`
- :func:`hazelcast.proxy.ringbuffer.Ringbuffer.read_many`
- :func:`hazelcast.proxy.transactional_map.TransactionalMap.values`
- :func:`hazelcast.proxy.transactional_map.TransactionalMap.key_set`
- :func:`hazelcast.proxy.transactional_multi_map.TransactionalMultiMap.get`
- :func:`hazelcast.proxy.transactional_multi_map.TransactionalMultiMap.remove_all`

IdentifiedDataSerializable Serialization
----------------------------------------

For a faster serialization of objects, Hazelcast recommends to extend
the ``IdentifiedDataSerializable`` class.

The following is an example of a class that extends
``IdentifiedDataSerializable``:

.. code:: python

    from hazelcast.serialization.api import IdentifiedDataSerializable

    class Address(IdentifiedDataSerializable):
        def __init__(self, street=None, zip_code=None, city=None, state=None):
            self.street = street
            self.zip_code = zip_code
            self.city = city
            self.state = state

        def get_class_id(self):
            return 1

        def get_factory_id(self):
            return 1

        def write_data(self, output):
            output.write_string(self.street)
            output.write_int(self.zip_code)
            output.write_string(self.city)
            output.write_string(self.state)

        def read_data(self, input):
            self.street = input.read_string()
            self.zip_code = input.read_int()
            self.city = input.read_string()
            self.state = input.read_string()


.. Note:: Refer to ``ObjectDataInput``/``ObjectDataOutput`` classes in
    the ``hazelcast.serialization.api`` package to understand methods
    available on the ``input``/``output`` objects.

The IdentifiedDataSerializable uses ``get_class_id()`` and
``get_factory_id()`` methods to reconstitute the object. To complete the
implementation, an ``IdentifiedDataSerializable`` factory should also be
created and registered into the client using the
``data_serializable_factories`` argument. A factory is a dictionary that
stores class ID and the ``IdentifiedDataSerializable`` class type pairs
as the key and value. The factory’s responsibility is to store the right
``IdentifiedDataSerializable`` class type for the given class ID.

A sample ``IdentifiedDataSerializable`` factory could be created as
follows:

.. code:: python

    factory = {
        1: Address
    }

Note that the keys of the dictionary should be the same as the class IDs
of their corresponding ``IdentifiedDataSerializable`` class types.

.. Note:: For IdentifiedDataSerializable to work in Python client, the
    class that inherits it should have default valued parameters in its
    ``__init__`` method so that an instance of that class can be created
    without passing any arguments to it.

The last step is to register the ``IdentifiedDataSerializable`` factory
to the client.

.. code:: python

    client = hazelcast.HazelcastClient(
        data_serializable_factories={
            1: factory
        }
    )

Note that the ID that is passed as the key of the factory is same as the
factory ID that the ``Address`` class returns.

Portable Serialization
----------------------

As an alternative to the existing serialization methods, Hazelcast
offers portable serialization. To use it, you need to extend the
``Portable`` class. Portable serialization has the following advantages:

- Supporting multiversion of the same object type.
- Fetching individual fields without having to rely on the reflection.
- Querying and indexing support without deserialization and/or
  reflection.

In order to support these features, a serialized ``Portable`` object
contains meta information like the version and concrete location of the
each field in the binary data. This way Hazelcast is able to navigate in
the binary data and deserialize only the required field without actually
deserializing the whole object which improves the query performance.

With multiversion support, you can have two members each having
different versions of the same object; Hazelcast stores both meta
information and uses the correct one to serialize and deserialize
portable objects depending on the member. This is very helpful when you
are doing a rolling upgrade without shutting down the cluster.

Also note that portable serialization is completely language independent
and is used as the binary protocol between Hazelcast server and clients.

A sample portable implementation of a ``Foo`` class looks like the
following:

.. code:: python

    from hazelcast.serialization.api import Portable

    class Foo(Portable):
        def __init__(self, foo=None):
            self.foo = foo

        def get_class_id(self):
            return 1

        def get_factory_id(self):
            return 1

        def write_portable(self, writer):
            writer.write_string("foo", self.foo)

        def read_portable(self, reader):
            self.foo = reader.read_string("foo")


.. Note:: Refer to ``PortableReader``/``PortableWriter`` classes in the
    ``hazelcast.serialization.api`` package to understand methods
    available on the ``reader``/``writer`` objects.


.. Note:: For Portable to work in Python client, the class that
    inherits it should have default valued parameters in its ``__init__``
    method so that an instance of that class can be created without
    passing any arguments to it.

Similar to ``IdentifiedDataSerializable``, a ``Portable`` class must
provide the ``get_class_id()`` and ``get_factory_id()`` methods. The
factory dictionary will be used to create the ``Portable`` object given
the class ID.

A sample ``Portable`` factory could be created as follows:

.. code:: python

    factory = {
        1: Foo
    }

Note that the keys of the dictionary should be the same as the class IDs
of their corresponding ``Portable`` class types.

The last step is to register the ``Portable`` factory to the client.

.. code:: python

    client = hazelcast.HazelcastClient(
        portable_factories={
            1: factory
        }
    )

Note that the ID that is passed as the key of the factory is same as the
factory ID that ``Foo`` class returns.

Versioning for Portable Serialization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

More than one version of the same class may need to be serialized and
deserialized. For example, a client may have an older version of a class
and the member to which it is connected may have a newer version of the
same class.

Portable serialization supports versioning. It is a global versioning,
meaning that all portable classes that are serialized through a member
get the globally configured portable version.

You can declare the version using the ``portable_version`` argument, as
shown below.

.. code:: python

    client = hazelcast.HazelcastClient(
        portable_version=1
    )

If you update the class by changing the type of one of the fields or by
adding a new field, it is a good idea to upgrade the version of the
class, rather than sticking to the global version specified in the
configuration. In the Python client, you can achieve this by simply
adding the ``get_class_version()`` method to your class’s implementation
of ``Portable``, and returning class version different than the default
global version.

.. Note:: If you do not use the ``get_class_version()`` method in your
    ``Portable`` implementation, it will have the global version, by
    default.

Here is an example implementation of creating a version 2 for the above
Foo class:

.. code:: python

    from hazelcast.serialization.api import Portable

    class Foo(Portable):
        def __init__(self, foo=None, foo2=None):
            self.foo = foo
            self.foo2 = foo2

        def get_class_id(self):
            return 1

        def get_factory_id(self):
            return 1

        def get_class_version(self):
            return 2

        def write_portable(self, writer):
            writer.write_string("foo", self.foo)
            writer.write_string("foo2", self.foo2)

        def read_portable(self, reader):
            self.foo = reader.read_string("foo")
            self.foo2 = reader.read_string("foo2")

You should consider the following when you perform versioning:

- It is important to change the version whenever an update is performed
  in the serialized fields of a class, for example by incrementing the
  version.
- If a client performs a Portable deserialization on a field and then
  that Portable is updated by removing that field on the cluster side,
  this may lead to problems such as an AttributeError being raised when
  an older version of the client tries to access the removed field.
- Portable serialization does not use reflection and hence, fields in
  the class and in the serialized content are not automatically mapped.
  Field renaming is a simpler process. Also, since the class ID is
  stored, renaming the Portable does not lead to problems.
- Types of fields need to be updated carefully. Hazelcast performs
  basic type upgradings, such as ``int`` to ``float``.

Example Portable Versioning Scenarios:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Assume that a new client joins to the cluster with a class that has been
modified and class’s version has been upgraded due to this modification.

If you modified the class by adding a new field, the new client’s put
operations include that new field. If this new client tries to get an
object that was put from the older clients, it gets null for the newly
added field.

If you modified the class by removing a field, the old clients get null
for the objects that are put by the new client.

If you modified the class by changing the type of a field to an
incompatible type (such as from ``int`` to ``str``), a ``TypeError``
(wrapped as ``HazelcastSerializationError``) is generated as the client
tries accessing an object with the older version of the class. The same
applies if a client with the old version tries to access a new version
object.

If you did not modify a class at all, it works as usual.

Custom Serialization
--------------------

Hazelcast lets you plug a custom serializer to be used for serialization
of objects.

Let’s say you have a class called ``Musician`` and you would like to
customize the serialization for it, since you may want to use an
external serializer for only one class.

.. code:: python

    class Musician:
        def __init__(self, name):
            self.name = name

Let’s say your custom ``MusicianSerializer`` will serialize
``Musician``. This time, your custom serializer must extend the
``StreamSerializer`` class.

.. code:: python

    from hazelcast.serialization.api import StreamSerializer

    class MusicianSerializer(StreamSerializer):
        def get_type_id(self):
            return 10

        def destroy(self):
            pass

        def write(self, output, obj):
            output.write_string(obj.name)

        def read(self, input):
            name = input.read_string()
            return Musician(name)

Note that the serializer ``id`` must be unique as Hazelcast will use it
to lookup the ``MusicianSerializer`` while it deserializes the object.
Now the last required step is to register the ``MusicianSerializer`` to
the client.

.. code:: python

    client = hazelcast.HazelcastClient(
        custom_serializers={
            Musician: MusicianSerializer
        }
    )

From now on, Hazelcast will use ``MusicianSerializer`` to serialize
``Musician`` objects.

JSON Serialization
------------------

You can use the JSON formatted strings as objects in Hazelcast cluster.
Creating JSON objects in the cluster does not require any server side
coding and hence you can just send a JSON formatted string object to the
cluster and query these objects by fields.

In order to use JSON serialization, you should use the
``HazelcastJsonValue`` object for the key or value.

``HazelcastJsonValue`` is a simple wrapper and identifier for the JSON
formatted strings. You can get the JSON string from the
``HazelcastJsonValue`` object using the ``to_string()`` method.

You can construct ``HazelcastJsonValue`` from strings or JSON
serializable Python objects. If a Python object is provided to the
constructor, ``HazelcastJsonValue`` tries to convert it to a JSON
string. If an error occurs during the conversion, it is raised directly.
If a string argument is provided to the constructor, it is used as it
is.

In the constructor, no JSON parsing is performed. It is your responsibility
to provide correctly formatted JSON strings. The client will not validate the
string, it will send it to the cluster as it is. If you submit incorrectly
formatted JSON strings and, later, if you query those objects, it is highly
possible that you will get formatting errors since the server will fail to
deserialize or find the query fields.

Here is an example of how you can construct a ``HazelcastJsonValue`` and
put to the map:

.. code:: python

    # From JSON string
    json_map.put("item1", HazelcastJsonValue("{\"age\": 4}"))

    # # From JSON serializable object
    json_map.put("item2", HazelcastJsonValue({"age": 20}))

You can query JSON objects in the cluster using the ``Predicate`` of
your choice. An example JSON query for querying the values whose age is
less than 6 is shown below:

.. code:: python

    # Get the objects whose age is less than 6
    result = json_map.values(less_or_equal("age", 6))
    print("Retrieved %s values whose age is less than 6." % len(result))
    print("Entry is", result[0].to_string())

Global Serialization
--------------------

The global serializer is identical to custom serializers from the
implementation perspective. The global serializer is registered as a
fallback serializer to handle all other objects if a serializer cannot
be located for them.

By default, ``cPickle/pickle`` serialization is used if the class is not
``IdentifiedDataSerializable`` or ``Portable`` or there is no custom
serializer for it. When you configure a global serializer, it is used
instead of ``cPickle/pickle`` serialization.

**Use Cases:**

- Third party serialization frameworks can be integrated using the
  global serializer.

- For your custom objects, you can implement a single serializer to
  handle all of them.

A sample global serializer that integrates with a third party serializer
is shown below.

.. code:: python

    import some_third_party_serializer
    from hazelcast.serialization.api import StreamSerializer

    class GlobalSerializer(StreamSerializer):
        def get_type_id(self):
            return 20

        def destroy(self):
            pass

        def write(self, output, obj):
            output.write_string(some_third_party_serializer.serialize(obj))

        def read(self, input):
            return some_third_party_serializer.deserialize(input.read_string())

You should register the global serializer to the client.

.. code:: python

    client = hazelcast.HazelcastClient(
        global_serializer=GlobalSerializer
    )
