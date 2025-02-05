Algorithm Implementation Tips
=============================

Implementing algorithms is fun, but there are a few things that are good to keep in mind.

In general, development follows the following:

1. Correct
2. Clear
3. Fast

In that order.  Further, we always want LensKit to be *usable* in an easy fashion.  Code
implementing algorithms, however, may be quite complex in order to achieve good performance.

Performance
-----------

We use Numba to optimize critical code paths and provide parallelism in a number of cases,
such as ALS training.  See the ALS source code for examples.

We also directly use MKL sparse matrix routines when available for some operations.  Whenever
this is done in the main LensKit code base, however, we also provide fallback implementations
when the MKL is not available.  The k-NN recommenders both demonstrate different versions of
this.  The ``_mkl_ops`` module exposes MKL operations; we implement them through C wrappers in
the ``mkl_ops.c`` file, that are then called through FFI.  This extra layer is because the raw
MKL calls are quite complex to call via FFI, and are not particularly amenable to use with Numba.
We re-expose simplified interfaces that are also usable with Numba.

Pickling and Sharing
--------------------

LensKit uses Python pickling (or JobLib's modified pickling in :py:func:`joblib.dump`) quite
a bit to save and reload models and to share model data between concurrent processes.  This
generally just works, and you don't need to implement any particular save/load logic in order
to have your algorithm be savable and sharable.

There are a few exceptions, though.

**If your algorithm updates state after fitting**, this should *not* be pickled.  An example of
this would be caching predictions or recommendations to save time in subsequent calls.  Only the
model parameters and estimated parameters should be pickled.  If you have caches or other
ephemeral structures, override ``__getstate__`` and ``__setstate__`` to exclude them from the
saved data and to initialize caches to empty values on unpickling.

**If your model excludes secondary data structures from pickling**, such as a reverse index of
user-item interactions, then you should only exclude them when pickling for serialization. When
pickling for model sharing (see :py:func:`lenskit.sharing.in_share_context`), you should include
the derived structures so they can also be shared.

**If your algorithm uses subsidiary models as a part of the training process**, but does not need them
for prediction or recommendation, then consider overriding ``__getstate__`` to remove the underlying
model or replace it with a cloned copy (with :py:func:`lenskit.util.clone`) to reduce serialized
disk space (and deserialized memory use).

Random Number Generation
------------------------

LensKit uses :py:mod:`seedbank` for managing RNG seeds and constructing random number generation.

In general, algorithms using randomization should have an ``rng_spec`` parameter that takes a seed
or RNG, and pass this to :py:func:`seedbank.numpy_rng` to get a random number generator. Algorithms
that use randomness at predict or recommendation time, not just training time, should support the
value ``'user'`` for the ``rng`` parameter, and if it is passed, derive a new seed for each user
using :py:func:`seedbank.derive_seed` to allow reproducibility in the face of parallelism for common
experimental designs.  :py:func:`lenskit.util.derivable_rng` automates this logic.

Memory Map Friendliness
-----------------------

LensKit uses :py:class:`joblib.Parallel` to parallelize internal operations (when it isn't using Numba).
Joblib is pretty good about using shared memory to minimize memory overhead in parallel computations,
and LensKit has some tricks to maximize this use. However, it does require a bit of attention in
your algorithm implementation.

The easiest way to make this fail is to use many small NumPy or Pandas data structures.  If you have
a dictionary of :py:class:`np.ndarray` objects, for instance, it will cause a problem.  This is because
each array will be memory-mapped, and each map will *reopen* the file.  Having too many active
open files will cause your process to run out of file descriptors on many systems.  Keep your
object count to a small, ideally fixed number; in :py:class:`lenskit.algorithms.basic.UnratedItemSelector`,
we do this by storing user and item indexes along with a :py:class:`matrix.CSR` containing the items
rated by each user.  The old implementation had a dictionary mapping user IDs to ``ndarray``s with
each user's rated items.  This is a change from :math:`|U|+1` arrays to 5 arrays.
