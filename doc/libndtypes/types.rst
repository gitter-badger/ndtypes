

.. meta::
   :robots: index,follow
   :description: libndtypes documentation


Types
=====

Types are implemented as a tagged union.  For the defined type enum values
it is best to refer to :macro:`ndtypes.h` directly or to search the constructor
functions below.


Abstract and concrete types
---------------------------

.. topic:: abstract_concrete

.. code-block:: c

   /* Protect access to concrete type fields. */
   enum ndt_access {
     Abstract,
     Concrete
   };

An important concept in libndtypes are abstract and concrete types.

Abstract types can have symbolic values like dimension or type variables
and are used for type checking.

Concrete types additionally have full memory layout information like
alignment and data size.

In order to protect against accidental access to undefined concrete fields, types
have the *ndt_access* field that is set to *Abstract* or *Concrete*.


Flags
-----

.. code-block:: c

   /* flags */
   #define NDT_LITTLE_ENDIAN  0x00000001U
   #define NDT_BIG_ENDIAN     0x00000002U
   #define NDT_OPTION         0x00000004U
   #define NDT_SUBTREE_OPTION 0x00000008U
   #define NDT_ELLIPSIS       0x00000010U

The endian flags are set if a type has explicit endianness. If native order
is used, they are unset.

:macro:`NDT_OPTION` is set if a type itself is optional.

:macro:`NDT_SUBTREE_OPTION` is set if any subtree of a type is optional.

:macro:`NDT_ELLIPSIS` is set if the tail of a dimension sequence contains
an ellipsis dimension.  The flag is not propagated to an outer array with
a dtype that contains an inner array with an ellipsis.


Common fields
-------------

.. topic:: common_fields

.. code-block:: c

   struct _ndt {
       /* Always defined */
       enum ndt tag;
       enum ndt_access access;
       uint32_t flags;
       int ndim;
       /* Undefined if the type is abstract */
       int64_t datasize;
       uint16_t align;
       ...
   };

*tag*, *access* and *flags* are explained above.  Every type has an *ndim* field
even when it is not an array, in which case *ndim* is zero.

The *datasize* and *align* fields are defined for concrete types.


Abstract fields
---------------

.. topic:: abstract_fields

.. code-block:: c

    union {
        ...

        struct {
            int64_t shape;
            ndt_t *type;
        } FixedDim;

        ...

    };

These fields are always defined for both abstract and concrete types.
:macro:`FixedDim` is just an example field.  Refer to :macro:`ndtypes.h`
directly for the complete set of fields.


Concrete fields
---------------

.. topic:: concrete_fields

.. code-block:: c

   struct {
        union {
            struct {
                int64_t itemsize;
                int64_t step;
            } FixedDim;

        ...

        };
   } Concrete;

These fields are only defined for concrete types.  For internal reasons
(facilitating copying etc.) they are initialized to zero for abstract
types.


Type constructor functions
--------------------------

All functions in this section steal their arguments.  On success, heap
allocated memory like *type* and *name* arguments belong to the return
value.

On error, all arguments are deallocated within the respective functions.


Special types
--------------

The types in this section all have some property that makes them different
from the regular types.

.. topic:: ndt_option

.. code-block:: c

   ndt_t *ndt_option(ndt_t *type);

This constructor is unique in that it does *not* create a new type with an
:macro:`Option` tag, but sets the :macro:`NDT_OPTION` flag of its argument.

The reason is that having a separate :macro:`Option` tag complicates the
type traversal when using libndtypes.

The function returns its argument and cannot fail.


.. topic:: ndt_module

.. code-block:: c

   ndt_t *ndt_module(char *name, ndt_t *type, ndt_context_t *ctx);

The module type is for implementing type name spaces and is always abstract.
Used in type checking.


.. topic:: ndt_function

.. code-block:: c

   ndt_t *ndt_function(ndt_t *ret, ndt_t *pos, ndt_t *kwds, ndt_context_t *ctx);

The function type is used for declaring function signatures.
Used in type checking.


.. topic:: ndt_void

.. code-block:: c

   ndt_t *ndt_void(ndt_context_t *ctx)

Currently only used as the empty return value in function signatures.


Any type
--------

.. topic:: ndt_any

.. code-block:: c

   ndt_t *ndt_any_kind(ndt_context_t *ctx);

Constructs the abstract *Any* type.  Used in type checking.


Dimension types
---------------

.. code-block:: c

   ndt_t *ndt_fixed_dim(ndt_t *type, int64_t shape, int64_t step, ndt_context_t *ctx);

*type* is either a dtype or the tail of the dimension list.

*shape* is the dimension size and must be a natural number.

*step* is the amount to add to the linear index in order to move to
the next dimension element. *step* may be negative.


