.. index:: error handling

**************
Error Handling
**************

This chapter describes the way that SciC functions report and handle
errors.  By examining the status information returned by every function
you can determine whether it succeeded or failed, and if it failed you
can find out what the precise cause of failure was.  You can also define
your own error handling functions to modify the default behavior of the
library.

The functions described in this chapter are declared in the header
file :file:`scic/errno.h`.

Error Reporting
===============

The library follows the thread-safe error reporting conventions of the
POSIX Threads library.  Functions return a non-zero error code to
indicate an error and :code:`0` to indicate success::

    int status = scic_function (...)

    if (status) { /* an error occurred */
      .....
      /* status value specifies the type of error */
    }

The routines report an error whenever they cannot perform the task
requested of them.  For example, a root-finding function would return a
non-zero error code if could not converge to the requested accuracy, or
exceeded a limit on the number of iterations.  Situations like this are
a normal occurrence when using any mathematical library and you should
check the return status of the functions that you call.

Whenever a routine reports an error the return value specifies the type
of error.  The return value is analogous to the value of the variable
:code:`errno` in the C library.  The caller can examine the return code
and decide what action to take, including ignoring the error if it is
not considered serious.

In addition to reporting errors by return codes the library also has an
error handler function :func:`scic_error`.  This function is called by
other library functions when they report an error, just before they
return to the caller.  The default behavior of the error handler is to
print a message and abort the program::

    scic: file.c:67: ERROR: invalid argument supplied by user
    Default SCIC error handler invoked.
    Aborted

The purpose of the :func:`scic_error` handler is to provide a function
where a breakpoint can be set that will catch library errors when
running under the debugger.  It is not intended for use in production
programs, which should handle any errors using the return codes.

.. index::
   single: error codes, reserved

Error Codes
===========

The error code numbers returned by library functions are defined in
the file :file:`scic/errno.h`.  They all have the prefix :code:`SCIC_` and
expand to non-zero constant integer values. Error codes above 1024 are
reserved for applications, and are not used by the library.  Many of
the error codes use the same base name as the corresponding error code
in the C library.  Here are some of the most common error codes,

.. index::
   single: error codes

.. var:: int SCIC_EDOM

   Domain error; used by mathematical functions when an argument value does
   not fall into the domain over which the function is defined (like
   :data:`EDOM` in the C library)

.. var:: int SCIC_ERANGE

   Range error; used by mathematical functions when the result value is not
   representable because of overflow or underflow (like :data:`ERANGE` in the C
   library)

.. var:: int SCIC_ENOMEM

   No memory available.  The system cannot allocate more virtual memory
   because its capacity is full (like :data:`ENOMEM` in the C library).  This error
   is reported when a SCIC routine encounters problems when trying to
   allocate memory with :func:`malloc`.

.. var:: int SCIC_EINVAL

   Invalid argument.  This is used to indicate various kinds of problems
   with passing the wrong argument to a library function (like :data:`EINVAL` in the C
   library).

The error codes can be converted into an error message using the
function :func:`scic_strerror`.

.. function:: const char * scic_strerror (const int scic_errno)

   This function returns a pointer to a char describing the error code
   :data:`scic_errno`. For example::

     printf ("error: %s\n", scic_strerror (status));

   would print an error message like :code:`error: output range error` for a
   status value of :data:`SCIC_ERANGE`.

.. index:: error handlers

Error Handlers
==============

The default behavior of the SCIC error handler is to print a short
message and call :func:`abort`.  When this default is in use programs
will stop with a core-dump whenever a library routine reports an error.
This is intended as a fail-safe default for programs which do not check
the return status of library routines (we don't encourage you to write
programs this way).

If you turn off the default error handler it is your responsibility to
check the return values of routines and handle them yourself.  You can
also customize the error behavior by providing a new error handler. For
example, an alternative error handler could log all errors to a file,
ignore certain error conditions (such as underflows), or start the
debugger and attach it to the current process when an error occurs.

