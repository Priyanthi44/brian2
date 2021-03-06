Computational methods and efficiency
====================================

.. contents::
    :local:
    :depth: 1

Brian has several different methods for running the computations in a
simulation. The default mode is :ref:`runtime`, which runs the simulation loop
in Python but compiles and executes the modules doing the actual simulation
work (numerical integration, synaptic propagation, etc.) in a defined target
language. Brian will select the best available target language automatically.
On Windows, to ensure that you get the advantages of compiled code, read
the instructions on installing a suitable compiler in
:ref:`compiler_setup_windows`.
Runtime mode has the advantage that you can combine the computations
performed by Brian with arbitrary Python code specified as `NetworkOperation`.

The fact that the simulation is run in Python means that there is a (potentially
big) overhead for each simulated time step. An alternative is to run Brian in with
:ref:`cpp_standalone` -- this is in general faster (for certain types of simulations
*much* faster) but cannot be used for all kinds of simulations. To enable this
mode, add the following line after your Brian import, but before your simulation
code::

    set_device('cpp_standalone')

For detailed control over the compilation process (both for runtime and standalone
code generation), you can change the :ref:`compiler_settings` that are used.

.. admonition:: The following topics are not essential for beginners.

    |

.. _runtime:

Runtime code generation
-----------------------
Code generation means that Brian takes the Python code and strings
in your model and generates code in one of several possible different
languages and actually executes that. The target language for this code
generation process is set in the `codegen.target` preference. By default, this
preference is set to ``'auto'``, meaning that it will chose a compiled language
target if possible and fall back to Python otherwise (it will also raise a warning
in this case, set `codegen.target` to ``'numpy'`` explicitly to avoid this warning).
There are two compiled language targets for Python 2.x, ``'weave'`` (needing a
working installation of a C++ compiler) and ``'cython'`` (needing the `Cython`_
package in addition); for Python 3.x, only ``'cython'`` is available. If you want to
chose a code generation target explicitly (e.g. because you want to get rid of the
warning that only the Python fallback is available), set the preference to ``'numpy'``,
``'weave'`` or ``'cython'`` at the beginning of your script::

    from brian2 import *
    prefs.codegen.target = 'numpy'  # use the Python fallback

See :doc:`../advanced/preferences` for different ways of setting preferences.

.. warning::
    Do not use the ``weave`` code generation targets when running multiple
    simulations in parallel. See :doc:`../introduction/known_issues` for more
    details.

 .. _Cython: http://cython.org/

Caching
~~~~~~~
When you run code with ``weave`` or ``cython`` for the first time, it will take
some time to compile the code. For short simulations, this can make these
targets to appear slow compared to the ``numpy`` target where such compilation
is not necessary. However, the compiled code is stored on disk and will be
re-used for later runs, making these simulations start faster. If you run many
simulations with different code (e.g. Brian's
:doc:`test suite <../developer/guidelines/testing>`), this code can take quite
a bit of space on the disk. During the import of the ``brian2`` package, we
check whether the size of the disk cache exceeds the value set by the
`codegen.max_cache_dir_size` preference (by default, 1GB) and display a message
if this is the case. You can clear the disk cache manually, or use the
`~brian2.__init__.clear_cache` function, e.g. ``clear_cache('cython')``.


.. _cpp_standalone:

Standalone code generation
--------------------------
Brian supports generating standalone code for multiple devices. In this mode, running a Brian script generates
source code in a project tree for the target device/language. This code can then be compiled and run on the device,
and modified if needed. At the moment, the only "device" supported is standalone C++ code.
In some cases, the speed gains can be impressive, in particular for smaller networks with complicated spike
propagation rules (such as STDP).

To use the C++ standalone mode, you only have to make very small changes to your script. The exact change depends on
whether your script has only a single `run` (or `Network.run`) call, or several of them:

Single run call
~~~~~~~~~~~~~~~
At the beginning of the script, i.e. after the import statements, add::

    set_device('cpp_standalone')

The `CPPStandaloneDevice.build` function will be automatically called with default arguments right after the `run`
call. If you need non-standard arguments then you can specify them as part of the `set_device` call::

    set_device('cpp_standalone', directory='my_directory', debug=True)

Multiple run calls
~~~~~~~~~~~~~~~~~~
At the beginning of the script, i.e. after the import statements, add::

    set_device('cpp_standalone', build_on_run=False)