If *step* is :macro:`INT64_MAX`, the steps are computed from the dimensions
shapes and the resulting array is C-contiguous. This is the regular case.

If *step* is given, it is used without further checks. This is mostly useful
for slicing. The computed datasize is the minimum datasize such that all index
combinations are within the bounds of the allocated memory.


.. topic:: ndt_to_fortran

.. code-block:: c

   ndt_t *ndt_to_fortran(const ndt_t *type, ndt_context_t *ctx);

Convert a C-contiguous chain of fixed dimensions to Fortran order.



.. topic:: ndt_abstract_var_dim

.. code-block:: c

   ndt_t *ndt_abstract_var_dim(ndt_t *type, ndt_context_t *ctx);

Create an abstract *var* dimension for pattern matching.



.. topic:: ndt_var_dim

.. code-block:: c

   /* Ownership flag for var dim offsets */
   enum ndt_offsets {
     InternalOffsets,
     ExternalOffsets,
   };

   ndt_t *ndt_var_dim(ndt_t *type,
                      enum ndt_offsets flag, int32_t noffsets, const int32_t *offsets,
                      int32_t nslices, ndt_slice_t *slices,
                      ndt_context_t *ctx);


Create a concrete *var* dimension.  Variable dimensions are offset-based
and use the same addressing scheme as the Arrow data format.

Offset arrays can be very large, so copying must be avoided. For ease of
use, libndtypes supports creating offset arrays from a datashape string.
In that case, *flag* must be set to :macro:`InternalOffsets` and the offsets
are managed by the type.

However, in the most common case offsets are generated and managed elsewhere.
In that case, *flag* must be set to :macro:`ExternalOffsets`.


The offset-based scheme makes it hard to store a sliced var dimension or
repeatedly slice a var dimension.  This would require additional shape
arrays that are as large as the offset arrays.

Instead, var dimensions have the concept of a slice stack that stores
all slices that need to be applied to a var dimension.

Accessing elements recomputes the (start, stop, step) triples that result
from applying the entire slice stack.

The *nslices* and *slices* arguments are used to provide this stack.  For
an unsliced var dimension these arguments must be *0* and *NULL*.



.. topic:: ndt_symbolic_dim

.. code-block:: c

   ndt_t *ndt_symbolic_dim(char *name, ndt_t *type, ndt_context_t *ctx);

Create a dimension variable for pattern matching. The variable stands for
a fixed dimension.



.. topic:: ndt_ellipsis_dim

.. code-block:: c

   ndt_ellipsis_dim(char *name, ndt_t *type, ndt_context_t *ctx);

Create an ellipsis dimension for pattern matching. If *name* is non-NULL,
a named ellipsis variable is created.

In pattern matching, multiple named ellipsis variables always stand for
the exact same sequence of dimensions.

By contrast, multiple unnamed ellipses stand for any sequence of dimensions
that can be broadcast together.


Container types
---------------

.. topic:: ndt_tuple

.. code-block:: c

   ndt_t *ndt_tuple(enum ndt_variadic flag, ndt_field_t *fields, int64_t shape,
                    uint16_opt_t align, uint16_opt_t pack, ndt_context_t *ctx);

Construct a tuple type. *fields* is the field sequence, *shape* the length
of the tuple.

*align* and *pack* are mutually exclusive and have the exact same meaning
as gcc's *aligned* and *packed* attributes applied to an entire struct.

Either of these may only be given if no field has an *align* or *pack*
attribute.


.. topic:: ndt_record

.. code-block:: c

   ndt_t *ndt_record(enum ndt_variadic flag, ndt_field_t *fields, int64_t shape,
                     uint16_opt_t align, uint16_opt_t pack, ndt_context_t *ctx);

Construct a record (struct) type. *fields* is the field sequence, *shape*
the length of the record.

*align* and *pack* are mutually exclusive and have the exact same meaning
as gcc's *aligned* and *packed* attributes applied to an entire struct.

Either of these may only be given if no field has an *align* or *pack*
attribute.


.. topic:: ndt_ref

.. code-block:: c

   ndt_t *ndt_ref(ndt_t *type, ndt_context_t *ctx);

Construct a reference type.  References are pointers whose contents (the values
pointed to) are addressed transparently.


.. topic:: ndt_constr

.. code-block:: c

   ndt_t *ndt_constr(char *name, ndt_t *type, ndt_context_t *ctx);

Create a constructor type.  Constructor types are equal if their names
and types are equal.


.. topic:: ndt_nominal

.. code-block:: c

   ndt_t *ndt_nominal(char *name, ndt_t *type, ndt_context_t *ctx);

Same as constructor, but the type is stored in a lookup table. Comparisons
and pattern matching are only by name.  The name is globally unique.

