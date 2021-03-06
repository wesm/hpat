.. _supported:

User Guide
==========

HPAT supports a subset of Python that is commonly used for data analytics and
machine learning. This section describes this subset.

HPAT compiles and parallelizes the functions annotated with the `@hpat.jit`
decorator. Therefore, file I/O and computations on large datasets should be
inside the jitted functions. The supported data structures for large datasets
are `Numpy <http://www.numpy.org/>`_ arrays and
`Pandas <http://pandas.pydata.org/>`_ dataframes.

Enabling Parallelization
------------------------

To enable parallelization, HPAT needs to recognize the large datasets and their
computation in the program. Hence, only the supported data-parallel operators of
Numpy and Pandas can be used for large datasets and computations. The sequential
computation on small data can be any code that
`Numba supports <http://numba.pydata.org/numba-doc/latest/index.html>`_.

Supported Numpy Operations
--------------------------

Below is the list of the data-parallel Numpy operators that HPAT can optimize
and parallelize.

1. Numpy `element-wise` array operations:

    * Unary operators: ``+`` ``-`` ``~``
    * Binary operators: ``+`` ``-`` ``*`` ``/`` ``/?`` ``%`` ``|`` ``>>`` ``^``
        ``<<`` ``&`` ``**`` ``//``
    * Comparison operators: ``==`` ``!=`` ``<`` ``<=`` ``>`` ``>=``
    * data-parallel math operations: ``add``, ``subtract``, ``multiply``,
        ``divide``, ``logaddexp``, ``logaddexp2``, ``true_divide``,
        ``floor_divide``, ``negative``, ``power``, ``remainder``,
        ``mod``, ``fmod``, ``abs``, ``absolute``, ``fabs``, ``rint``, ``sign``,
        ``conj``, ``exp``, ``exp2``, ``log``, ``log2``, ``log10``, ``expm1``,
        ``log1p``, ``sqrt``, ``square``, ``reciprocal``, ``conjugate``
    * Trigonometric functions: ``sin``, ``cos``, ``tan``, ``arcsin``,
        ``arccos``, ``arctan``, ``arctan2``, ``hypot``, ``sinh``, ``cosh``,
        ``tanh``, ``arcsinh``, ``arccosh``, ``arctanh``, ``deg2rad``,
        ``rad2deg``, ``degrees``, ``radians``
    * Bit manipulation functions: ``bitwise_and``, ``bitwise_or``,
        ``bitwise_xor``, ``bitwise_not``, ``invert``, ``left_shift``,
        ``right_shift``

2. Numpy reduction functions ``sum`` and ``prod``.

3. Numpy array creation functions ``empty``, ``zeros``, ``ones``,
    ``empty_like``, ``zeros_like``, ``ones_like``, ``full_like``, ``copy``.

4. Random number generator functions: ``rand``, ``randn``,
    ``ranf``, ``random_sample``, ``sample``, ``random``,
    ``standard_normal``, ``chisquare``, ``weibull``, ``power``, ``geometric``,
    ``exponential``, ``poisson``, ``rayleigh``, ``normal``, ``uniform``,
    ``beta``, ``binomial``, ``f``, ``gamma``, ``lognormal``, ``laplace``,
    ``randint``, ``triangular``.

4. Numpy ``dot`` function between a matrix and a vector, or two vectors.

Optional arguments are not supported unless if explicitly mentioned here.
For operations on multi-dimensional arrays, automatic broadcast of
dimensions of size 1 is not supported.


Explicit Parallel Loops
-----------------------

Sometimes a program cannot be written in terms of data-parallel operators easy
and explicit parallel loops are required.
In this case, one can use HPAT's ``prange`` instead of ``range`` to specify that a
loop can be parallelized. The user is required to make sure that the loop does
not have cross iteration dependencies except the supported reductions.
Currently, only sum using the ``+=`` operator is supported.
The example below demonstrates a parallel loop with a
reduction::

    from HPAT import jit, prange
    @jit
    def prange_test(n):
        A = np.random.ranf(n)
        s = 0
        for i in prange(len(A)):
            s += A[i]
        return s

Supported Pandas Operations
---------------------------

Below is the list of the Pandas operators that HPAT supports. Since Numba
doesn't support Pandas, only these operations can be used for both large and
small datasets.

1. HPAT supports Dataframe creation with the ``DataFrame`` constructor.
    Only a dictionary is supported as input. For example::

        df = pd.DataFrame({'A': np.ones(n), 'B': np.random.ranf(n)})

2. Accessing columns using both getitem (e.g. ``df['A']``) and attribute
    (e.g. ``df.A``) is supported.

3. Using columns similar to Numpy arrays and performing data-parallel operations
    listed previously is supported.

4. Filtering data frames using boolean arrays is supported
    (e.g. ``df[df.A > .5]``).

5. Rolling window operations with `window` and `center` options are supported.
    Here are a few examples::

         df.A.rolling(window=5).mean()
         df.A.rolling(3, center=True).apply(lambda a: a[0]+2*a[1]+a[2])

6. ``shift`` operation (e.g. ``df.A.shift(1)``) and ``pct_change`` operation
    (e.g. ``df.A.pct_change()``) are supported.

File I/O
--------

Currently, HPAT only supports I/O for the `HDF5 <http://www.h5py.org/>`_ format.
The syntax is the same as the `h5py <http://www.h5py.org/>`_ package.
For example::

    @hpat.jit
    def example():
        f = h5py.File("lr.hdf5", "r")
        X = f['points'][:]
        Y = f['responses'][:]

HPAT needs to know the types of input arrays. If the file name is a constant
string, HPAT tries to look at the file at compile time and recognize the types.
Otherwise, the user is responsile for providing the types similar to
`Numba's typing syntax
<http://numba.pydata.org/numba-doc/latest/reference/types.html>`_. For
example::

     @hpat.jit(locals={'X': hpat.float64[:,:], 'Y': hpat.float64[:]})
     def example(file_name):
         f = h5py.File(file_name, "r")
         X = f['points'][:]
         Y = f['responses'][:]

Strings
-------

Currently, HPAT provides basic ASCII string support. Constant strings, equality
comparison of strings (``==`` and ``!=``), ``split`` function, extracting
characters (e.g. ``s[1]``), concatination, and convertion to `int` and `float`
are supported. Here are some examples::

    s = 'test_str'
    flag = (s == 'test_str')
    flag = (s != 'test_str')
    s_list = s.split('_')
    c = s[1]
    s = s+'_test'
    a = int('12')
    b = float('1.2')

Dictionaries
------------

HPAT supports basic integer dictionaries currently. ``DictIntInt`` is the type
for dictionaries with 64-bit integer keys and values, while ``DictInt32Int32``
is for 32-bit integer ones. Getting and setting values, ``pop`` and ``get``
operators, as well as ``min`` and ``max`` of keys is supported. For example::

    d = DictIntInt()
    d[2] = 3
    a = d[2]
    b = d.get(3, 0)
    d.pop(2)
    d[3] = 4
    a = min(d.keys())
