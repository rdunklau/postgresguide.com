Custom functions
================

Postgresql offers you the ability to write functions in numerous languages:

  - SQL
  - pl/pgsql
  - C
  - python
  - many others!

You can even bring support for your own language.

Writing functions
-----------------

A function can be written using the create function statement.

This statement must provide (at least)

  - the function name
  - its arguments
  - its return type
  - what language is it written in
  - the body of the function

You can find more thorough information on the `CREATE FUNCTION statement documentation <http://www.postgresql.org/docs/9.1/static/sql-createfunction.html>`_.

A simple example
~~~~~~~~~~~~~~~~

Let's begin by a simple stutter sql function, which will accept a single text argument,
and will concatenate it to itself.

.. code-block:: sql

  CREATE FUNCTION stutter(to_stutter text) RETURNS text LANGUAGE sql AS $$
    SELECT $1 || $1;
  $$;


The function arguments can be named: the function declared above names its only
argument `to_stutter`.

.. note::

  Functions writter in the `SQL` language must refer to their arguments by their
  position, indexed from 1.
  The ability to use named arguments in SQL function bodies will be added in
  Postgresql 9.2.
  Other languages do not suffer this limitation.

The function body, which will be evaluated in the target language (here, sql),
must be provided as a string literal.

.. note::

  For more convenience, we usually use the `Dollar-quoted string
  constants`. 
  See the `documentation, section 4.1.2.4  <http://www.postgresql.org/docs/current/static/sql-syntax-lexical.html>`_ for more information.


The function can then be called in a regular sql statement:


.. code-block:: sql

  SELECT stutter('postgresguide.com ');


This will result in:

.. code-block:: sql

                stutter               
  --------------------------------------
   postgresguide.com postgresguide.com 


Function can return almost anything, from sql data types to tables... 


Managing functions
------------------

When developing, it is useful to declare functions with the `CREATE OR REPLACE
FUNCTION` statement.

If the function already exists with the same parameters and return type, it will
update the definition.

Let's say we want the stutter function to concatenate the given text once more:

.. code-block:: sql

  CREATE OR replace FUNCTION stutter(to_stutter text) RETURNS text LANGUAGE sql AS $$
    SELECT $1 || $1 || $1;
  $$;


However, if you change the number / type of arguments, a new function will be
created, leaving the old definition around.

If you alter the return type, or the argument names, you have to drop the
function first.

The drop function statement refers to the function signature, including the
argument types.

.. code-block:: sql

  DROP FUNCTION stutter(text);

Function volatibility
---------------------

Every function has a volatibility classifier. 

There are 3 volatibility classifiers:

`VOLATILE`
  The default, this indicates that the function can do anything, including
  modifying the database.

`STABLE`
  The function does not modify the database, and will (at least for the duration
  of statement), return the same value if it is called with the same arguments.

`IMMUTABLE`
  The function will always return the same value when called with the same
  arguments. This is similar to `pure functions` in functional programming.


These qualifications help the postgresql optimizer to cache the result of a
function, and to forbid functions from being used as an index, or in an
index-scan clause.

- A function must be `STABLE` or `IMMUTABLE` to be used in an index-scan clause.
  This is because the index scan will evaluate the condition only once.

- A function must be `IMMUTABLE` to be used as an index.

See the `documentation
<http://www.postgresql.org/docs/9.1/static/xfunc-volatility.html>`_ for more
information on functions volatibility.


Examples
~~~~~~~~

The following function should be marked as `VOLATILE`, the default, since it
will return different values each time it is called. 

The function will return a random integer, between 0 and the given integer:

.. code-block:: sql

  CREATE FUNCTION random_integer(an_int integer) RETURNS integer AS $$
    SELECT (random() * $1)::integer;
  $$ LANGUAGE sql VOLATILE;

.. note::

  - The various parts of the `CREATE FUNCTION` statement can appear in any
    order, before or after the function body. For example, the language it is
    implemented in has been written after the function body in this example,
    whereas it was written before in the previous example.

  - The random() built-in function returns a `double precision`, thus we need to
    cast the result to integer using the ``::integer`` notation to return it.

As we see, this function will return different results every time.

.. code-block:: sql

  SELECT random_integer(6), last_name from employees;

.. code-block:: sql

   random_integer | last_name 
  ----------------+-----------
                5 | Jones
                3 | Adams
                0 | Johnson

The following  function should be marked `STABLE`, because it will always return
the same result when invoked during the same statement. That is because the
`now()` function returns the time at which the transaction began, which is the
same for the whole statement duration. The `now()` function is stable itself.

.. code-block:: sql

  CREATE FUNCTION sleep_and_time(an_int integer) RETURNS timestamptz AS $$
    SELECT n from (
      SELECT now() as n, pg_sleep($1) as sleep
    ) t ;
  $$ language sql STABLE;


.. code-block:: sql

  select last_name, sleep_and_time(3) from employees;

.. code-block:: sql

  last_name |        sleep_and_time         
  -----------+-------------------------------
   Jones     | 2012-04-29 10:37:38.306299+02
   Adams     | 2012-04-29 10:37:38.306299+02
   Johnson   | 2012-04-29 10:37:38.306299+02

Finally, the following function can be marked `IMMUTABLE` because it will always
return the same result, no watter what.

.. code-block:: sql

  CREATE FUNCTION square(an_int integer) RETURNS integer AS $$
    SELECT $1 * $1;
  $$ language SQL IMMUTABLE;