After the last `run` call, call `device.build` explicitly::

    device.build(directory='output', compile=True, run=True, debug=False)

The `~CPPStandaloneDevice.build` function has several arguments to specify the output directory, whether or not to
compile and run the project after creating it and whether or not to compile it with debugging support or not.

Multiple builds
~~~~~~~~~~~~~~~
To run multiple full simulations (i.e. multiple ``device.build`` calls, not just
multiple `run` calls as discussed above), you have to reinitialize the device
again::

    device.reinit()
    device.activate()

Note that the device "forgets" about all previously set build options provided
to `set_device` (most importantly the ``build_on_run`` option, but also e.g. the
directory), you'll have to specify them as part of the `Device.activate` call.
Also, `Device.activate` will reset the `defaultclock`, you'll therefore have to
set its ``dt`` *after* the ``activate`` call if you want to use a non-default
value.

Limitations
~~~~~~~~~~~
Not all features of Brian will work with C++ standalone, in particular Python based network operations and
some array based syntax such as ``S.w[0, :] = ...`` will not work. If possible, rewrite these using string
based syntax and they should work. Also note that since the Python code actually runs as normal, code that does
something like this may not behave as you would like::

    results = []
    for val in vals:
        # set up a network
        run()
        results.append(result)

The current C++ standalone code generation only works for a fixed number of `~Network.run` statements, not with loops.
If you need to do loops or other features not supported automatically, you can do so by inspecting the generated
C++ source code and modifying it, or by inserting code directly into the main loop as follows::

    device.insert_code('main', '''
    cout << "Testing direct insertion of code." << endl;
    ''')


Variables
~~~~~~~~~
After a simulation has been run (after the `run` call if `set_device` has been called with ``build_on_run`` set to
``True`` or after the `Device.build` call with ``run`` set to ``True``), state variables and
monitored variables can be accessed using standard syntax, with a few exceptions (e.g. string expressions for indexing).

.. _openmp:

Multi-threading with OpenMP
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. warning::
    OpenMP code has not yet been well tested and so may be inaccurate.

When using the C++ standalone mode, you have the opportunity to turn on multi-threading, if your C++ compiler is compatible with
OpenMP. By default, this option is turned off and only one thread is used. However, by changing the preferences of the codegen.cpp_standalone
object, you can turn it on. To do so, just add the following line in your python script::

    prefs.devices.cpp_standalone.openmp_threads = XX

XX should be a positive value representing the number of threads that will be
used during the simulation. Note that the speedup will strongly depend on the
network, so there is no guarantee that the speedup will be linear as a function
of the number of threads. However, this is working fine for networks with not
too small timestep (dt > 0.1ms), and results do not depend on the number of
threads used in the simulation.

.. _standalone_custom_build:

Customizing the build process
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In standalone mode, a standard "make file" is used to orchestrate the
compilation and linking. To provide additional arguments to the ``make`` command
(respectively ``nmake`` on Windows), you can use the
`devices.cpp_standalone.extra_make_args_unix` or
`devices.cpp_standalone.extra_make_args_windows` preference. On Linux,
this preference is by default set to ``['-j']`` to enable parallel compilation.
Note that you can also use these arguments to overwrite variables in the make
file, e.g. to use `clang <https://clang.llvm.org/>`_ instead of the default
`gcc <https://gcc.gnu.org/>`_ compiler::

    prefs.devices.cpp_standalone.extra_make_args_unix += ['CC=clang++']


.. _compiler_settings:

Compiler settings
-----------------

If using C++ code generation (either via weave, cython or standalone), the
compiler settings can make a big difference for the speed of the simulation.
By default, Brian uses a set of compiler settings that switches on various
optimizations and compiles for running on the same architecture where the
code is compiled. This allows the compiler to make use of as many advanced
instructions as possible, but reduces portability of the generated executable
(which is not usually an issue).

If there are any issues with these compiler settings, for example because
you are using an older version of the C++ compiler or because you want to
run the generated code on a different architecture, you can change the
settings by manually specifying the `codegen.cpp.extra_compile_args`
preference (or by using `codegen.cpp.extra_compile_args_gcc` or
`codegen.cpp.extra_compile_args_msvc` if you want to specify the settings
for either compiler only).
