Tutorial
========

This tutorial is intended as an introduction to working with
Cassandra and **pycassa**.

.. toctree::
    :maxdepth: 2

Prerequisites
-------------
Before we start, make sure that you have **pycassa**
:doc:`installed <installation>`. In the Python shell, the following
should run without raising an exception:

.. code-block:: python

  >>> import pycassa

This tutorial also assumes that a Cassandra instance is running on the
default host and port. Read the `instructions for getting started
with Cassandra <http://www.datastax.com/docs/0.7/getting_started/index>`_ , 
making sure that you choose a `version that is compatible with
pycassa <http://wiki.github.com/pycassa/pycassa/pycassa-cassandra-compatibility>`_.
You can start Cassandra like so:

.. code-block:: bash

  $ pwd
  ~/cassandra
  $ bin/cassandra -f

Creating a Keyspace and Column Families
---------------------------------------
We need to create a keyspace and some column families to work with.  There
are two good ways to do this: using cassandra-cli, or using pycassaShell. Both
are documented below.

Using cassandra-cli
^^^^^^^^^^^^^^^^^^^
The cassandra-cli utility is included with Cassandra. It allows you to create
and modify the schema, explore or modify data, and examine a few things about
your cluster.  Here's how to create the keyspace and column family we need
for this tutorial:

.. code-block:: none

    user@~ $ cassandra-cli 
    Welcome to cassandra CLI.

    Type 'help;' or '?' for help. Type 'quit;' or 'exit;' to quit.
    [default@unknown] connect localhost/9160;
    Connected to: "Test Cluster" on localhost/9160
    [default@unknown] create keyspace Keyspace1;
    4f9e42c4-645e-11e0-ad9e-e700f669bcfc
    Waiting for schema agreement...
    ... schemas agree across the cluster
    [default@unknown] use Keyspace1;
    Authenticated to keyspace: Keyspace1
    [default@Keyspace1] create column family ColumnFamily1;
    632cf985-645e-11e0-ad9e-e700f669bcfc
    Waiting for schema agreement...
    ... schemas agree across the cluster
    [default@Keyspace1] quit;
    user@~ $

This connects to a local instance of Cassandra and creates a keyspace
named 'Keyspace1' with a column family named 'ColumnFamily1'.

Using pycassaShell
^^^^^^^^^^^^^^^^^^
:ref:`pycassa-shell` is an interactive Python shell that is included
with **pycassa**.  Upon starting, it sets up many of the objects that
you typically work with when using **pycassa**.  It provides most of the
functionality that cassandra-cli does, but also gives you a full Python
environment to work with.

Here's how to create the keyspace and column family:

.. code-block:: bash

    user@~ $ pycassaShell 
    ----------------------------------
    Cassandra Interactive Python Shell
    ----------------------------------
    Keyspace: None
    Host: localhost:9160

    ColumnFamily instances are only available if a keyspace is specified with -k/--keyspace

    Schema definition tools and cluster information are available through SYSTEM_MANAGER.

.. code-block:: python

    >>> SYSTEM_MANAGER.create_keyspace('Keyspace1', replication_factor=1)
    >>> SYSTEM_MANAGER.create_column_family('Keyspace1', 'ColumnFamily1')

Connecting to Cassandra
-----------------------
The first step when working with **pycassa** is to connect to the
running cassandra instance:

.. code-block:: python

  >>> import pycassa
  >>> pool = pycassa.connect('Keyspace1')

The above code will connect by default to ``localhost:9160``. We can
also specify the host and port explicitly, as follows:

.. code-block:: python

  >>> pool = pycassa.connect('Keyspace1', ['localhost:9160'])

This creates a small connection pool for use with a
:class:`~pycassa.columnfamily.ColumnFamily` . See `Connection Pooling`_
for more details.

Getting a ColumnFamily
----------------------
A column family is a collection of rows and columns in Cassandra,
and can be thought of as roughly the equivalent of a table in a
relational database. We'll use one of the column families that
are included in the default schema file:

.. code-block:: python

  >>> pool = pycassa.connect('Keyspace1')
  >>> col_fam = pycassa.ColumnFamily(pool, 'ColumnFamily1')

If you get an error about the keyspace or column family not
existing, make sure you created the keyspace and column family as
shown above.

Inserting Data
--------------
To insert a row into a column family we can use the
:meth:`~pycassa.columnfamily.ColumnFamily.insert` method:

.. code-block:: python

  >>> col_fam.insert('row_key', {'col_name': 'col_val'})
  1354459123410932

We can also insert more than one column at a time:

