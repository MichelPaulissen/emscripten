.. _FAQ:

===
FAQ
===

This FAQ contains answers to many questions asked on IRC and the mailing list.

How do I compile code?
======================

See the :ref:`Emscripten Tutorial <Tutorial>` and :ref:`emcc <emccdoc>`.


Why do I get multiple errors building basic code and the tests?
===============================================================

All the tests in the :ref:`Emscripten test suite <emscripten-test-suite>` are known to build and pass on the **master** branch, so if these are failing it is likely that there is some problem with your environment.

First call ``./emcc -v``, which runs basic sanity checks and prints out useful environment information. If that doesn't help, follow the instructions in :ref:`verifying-the-emscripten-environment`.

You might also want to go through the :ref:`Tutorial` again, as this is updated as Emscripten changes.


I tried something: why doesn’t it work?
=======================================

If something doesn't work (for example a :ref:`compiler flag <emccdoc>`, a *libc* function, etc.) then first search the comprehensive documentation on this site.

Next check if there is a test for the failing functionality in the :ref:`Emscripten test suite <emscripten-test-suite>` (run ``grep -r`` in **tests/**). **All** the tests are known to pass on the master branch, so they provide concrete "known-good" examples of how various options and code are used.


Do I need to change my build system to use Emscripten?
======================================================

In most cases you will be able to use your project's current build system with Emscripten. See :ref:`Building-Projects`.



Why is code compilation so slow?
================================

Emscripten makes some trade-offs that make the generated code faster and smaller, at the cost of longer compilation times. For example, we build parts of the standard library along with your code, which enables some additional optimizations, but takes a little longer to compile.

.. note:: You can determine what compilation steps take longest by compiling with ``EMCC_DEBUG=1`` in the environment and then reviewing the debug logs (by default in ``/tmp/emscripten_temp``). Note that compiling in debug mode takes longer than normal, because we print out a lot of intermediate steps to disk.

The main tips for improving build time are:

- Create fully optimized builds less frequently — use ``-O0`` during frequent development iterations (or don't specify an optimization level).
	
	- Compiling with higher levels of optimization can in some cases be noticeably slower: ``-O2`` is slower than ``-O1``, which is in turn slower than ``-O0``. 
	- Compiling with ``-O3`` is **especially** slow — this can be mitigated by also enabling ``-s AGGRESSIVE_VARIABLE_ELIMINATION=1`` (removing variables makes the ``-O3`` regalloc easier).
	
- Compile without :ref:`line number debug information <emcc-g>` (use ``-O1`` or ``-g0`` instead of ``-g``):

	- Currently builds with line-number debug information are slow (see issue `#216 <https://github.com/kripken/emscripten/issues/216>`_). Stripping the debug information significantly improves compile times.
	
- Compile on a machine with more cores: 

	- Emscripten can run some passes in parallel (specifically, the JavaScript optimisations). Increasing the number of cores results in an almost linear improvement. 
	- Emscripten will automatically use more cores if they are available. You can control how many cores are used  with ``EMCC_CORES=N`` (this is useful if you have many cores but relatively less memory).


	
Why does my code run slowly?
============================

Make sure you optimize code by building with ``-O2`` (even more :ref:`aggressive optimization <emcc-O3>` is available, at the cost of significantly increased compilation time).

.. note: This is necessary both for each source file, and for the final stage of linking and compiling to JavaScript. For more information see :ref:`Building-Projects` and :ref:`Optimizing-Code`.


Why is my compiled code so big?
===============================

Make sure you build with ``-O2`` so code is optimized and minified. You should also set up gzip compression on your webserver, which all browsers now support.

.. note:: You can :ref:`use the closure compiler <emcc-closure>` to reduce code size even further (``--closure 1``). However that will require that your code be prepared for closure compiler advanced optimizations, including proper exports and so forth. It is usually not worth the effort over an optimized build and supporting gzip on your webserver.



Why does compiling code that works on another machine gives me errors?
======================================================================

Make sure you are using the Emscripten bundled system headers. Using :ref:`emcc <emccdoc>` will do so by default, but problems may occur if you use your local system headers with ``emcc`` or compile into LLVM bitcode yourself.


How can I reduce startup time?
==============================

Make sure that you are running an :ref:`optimized build <Optimizing-Code>` (smaller builds are faster to start up). If the sheer code size is causing the slow startup, you can try `Outlining: a workaround for JITs and big functions <http://mozakai.blogspot.com/2013/08/outlining-workaround-for-jits-and-big.html>`_.

Network latency is also a possible factor in startup time. Consider putting the file loading code in a separate script element from the generated code so that the browser can start the network download in parallel to starting up the codebase (run the :ref:`file packager <packaging-files>` and put file loading code in one script element, and the generated codebase in a later script element).


Why does my code fail to compile with an error message about inline assembly (or ``{"text":"asm"}``)?
=====================================================================================================

Emscripten cannot compile inline assembly code (because it is CPU specific, and Emscripten is not a CPU emulator).

You will need to find where inline assembly is used, and disable it or replace it with platform-independent code. 

.. note:: Emscripten automatically unsets the following ``#define`` values, as these are commonly set in projects to enable platform dependent code (inline assembly):

	::

		#undef __i386__
		#undef __x86_64__


.. _faq-my-html-app-hangs:

Why does my HTML app hang?
==========================

The browser event model uses *co-operative multitasking* — each event has a "turn" to run, and must then return control to the browser event loop so that other events can be processed. A common cause of HTML pages hanging is JavaScript that does not complete and return control to the browser.

Graphical C++ apps typically have an infinite main loop in which event handling, processing and rendering is done, followed by a delay to keep the frame-rate right (``SDL_DELAY`` in :term:`SDL` apps). As the main loop does not complete (is infinite) it cannot return control to the browser, and the app will hang. 

Apps that use an infinite main loop should be re-coded to put the actions for a single iteration of the loop into a single "finite" function. In the native build this function can be run in an infinite loop as before. In the Emscripten build it is set as the :ref:`main loop function <faq-how-run-event-loop>` and will be called by the browser at a specified frequency.

There is more information on this topic in :ref:`emscripten-runtime-environment`.


.. _faq-how-run-event-loop:

How do I run an event loop?
===========================

To run a C function repeatedly, use :c:func:`emscripten_set_main_loop` (this is discussed in :ref:`emscripten-runtime-environment`). The related functions in :ref:`emscripten.h <emscripten-h-browser-execution-environment>` are also useful, allowing you to add events that block the main loop, etc. 

To respond to browser events use the SDL API in the normal way. There are examples in the SDL tests (search for SDL in **tests/runner.py**).

See also: :ref:`faq-my-html-app-hangs`


   
Why doesn't my SDL app work?
=============================

See the :term:`SDL` automatic tests for working examples: ``python tests/runner.py browser``.


How do I link against system libraries like SDL, boost, etc.?
=============================================================

System libraries that are included with Emscripten are automatically linked when you compile (just the necessary parts). This includes *libc*, *libc++* (C++ standard library) and :term:`SDL`.

Libraries not included with Emscripten (like Boost) must be compiled and linked with the program just as if they were a module in the project. For example, see how `BananaBread links in libz <https://github.com/kripken/BananaBread/blob/master/cube2/src/web/Makefile>`_. 

Another option is to implement needed C APIs as JavaScript librarys (see ``--js-library`` in :ref:`emcc <emcc-js-library>` and :ref:`implement-c-in-javascript`). Emscripten itself does this for *libc* (not including *malloc*) and :term:`SDL` (but not *libc++* or *malloc*).  

.. note:: 

	- Unlike other compilers, you don't need ``-lSDL`` to include SDL (specifying it will do no harm).
	- In the specific case of *Boost*, if you only need the boost headers then you don't need to compile anything.


How can my compiled program access files?
=========================================

Emscripten uses a virtual file system that may be preloaded with data or linked to URLs for lazy loading. See the :ref:`file-system-overview` for more details.


Why can't my code access a file in the same directory?
======================================================

Emscripten-generated code running *in the browser* cannot access files in the local file system. Instead you can use :ref:`preloading <emcc-preload-file>` and :ref:`embedding <emcc-embed-file>` to work around the lack of synchronous file IO. See :ref:`file-system-overview` for more information.

It is possible to allow access to local file system for code running in *node.js*.


.. _faq-when-safe-to-call-compiled-functions:

How can I tell when the page is fully loaded and it is safe to call compiled functions?
=======================================================================================

(You may need this answer if you see an error saying something like ``you need to wait for the runtime to be ready (e.g. wait for main() to be called)``.)

Calling a compiled function before a page has fully loaded can result in an error, if the function relies on files that may not be present (for example the :ref:`.mem <emcc-memory-init-file>` file and :ref:`preloaded <emcc-preload-file>` files are loaded asynchronously).

The easiest way to find out when loading is complete is to add a ``main()`` function, and within it call a JavaScript function to notify your code that loading is complete. 

.. note:: The ``main()`` function is called after startup is complete as a signal that it is safe to call any compiled method.

For example, if ``allReady()`` is a JavaScript function you want called when everything is ready, you can do:

::

  #include <emscripten.h>

  int main() {
    EM_ASM( allReady() );
  }

You can also define a ``main()`` function in JavaScript:

::

  Module['_main'] = function() { ... };

or

::

  Module['_main'] = allReady;

What happens in practice is that when code is ready to be run, we check for ``Module._main``. If present, we call it. If a ``main()`` function was compiled from C, it will be there (and it will be a JavaScript function). But, you can also just define a JavaScript function there, either will work.


.. _faq-dead-code-elimination:

Why do functions in my C/C++ source code vanish when I compile to JavaScript, and/or I get ``No functions to process``?
=======================================================================================================================

Emscripten does dead code elimination of functions that are not called from the compiled code. While this does minimize code size, it can remove functions that you plan to call yourself (outside of the compiled code). 

To make sure a C function remains available to be called from normal JavaScript, it must be added to the `EXPORTED_FUNCTIONS <https://github.com/kripken/emscripten/blob/master/src/settings.js#L359>`_ using the *emcc* command line. For example, to prevent functions ``my_func()`` and ``main()`` from being removed/renamed, run *emcc* with: ::

	./emcc -s EXPORTED_FUNCTIONS="['_main', '_my_func']"  ...

If your function is used in other functions, LLVM may inline it and it will not appear as a unique function in the JavaScript. Prevent inlining by defining the function with :c:type:`EMSCRIPTEN_KEEPALIVE`: ::

	void EMSCRIPTEN_KEEPALIVE yourCfunc() {..}
	
.. note:: 

	- All functions not kept alive through ``EXPORTED_FUNCTIONS`` or :c:type:`EMSCRIPTEN_KEEPALIVE` will potentially be removed. Make sure you keep the things you need alive using one or both of those methods.
	
	- Exported functions need to be C functions (to avoid C++ name mangling).

	- Decorating your code with :c:type:`EMSCRIPTEN_KEEPALIVE` can be useful if you don't want to have to keep track of functions to export explicitly, and when these exports do not change. It is not necessarily suitable for exporting functions from other libraries — for example it is not a good idea to decorate and and recompile the source code of the C standard library. If you build the same source in multiple ways and change what is exported, then managing exports on the command line is easier. 

	- Running *emcc* with ``-s LINKABLE=1`` will also disable link-time optimizations and dead code elimination. This is not recommended as it makes the code larger and less optimized. 	
	
Another possible cause of missing code is improper linking of ``.a`` files. The ``.a`` files link only the internal object files needed by previous files on the command line, so the order of files matters, and this can be surprising. If you are linking ``.a`` files, make sure they are at the end of the list of files, and in the right order amongst themselves. Alternatively, just use ``.so`` files instead in your project.

.. tip:: It can be useful to compile with ``EMCC_DEBUG=1`` set for the environment (``EMCC_DEBUG=1 emcc ...`` on Linux, ``set EMMCC_DEBUG=1`` on Windows). This splits up the compilation steps and saves them in ``/tmp/emscripten_temp``. You can then see at what stage the code vanishes (you will need to do ``llvm-dis`` on the bitcode  stages to read them, or ``llvm-nm``, etc.).



Why is the File System API is not available when I build with closure?
======================================================================

The :term:`Closure Compiler` will minify the File Server API code. Code that uses the file system must be optimized **with** the File System API, using emcc's ``--pre-js`` :ref:`option <emcc-pre-js>`.


Why does my code break and gives odd errors when using ``-O2 --closure 1``?
===========================================================================

The :term:`Closure Compiler` minifies variable names, which results in very short variable names like ``i``, ``j``, ``xa``, etc. If other code declares variables with the same names in global scope, this can cause serious problems. 

This is likely to be the cause if you can successfully run code compiled with ``-O2`` set and ``--closure`` unset.

One solution is to stop using small variable names in the global scope (often this is a mistake — forgetting to use ``var`` when assigning to a variable). 

Another alternative is to wrap the generated code (or your other code) in a closure, as shown:

::

	var CompiledModule = (function() {
		.. GENERATED CODE ..
		return Module;
		})();


Why do I get ``undefined is not a function`` or ``NAME is not a function``?
===========================================================================

The likely cause is an undefined function — a function that was referred to, but not implemented or linked in. If you get ``undefined``, look at the line number to see the function name.

Emscripten by default does *not* give fatal errors on undefined symbols, so you can get *runtime* errors like these (because in practice, for many codebases it is easiest to get them working without refactoring them to remove all undefined symbol calls). If you prefer compile-time notifications, run *emcc* with ``-s WARN_ON_UNDEFINED_SYMBOLS=1`` or ``-s ERROR_ON_UNDEFINED_SYMBOLS=1``.

Aside from just forgetting to link in a necessary object file, one possible cause for this error is inline functions in headers. If you have a header with ``inline int my_func() { .. }`` then *Clang* may not actually inline the function (since inline is just a hint), and also not generate code for it (since it's in a header). The result is that the generated bitcode and JavaScript will not have that function implemented. One solution is to add ``static`` to the function declaration, which forces code to be generated in the object file: ``static inline int my_func() { .. }``.


Why do I get an odd python error complaining about libcxx.bc or libcxxabi.bc?
=============================================================================

A possible cause is that building *libcxx* or *libcxxabi* failed. Go to **system/lib/libcxx** (or libcxxabi) and do ``emmake make`` to see the actual error. Or, clean the Emscripten cache (``./emcc --clear-cache``) and then compile your file with ``EMCC_DEBUG=1`` in the environment. *libcxx* will then be built in **/tmp/emscripten_temp/libcxx**, and you can see ``configure*, make*`` files that are the output of configure and make, etc.

Another possible cause of this error is the lack of ``make``, which is necessary to build these libraries. If you are on Windows, you need *cmake*.


Why does running LLVM bitcode generated by emcc through **lli** break with errors about ``impure_ptr``?
=======================================================================================================

.. note:: :term:`lli` is not maintained, and has odd errors and crashes. We do include **tools/nativize_llvm.py** (which compiles bitcode to a native executable) but it will also hit the ``impure_ptr`` error. 

The issue is that *newlib* uses ``impure_ptr`` code, while *glibc* uses something else. The result is that bitcode built with the Emscripten will not run locally unless your machine uses *newlib* (basically, only embedded systems). 

The ``impure_ptr`` error only occurs during explicit use of ``stdout`` etc., so ``printf(..)`` will work, but ``fprintf(stdout, ..)`` will not. **Usually it is simple to modify your code to avoid this problem.**


Why do I get a stack size error when optimizing: ``RangeError: Maximum call stack size exceeded`` or similar?
=============================================================================================================

You may need to increase the stack size for :term:`node.js`. 

On Linux and Mac OS X, you can just do ``NODE_JS = ['node', '--stack_size=8192']`` in the :ref:`compiler-configuration-file`. On Windows, you will also need ``--max-stack-size=8192``, and also run ``editbin /stack:33554432 node.exe``.


Why do I get ``error: cannot compile this aggregate va_arg expression yet`` and it says ``compiler frontend failed to generate LLVM bitcode, halting`` afterwards?
===================================================================================================================================================================

This is a limitation of the le32 frontend in :term:`Clang`. You can use the x86 frontend instead by compiling with ``EMCC_LLVM_TARGET=i386-pc-linux-gnu`` in the environment (however you will lose the advantages of le32, which includes better alignment of doubles).

		
Why does building from source fail during linking (at 100%)?
============================================================

Building :ref:`Fastcomp from source <building-fastcomp-from-source>` (and hence the SDK) can fail at 100% progress. This is due to out of memory in the linking stage, and is reported as an error: ``collect2: error: ld terminated with signal 9 [Killed]``.

The solution is to ensure the system has sufficient memory. On Ubuntu 14.04.1 LTS 64bit, you should use use at least 6Gb.


		
Why the weird name for the project?
===================================

I don't know why; it's a perfectly `cromulent <http://en.wikipedia.org/wiki/Lisa_the_Iconoclast>`_ word!

