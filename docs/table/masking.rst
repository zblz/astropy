.. include:: references.txt

Masking and missing values
--------------------------

The `astropy.table` package provides support for masking and missing
values in a table by wrapping the `numpy.ma` masked array package.
This allows handling tables with missing or invalid entries in much
the same manner as for standard (unmasked) tables.  It
is useful to be familiar with the `masked array
<http://docs.scipy.org/doc/numpy/reference/maskedarray.generic.html>`_
documentation when using masked tables within `astropy.table`.

In a nutshell, the concept is to define a boolean mask that mirrors
the structure of the table data array.  Wherever a mask value is
``True``, the corresponding entry is considered to be missing or invalid.
Operations involving column or row access and slicing are unchanged.
The key difference is that arithmetic or reduction operations involving
columns or column slices follow the rules for `operations
on masked arrays
<http://docs.scipy.org/doc/numpy/reference/maskedarray.generic.html#operations-on-masked-arrays>`_.

.. Note::

   Reduction operations like `numpy.sum` or `numpy.mean` follow the
   convention of ignoring masked (invalid) values.  This differs from
   the behavior of the floating point ``NaN``, for which the sum of an
   array including one or more ``NaN's`` will result in ``NaN``.
   See `<http://numpy.scipy.org/NA-overview.html>`_ for a very
   interesting discussion of different strategies for handling
   missing data in the context of `numpy`.

.. Note::

   Masked tables are only available for `numpy` version 1.5 and later
   because of issues in the masked array implementation for
   prior `numpy` versions.

Table creation
^^^^^^^^^^^^^^^

A masked table can be created in several ways:

**Create a new table object and specify masked=True** ::

  >>> from astropy.table import Table, Column, MaskedColumn
  >>> t = Table([(1, 2), (3, 4)], names=('a', 'b'), masked=True)
  >>> t
  <Table rows=2 names=('a','b')>
  masked_array(data = [(1, 3) (2, 4)],
               mask = [(False, False) (False, False)],
         fill_value = (999999, 999999),
              dtype = [('a', '<i8'), ('b', '<i8')])

Notice the table attributes ``mask`` and ``fill_value`` that are
available for a masked table.

**Create a table with one or more columns as a MaskedColumn object**

  >>> a = MaskedColumn('a', [1, 2])
  >>> b = Column('b', [3, 4])
  >>> t = Table([a, b])

The |MaskedColumn| is the masked analog of the |Column| class and
provides the interface for creating and manipulating a column of
masked data.  The |MaskedColumn| class inherits from
`numpy.ma.MaskedArray`, in contrast to |Column| which inherits from
`numpy.ndarray`.  This distinction is the main reason there are
different classes for these two cases.

**Create a table with one or more columns as a numpy MaskedArray**

  >>> from numpy import ma  # masked array package
  >>> a = ma.array([1, 2])
  >>> b = [3, 4]
  >>> t = Table([a, b], names=('a', 'b'))

**Add a MaskedColumn object to an existing table**

  >>> a = Column('a', [1, 2])
  >>> b = MaskedColumn('b', [3, 4], mask=[True, False])
  >>> t = Table([a])
  >>> t.add_column(b)
  INFO: Upgrading Table to masked Table [astropy.table.table]

Note the INFO message because the underlying type of the table is modified in this operation.

**Add a new row to an existing table and specify a mask argument**

  >>> a = Column('a', [1, 2])
  >>> b = Column('b', [3, 4])
  >>> t = Table([a, b])
  >>> t.add_row([3, 6], mask=[True, False])
  INFO: Upgrading Table to masked Table [astropy.table.table]

**Convert an existing table to a masked table**

  >>> t = Table([[1, 2], ['x', 'y']])  # standard (unmasked) table
  >>> t = Table(t, masked=True)  # convert to masked table

Table access
^^^^^^^^^^^^

Nearly all the of standard methods for accessing and modifying data
columns, rows, and individual elements also apply to masked tables.

There are two minor differences for the |Row| object that is obtained by
indexing a single row of a table:  

- For standard tables, two such rows can be compared for equality, but
  in masked tables this comparison will produce an exception.
- For standard tables a |Row| object provides a view of the underlying
  table data so that it is possible to modify a table by modifying the
  row values.  In masked tables this is a copy so that modifying the
  |Row| object has no effect on the original table data.

Both of these differences are due to issues in the underlying
`numpy.ma.MaskedArray` implementation.

Masking and filling
^^^^^^^^^^^^^^^^^^^^

Both the |Table| and |MaskedColumn| classes provide 
attributes and methods to support manipulating tables with missing or
invalid data.

Mask
""""

The actual mask for the table as a whole or a single column can be
viewed and modified via the ``mask`` attribute::

  >>> t = Table([(1, 2), (3, 4)], names=('a', 'b'), masked=True)
  >>> t.mask['a'] = [False, True]  # Modify table mask (structured array)
  >>> t['b'].mask = [True, False]  # Modify column mask (boolean array)
  >>> print(t)
   a   b 
  --- ---
    1  --
   --   4

Masked entries are shown as ``--`` when the table is printed.

Filling
"""""""

The entries which are masked (i.e. missing or invalid) can be replaced
with specified fill values.  In this case the |MaskedColumn| or masked
|Table| will be converted to a standard |Column| or table. Each column
in a masked table has a ``fill_value`` attribute that specifies the
default fill value for that column.  To perform the actual replacement
operation the ``filled()`` method is called.  This takes an optional
argument which can override the default column ``fill_value``
attribute.
::

  >>> t['a'].fill_value = -99
  >>> t['b'].fill_value = 33

  >>> print t.filled()
   a   b 
  --- ---
    1  33
  -99   4

  >>> print t['a'].filled()
   a 
  ---
    1
  -99

  >>> print t['a'].filled(999)
   a 
  ---
    1
  999

  >>> print t.filled(1000)
   a    b  
  ---- ----
     1 1000
  1000    4