All SCIC error handlers have the type :code:`scic_error_handler_t`, which is
defined in :file:`scic_errno.h`,

.. type:: scic_error_handler_t

   This is the type of SCIC error handler functions.  An error handler will
   be passed four arguments which specify the reason for the error (a
   string), the name of the source file in which it occurred (also a
   string), the line number in that file (an integer) and the error number
   (an integer).  The source file and line number are set at compile time
   using the :code:`__FILE__` and :code:`__LINE__` directives in the
   preprocessor.  An error handler function returns type :code:`void`.
   Error handler functions should be defined like this::

     void handler (const char * reason,
                   const char * file,
                   int line,
                   int scic_errno)

To request the use of your own error handler you need to call the
function :func:`scic_set_error_handler` which is also declared in
:file:`scic_errno.h`,

.. function:: scic_error_handler_t * scic_set_error_handler (scic_error_handler_t * new_handler)

   This function sets a new error handler, :data:`new_handler`, for the SCIC
   library routines.  The previous handler is returned (so that you can
   restore it later).  Note that the pointer to a user defined error
   handler function is stored in a static variable, so there can be only
   one error handler per program.  This function should be not be used in
   multi-threaded programs except to set up a program-wide error handler
   from a master thread.  The following example shows how to set and
   restore a new error handler::

     /* save original handler, install new handler */
     old_handler = scic_set_error_handler (&my_handler);

     /* code uses new handler */
     .....

     /* restore original handler */
     scic_set_error_handler (old_handler);

   To use the default behavior (:func:`abort` on error) set the error
   handler to :code:`NULL`::

     old_handler = scic_set_error_handler (NULL);

.. function:: scic_error_handler_t * scic_set_error_handler_off ()

   This function turns off the error handler by defining an error handler
   which does nothing. This will cause the program to continue after any
   error, so the return values from any library routines must be checked.
   This is the recommended behavior for production programs.  The previous
   handler is returned (so that you can restore it later).

The error behavior can be changed for specific applications by
recompiling the library with a customized definition of the
:code:`SCIC_ERROR` macro in the file :file:`scic_errno.h`.

.. index:: error handling macros

Using SCIC error reporting in your own functions
===============================================

If you are writing numerical functions in a program which also uses SCIC
code you may find it convenient to adopt the same error reporting
conventions as in the library.

To report an error you need to call the function :func:`scic_error` with a
string describing the error and then return an appropriate error code
from :file:`scic_errno.h`, or a special value, such as :code:`NaN`.  For
convenience the file :file:`scic_errno.h` defines two macros which carry
out these steps:

.. macro:: SCIC_ERROR (reason, scic_errno)

   This macro reports an error using the SCIC conventions and returns a
   status value of :code:`scic_errno`.  It expands to the following code fragment::

     scic_error (reason, __FILE__, __LINE__, scic_errno);
     return scic_errno;

   The macro definition in :file:`scic_errno.h` actually wraps the code
   in a :code:`do { ... } while (0)` block to prevent possible
   parsing problems.

Here is an example of how the macro could be used to report that a
routine did not achieve a requested tolerance.  To report the error the
routine needs to return the error code :code:`SCIC_ETOL`::

    if (residual > tolerance)
      {
        SCIC_ERROR("residual exceeds tolerance", SCIC_ETOL);
      }

.. macro:: SCIC_ERROR_VAL (reason, scic_errno, value)

   This macro is the same as :code:`SCIC_ERROR` but returns a user-defined
   value of :data:`value` instead of an error code.  It can be used for
   mathematical functions that return a floating point value.

The following example shows how to return a :code:`NaN` at a mathematical
singularity using the :code:`SCIC_ERROR_VAL` macro::

    if (x == 0)
      {
        SCIC_ERROR_VAL("argument lies on singularity", SCIC_ERANGE, SCIC_SCIC_NAN);
      }