.. code-block:: python

  >>> col_fam.insert('row_key', {'col_name':'col_val', 'col_name2':'col_val2'})
  1354459123410932

And we can insert more than one row at a time:

.. code-block:: python

  >>> col_fam.batch_insert({'row1': {'name1': 'val1', 'name2': 'val2'},
  ...                       'row2': {'foo': 'bar'}})
  1354491238721387

Getting Data
------------
There are many more ways to get data out of Cassandra than there are
to insert data.

The simplest way to get data is to use
:meth:`~pycassa.columnfamily.ColumnFamily.get()`:

.. code-block:: python

  >>> col_fam.get('row_key')
  {'col_name': 'col_val', 'col_name2': 'col_val2'}

Without any other arguments, :meth:`~pycassa.columnfamily.ColumnFamily.get()`
returns every column in the row (up to `column_count`, which defaults to 100).
If you only want a few of the columns and you know them by name, you can
specify them using a `columns` argument:

.. code-block:: python

  >>> col_fam.get('row_key', columns=['col_name', 'col_name2'])
  {'col_name': 'col_val', 'col_name2': 'col_val2'}

We may also get a slice (or subrange) of the columns in a row. To do this,
use the `column_start` and `column_finish` parameters.  One or both of these may
be left empty to allow the slice to extend to one or both ends.
Note that `column_finish` is inclusive.

.. code-block:: python

    >>> for i in range(1, 10):
    ...     col_fam.insert('row_key', {str(i): 'val'})
    ... 
    1302542571215334
    1302542571218485
    1302542571220599
    1302542571221991
    1302542571223388
    1302542571224629
    1302542571225859
    1302542571227029
    1302542571228472
    >>> col_fam.get('row_key', column_start='5', column_finish='7')
    {'5': 'val', '6': 'val', '7': 'val'}

Sometimes you want to get columns in reverse sorted order.  A common
example of this is getting the last N columns from a row that
represents a timeline.  To do this, set `column_reversed` to ``True``.
If you think of the columns as being sorted from left to right, when
`column_reversed` is ``True``, `column_start` will determine the right
end of the range while `column_finish` will determine the left.

Here's an example of getting the last three columns in a row:

.. code-block:: python

  >>> col_fam.get('row_key', column_reversed=True, column_count=3)
  {'9': 'val', '8': 'val', '7': 'val'}

There are a few ways to get multiple rows at the same time.
The first is to specify them by name using
:meth:`~pycassa.columnfamily.ColumnFamily.multiget()`:

.. code-block:: python

  >>> col_fam.multiget(['row1', 'row2'])
  {'row1': {'name1': 'val1', 'name2': 'val2'}, 'row_key2': {'foo': 'bar'}}

Another way is to get a range of keys at once by using
:meth:`~pycassa.columnfamily.ColumnFamily.get_range()`. The parameter
`finish` is also inclusive here, too.  Assuming we've inserted some rows
with keys 'row_key1' through 'row_key9', we can do this:

.. code-block:: python

  >>> result = col_fam.get_range(start='row_key5', finish='row_key7')
  >>> for key, columns in result:
  ...     print key, '=>', columns
  ...
  'row_key5' => {'name':'val'}
  'row_key6' => {'name':'val'}
  'row_key7' => {'name':'val'}

.. note:: Cassandra must be using an OrderPreservingPartitioner for you to be
          able to get a meaningful range of rows; the default, RandomPartitioner,
          stores rows in the order of the MD5 hash of their keys. See
          http://www.riptano.com/docs/0.7/operations/clustering#partitioners.

The last way to get multiple rows at a time is to take advantage of
secondary indexes by using :meth:`~pycassa.columnfamily.ColumnFamily.get_indexed_slices()`,
which is described in the `Indexes`_ section.

It's also possible to specify a set of columns or a slice for 
:meth:`~pycassa.columnfamily.ColumnFamily.multiget()` and
:meth:`~pycassa.columnfamily.ColumnFamily.get_range()` just like we did for
:meth:`~pycassa.columnfamily.ColumnFamily.get()`.

Counting
--------
If you just want to know how many columns are in a row, you can use
:meth:`~pycassa.columnfamily.ColumnFamily.get_count()`:

.. code-block:: python

  >>> col_fam.get_count('row_key')
  3

If you only want to get a count of the number of columns that are inside
of a slice or have particular names, you can do that as well:

.. code-block:: python

  >>> col_fam.get_count('row_key', columns=['foo', 'bar'])
  2
  >>> col_fam.get_count('row_key', column_start='foo')
  3

You can also do this in parallel for multiple rows using
:meth:`~pycassa.columnfamily.ColumnFamily.multiget_count()`:

.. code-block:: python

  >>> col_fam.multiget_count(['fib0', 'fib1', 'fib2', 'fib3', 'fib4'])
  {'fib0': 1, 'fib1': 1, 'fib2': 2, 'fib3': 3, 'fib4': 5'}

.. code-block:: python

  >>> col_fam.multiget_count(['fib0', 'fib1', 'fib2', 'fib3', 'fib4'],
  ...                        columns=['col1', 'col2', 'col3'])
  {'fib0': 1, 'fib1': 1, 'fib2': 2, 'fib3': 3, 'fib4': 3'}

.. code-block:: python

  >>> col_fam.multiget_count(['fib0', 'fib1', 'fib2', 'fib3', 'fib4'],
  ...                        column_start='col1', column_finish='col3')
  {'fib0': 1, 'fib1': 1, 'fib2': 2, 'fib3': 3, 'fib4': 3'}

Super Columns
-------------
Cassandra allows you to group columns in "super columns". In a
``cassandra.yaml`` file, this looks like this:

::

  - name: Super1
    column_type: Super 

To use a super column in **pycassa**, you only need to
add an extra level to the dictionary:

.. code-block:: python

  >>> col_fam = pycassa.ColumnFamily(pool, 'Super1')
  >>> col_fam.insert('row_key', {'supercol_name': {'col_name': 'col_val'}})
  1354491238721345
  >>> col_fam.get('row_key')
  {'supercol_name': {'col_name': 'col_val'}}

The `super_column` parameter for :meth:`get()`-like methods allows
you to be selective about what subcolumns you get from a single
super column.

.. code-block:: python

  >>> col_fam = pycassa.ColumnFamily(pool, 'Letters')
  >>> col_fam.insert('row_key', {'lowercase': {'a': '1', 'b': '2', 'c': '3'}})
  1354491239132744
  >>> col_fam.get('row_key', super_column='lowercase')
  {'supercol1': {'a': '1': 'b': '2', 'c': '3'}}
  >>> col_fam.get('row_key', super_column='lowercase', columns=['a', 'b'])
  {'supercol1': {'a': '1': 'b': '2'}}
  >>> col_fam.get('row_key', super_column='lowercase', column_start='b')
  {'supercol1': {'b': '1': 'c': '2'}}
  >>> col_fam.get('row_key', super_column='lowercase', column_finish='b', column_reversed=True)
  {'supercol1': {'c': '2', 'b': '1'}}

Typed Column Names and Values
-----------------------------
In Cassandra 0.7, you can specify a comparator type for column names
and a validator type for column values.

The types available are:

* BytesType - no type
* IntegerType - 32 bit integer
* LongType - 64 bit integer
* AsciiType - ASCII string
* UTF8Type - UTF8 encoded string
* TimeUUIDType - version 1 UUID (timestamp based)
* LexicalUUID - non-version 1 UUID

The column name comparator types affect how columns are sorted within
a row. You can use these with standard column families as well as with
super column families; with super column families, the subcolumns may
even have a different comparator type.  Here's an example ``cassandra.yaml``:

::

  - name: StandardInt
    column_type: Standard
    compare_with: IntegerType

  - name: SuperLongSubAscii
    column_type: Super
    compare_with: LongType
    compare_subcolumns_with: AsciiType

Cassandra still requires you to pack these types into a format it can
understand by using something like :meth:`struct.pack()`.  Fortunately,
when **pycassa** sees that a column family uses these types, it knows
to pack and unpack these data types automatically for you. So, if we want to
write to the StandardInt column family, we can do the following:

.. code-block:: python

  >>> col_fam = pycassa.ColumnFamily(pool, 'StandardInt')
  >>> col_fam.insert('row_key', {42: 'some_val'})
  1354491238721387
  >>> col_fam.get('row_key')
  {42: 'some_val'}

Notice that 42 is an integer here, not a string.

As mentioned above, Cassandra also offers validators on column values with
the same set of types.  Validators can be set for an entire column family,
for individual columns, or both.  Here's another example ``cassandra.yaml``:

::

  - name: AllLongs
    column_type: Standard
    default_validation_class: LongType

  - name: OneUUID
    column_type: Standard
    column_metadata:
      - name: uuid
        validator_class: TimeUUIDType

  - name: LongsExceptUUID
    column_type: Standard
    default_validation_class: LongType
    column_metadata:
      - name: uuid
        validator_class: TimeUUIDType

**pycassa** knows to pack these column values automatically too:

.. code-block:: python

  >>> import uuid
  >>> col_fam = pycassa.ColumnFamily(pool, 'LongsExceptUUID')
  >>> col_fam.insert('row_key', {'foo': 123456789, 'uuid': uuid.uuid1()})
  1354491238782746
  >>> col_fam.get('row_key')
  {'foo': 123456789, 'uuid': UUID('5880c4b8-bd1a-11df-bbe1-00234d21610a')}

Of course, if **pycassa**'s automatic behavior isn't working for you, you
can turn it off when you create the
:class:`~pycassa.columnfamily.ColumnFamily`:

.. code-block:: python

  >>> col_fam = pycassa.ColumnFamily(pool, 'Standard1',
  ...                                autopack_names=False,
  ...                                autopack_values=False)

This mainly needs to be done when working with
:class:`~pycassa.columnfamilymap.ColumnFamilyMap`.

Version 1 UUIDs (TimeUUIDType)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Version 1 UUIDs are `frequently used for timelines <http://www.riptano.com/docs/0.6/data_model/uuids>`_
instead of timestamps.  Normally, this makes it difficult to get a slice of
columns for some time range or to create a column name or value for some
specific time.

To make this easier, if a :class:`datetime` object or a timestamp with the
same precision as the output of ``time.time()`` is passed where a TimeUUID
is expected, **pycassa** will convert that into a :class:`uuid.UUID` with an equivalent
timestamp component.

Suppose we have something like Twissandra's public timeline but with TimeUUIDs
for column names. If we want to get all tweets that happened yesterday, we
can do:

.. code-block:: python

  >>> import datetime
  >>> line = pycassa.ColumnFamily(pool, 'Userline')
  >>> today = datetime.datetime.utcnow()
  >>> yesterday = today - datetime.timedelta(days=1)
  >>> tweets = line.get('__PUBLIC__', column_start=yesterday, column_finish=today)

Now, suppose there was a tweet that was supposed to be posted on December 11th
at 8:02:15, but it was dropped and now we need to put it in the public timeline.
There's no need to generate a UUID, we can just pass another datetime object instead:

.. code-block:: python

  >>> from datetime import datetime
  >>> line = pycassa.ColumnFamily(pool, 'Userline')
  >>> time = datetime(2010, 12, 11, 8, 2, 15)
  >>> line.insert('__PUBLIC__', {time: 'some tweet stuff here'})

One limitation of this is that you can't ask for one specific column with a
TimeUUID name by passing a :class:`datetime` through something like the `columns` parameter
for :meth:`get()`; this is because there is no way to know the non-timestamp
components of the UUID ahead of time.  Instead, simply pass the same :class:`datetime`
object for both `column_start` and `column_finish` and you'll get one or more
columns for that exact moment in time.

Indexes
-------
Cassandra 0.7.0 adds support for secondary indexes, which allow you to
efficiently get only rows which match a certain expression.

To use secondary indexes with Cassandra, you need to specify what columns
will be indexed.  In a ``cassandra.yaml`` file, this might look like:

::

  - name: Indexed1
    column_type: Standard
    column_metadata:
      - name: birthdate
        validator_class: LongType
        index_type: KEYS

In order to use :meth:`~pycassa.columnfamily.ColumnFamily.get_indexed_slices()`
to get data from Indexed1 using the indexed column, we need to create an 
:class:`~pycassa.cassandra.ttypes.IndexClause` which contains a list of
:class:`~pycassa.cassandra.ttypes.IndexExpression` objects.  The IndexExpressions
inside the clause are ANDed together, meaning every expression must match for
a row to be returned.

Suppose we have a 'Users' column family with one row per user, and we
want to get all of the users from Utah with a birthdate after 1970.
We can make use of the :mod:`pycassa.index` module to make this easier:

.. code-block:: python

  >>> import pycassa
  >>> from pycassa.index import *
  >>> pool = pycassa.connect('Keyspace1')
  >>> users = pycassa.ColumnFamily(pool, 'Users')
  >>> state_expr = create_index_expression('state', 'Utah')
  >>> bday_expr = create_index_expression('birthdate', 1970, GT)
  >>> clause = create_index_clause([state_expr, bday_expr], count=20)
  >>> for key, user in users.get_indexed_slices(clause):
  ...     print user['name'] + ",", user['state'], user['birthdate']
  John Smith, Utah 1971
  Mike Scott, Utah 1980
  Jeff Bird, Utah 1973

Although at least one
:class:`~pycassa.cassandra.ttypes.IndexExpression` in the clause
must be on an indexed column, you may also have other expressions which are
on non-indexed columns.

Connection Pooling
------------------
Pycassa uses connection pools to maintain connections to Cassandra servers.
The :class:`~pycassa.pool.ConnectionPool` class is used to create the connection
pool.  After creating the pool, it may be used to create multiple
:class:`~pycassa.columnfamily.ColumnFamily` objects.

.. code-block:: python

  >>> pool = pycassa.ConnectionPool('Keyspace1', pool_size=20)
  >>> standard_cf = pycassa.ColumnFamily(pool, 'Standard1')
  >>> standard_cf.insert('key', {'col': 'val'})
  1354491238782746
  >>> super_cf = pycassa.ColumnFamily(pool, 'Super1')
  >>> super_cf.insert('key2', {'column' : {'col': 'val'}})
  1354491239779182
  >>> standard_cf.get('key')
  {'col': 'val'}
  >>> pool.dispose()

Automatic retries (or "failover") happen by default with ConectionPools.
This means that if any operation fails, it will be transparently retried
on other servers until it succeeds or a maximum number of failures is reached.

Class Mapping with Column Family Map
------------------------------------
You can map existing classes to column families using
:class:`~pycassa.columnfamilymap.ColumnFamilyMap`.

.. code-block:: python

  >>> class Test(object):
  ...     string_column       = pycassa.String(default='Your Default')
  ...     int_str_column      = pycassa.IntString(default=5)
  ...     float_str_column    = pycassa.FloatString(default=8.0)
  ...     float_column        = pycassa.Float64(default=0.0)
  ...     datetime_str_column = pycassa.DateTimeString() # default=None

The defaults will be filled in whenever you retrieve instances from the
Cassandra server and the column doesn't exist. If you want to add a
column in the future, you can simply add the relevant attribute to the class
and the default value will be used when you get old instances.

:class:`~pycassa.types.IntString`, :class:`~pycassa.types.FloatString`, and
:class:`~pycassa.types.DateTimeString` all use string representations for
storage. :class:`~pycassa.types.Float64` is stored as a double and is
native-endian. Be aware of any endian issues if you use it on different
architectures, or perhaps make your own column type.

.. code-block:: python

  >>> pool = pycassa.ConnectionPool('Keyspace1')
  >>> cf = pycassa.ColumnFamily(pool, 'Standard1', autopack_names=False, autopack_values=False)
  >>> Test.objects = pycassa.ColumnFamilyMap(Test, cf)

.. note:: As shown in the example, `autopack_names` and `autopack_values` should
          be set to ``False`` when a ColumnFamily is used with a ColumnFamilyMap.

All the functions are exactly the same, except that they return
instances of the supplied class when possible.

.. code-block:: python

  >>> t = Test()
  >>> t.key = 'maptest'
  >>> t.string_column = 'string test'
  >>> t.int_str_column = 18
  >>> t.float_column = t.float_str_column = 35.8
  >>> from datetime import datetime
  >>> t.datetime_str_column = datetime.now()
  >>> Test.objects.insert(t)
  1261395560186855

.. code-block:: python

  >>> Test.objects.get(t.key).string_column
  'string test'
  >>> Test.objects.get(t.key).int_str_column
  18
  >>> Test.objects.get(t.key).float_column
  35.799999999999997
  >>> Test.objects.get(t.key).datetime_str_column
  datetime.datetime(2009, 12, 23, 17, 6, 3)

.. code-block:: python

  >>> Test.objects.multiget([t.key])
  {'maptest': <__main__.Test object at 0x7f8ddde0b9d0>}
  >>> list(Test.objects.get_range())
  [<__main__.Test object at 0x7f8ddde0b710>]
  >>> Test.objects.get_count(t.key)
  5

.. code-block:: python

  >>> Test.objects.remove(t)
  1261395603906864
  >>> Test.objects.get(t.key)
  Traceback (most recent call last):
  ...
  cassandra.ttypes.NotFoundException: NotFoundException()

You may also use a ColumnFamilyMap with super columns:

.. code-block:: python

  >>> Test.objects = pycassa.ColumnFamilyMap(Test, cf)
  >>> t = Test()
  >>> t.key = 'key1'
  >>> t.super_column = 'super1'
  >>> t.string_column = 'foobar'
  >>> t.int_str_column = 5
  >>> t.float_column = t.float_str_column = 35.8
  >>> t.datetime_str_column = datetime.now()
  >>> Test.objects.insert(t)
  >>> Test.objects.get(t.key)
  {'super1': <__main__.Test object at 0x20ab350>}
  >>> Test.objects.multiget([t.key])
  {'key1': {'super1': <__main__.Test object at 0x20ab550>}}
