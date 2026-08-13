---
url: https://chatgpt.com/c/6a7dade5-6fdc-83eb-b279-65091d857a05
---

## 📗 C Enum, Typedef, and Function

> [!NOTE] Prompt
> 
> Suppose I have a C module and I want to create Python bindings (pyd package), exposing
> 
> - C enum as Python enum
> - C function as python function
> - C typedef and some C function as Python class plus with a method
> 
> I want it done in a cross-platform fashion. What would it take?

A clean cross-platform approach is:

* use **CFFI in API/out-of-line mode** to compile the native extension;
* expose a small C binding API with opaque handles;
* add a thin Python façade for `Enum` and classes;
* build wheels through `pyproject.toml` and `cibuildwheel`.

The result is a `.pyd` on Windows and an equivalent `.so` extension on Linux/macOS.

Suppose the C API is conceptually:

```c
typedef enum {
    MODE_FAST = 1,
    MODE_ACCURATE = 2
} Mode;

typedef struct Processor Processor;

int add(int a, int b);

Processor *processor_new(Mode mode);
void processor_free(Processor *processor);
double processor_process(Processor *processor, double value);
```

### 1. Make the C API binding-friendly

For classes, prefer an opaque structure:

```c
/* mylib.h */

#ifndef MYLIB_H
#define MYLIB_H

#ifdef _WIN32
#  ifdef MYLIB_BUILD
#    define MYLIB_API __declspec(dllexport)
#  else
#    define MYLIB_API __declspec(dllimport)
#  endif
#else
#  define MYLIB_API
#endif

typedef enum {
    MODE_FAST = 1,
    MODE_ACCURATE = 2
} Mode;

typedef struct Processor Processor;

MYLIB_API int add(int a, int b);

MYLIB_API Processor *processor_new(Mode mode);
MYLIB_API void processor_free(Processor *processor);
MYLIB_API double processor_process(
    Processor *processor,
    double value
);

#endif
```

Important properties:

* public functions use ordinary C types;
* the class state is opaque;
* construction and destruction are explicit;
* ownership is unambiguous;
* the public header contains no platform-dependent structure layout;
* exported functions have C linkage if the implementation might be compiled as C++:

```c
#ifdef __cplusplus
extern "C" {
#endif

/* declarations */

#ifdef __cplusplus
}
#endif
```

If Python needs error details, add an explicit error mechanism. Returning `NULL` or an integer status alone is often insufficient.

### 2. Declare the binding surface to CFFI

CFFI does not run a full C preprocessor, so the binding declaration is usually maintained separately:

```python
# src/mypackage/_build.py

from cffi import FFI

ffibuilder = FFI()

ffibuilder.cdef(
    """
    typedef enum {
        MODE_FAST = 1,
        MODE_ACCURATE = 2
    } Mode;

    typedef struct Processor Processor;

    int add(int a, int b);

    Processor *processor_new(Mode mode);
    void processor_free(Processor *processor);
    double processor_process(
        Processor *processor,
        double value
    );
    """
)

ffibuilder.set_source(
    "mypackage._native",
    '#include "mylib.h"',
    sources=["csrc/mylib.c"],
    include_dirs=["csrc"],
)

if __name__ == "__main__":
    ffibuilder.compile(verbose=True)
```

This embeds and compiles the C implementation into the Python extension. Consequently, you do not need to distribute a separate DLL unless that is specifically desirable.

Another option is linking an existing library:

```python
ffibuilder.set_source(
    "mypackage._native",
    '#include "mylib.h"',
    include_dirs=["path/to/include"],
    libraries=["mylib"],
    library_dirs=["path/to/lib"],
)
```

That introduces runtime-library distribution concerns, especially DLL discovery on Windows and `rpath` handling on Unix-like systems.

### 3. Expose the C enum as a Python enum

A C enum and a Python enum are different objects. Provide the Python-facing type explicitly:

```python
# src/mypackage/api.py

from enum import IntEnum

from ._native import ffi, lib


class Mode(IntEnum):
    FAST = lib.MODE_FAST
    ACCURATE = lib.MODE_ACCURATE
```

`IntEnum` is normally preferable because its members can be converted directly to the C integer value.

### 4. Expose a C function as a Python function

```python
def add(a: int, b: int) -> int:
    return int(lib.add(a, b))
```

The wrapper is a useful place for:

* Python type conversion;
* range validation;
* translating C errors into Python exceptions;
* documentation and type annotations.

Passing `lib.add` through directly is possible, but produces a poorer Python API.

### 5. Wrap the typedef and related functions as a class

```python
class Processor:
    def __init__(self, mode: Mode) -> None:
        handle = lib.processor_new(int(mode))

        if handle == ffi.NULL:
            raise MemoryError("Could not create Processor")

        self._handle = ffi.gc(handle, lib.processor_free)

    def process(self, value: float) -> float:
        return float(
            lib.processor_process(self._handle, value)
        )
```

Here:

* `Processor *` is the native instance;
* `processor_new()` acts as the constructor;
* `processor_free()` acts as the destructor;
* `processor_process()` becomes a method;
* `ffi.gc()` associates the native destructor with the CFFI object.

For deterministic release, add `close()` and context-manager support rather than depending entirely on garbage collection:

```python
class Processor:
    def __init__(self, mode: Mode) -> None:
        handle = lib.processor_new(int(mode))
        if handle == ffi.NULL:
            raise MemoryError("Could not create Processor")

        self._handle = handle
        self._closed = False

    def close(self) -> None:
        if not self._closed:
            lib.processor_free(self._handle)
            self._handle = ffi.NULL
            self._closed = True

    def process(self, value: float) -> float:
        if self._closed:
            raise RuntimeError("Processor is closed")

        return float(lib.processor_process(self._handle, value))

    def __enter__(self) -> "Processor":
        return self

    def __exit__(self, exc_type, exc_value, traceback) -> None:
        self.close()

    def __del__(self) -> None:
        self.close()
```

Production code should make `__del__` robust against partially completed construction and interpreter shutdown. Using a private `ffi.gc()` owner plus an explicit `close()` method is another workable design.

### 6. Package layout

A compact project could look like:

```text
project/
├── pyproject.toml
├── README.md
├── csrc/
│   ├── mylib.c
│   └── mylib.h
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── api.py
│       └── _build.py
└── tests/
    ├── test_enum.py
    ├── test_functions.py
    └── test_processor.py
```

The generated extension is imported as:

```python
from ._native import ffi, lib
```

`__init__.py` exports only the intended public API:

```python
from .api import Mode, Processor, add

__all__ = ["Mode", "Processor", "add"]
```

### 7. Build configuration

A setuptools-based `pyproject.toml` would contain approximately:

```toml
[build-system]
requires = [
    "setuptools>=77",
    "wheel",
    "cffi>=1.17",
]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["cffi>=1.17"]

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools]
cffi-modules = ["src/mypackage/_build.py:ffibuilder"]
```

The two `[tool.setuptools]` tables must be combined in a real TOML file:

```toml
[tool.setuptools]
package-dir = {"" = "src"}
cffi-modules = ["src/mypackage/_build.py:ffibuilder"]
```

A local build requires:

* MSVC Build Tools on Windows;
* GCC or Clang on Linux;
* Xcode Command Line Tools on macOS.

### 8. Wheel production

Cross-platform does not mean one binary works everywhere. You normally build separate wheels:

* Windows x86-64: `.pyd` inside a Windows wheel;
* Linux x86-64: `.so` inside a manylinux wheel;
* macOS Intel and/or ARM64: `.so` inside corresponding macOS wheels;
* generally one wheel per supported CPython ABI unless using the stable ABI.

`cibuildwheel` can build the platform matrix in GitHub Actions:

```toml
[tool.cibuildwheel]
build = "cp310-* cp311-* cp312-* cp313-*"
test-command = "pytest {project}/tests"
test-requires = ["pytest"]
```

You cannot ordinarily cross-compile all of these reliably from one Windows machine. CI runners for Windows, Linux, and macOS are the practical solution.

### The work involved

For a modest C API, the binding itself is not large. The important work is API engineering:

1. Define a stable, binding-friendly C surface.
2. Establish ownership for every pointer.
3. Define nullability and valid ranges.
4. Design C error reporting.
5. Write the CFFI declarations.
6. Create Python enums, functions, and classes.
7. Translate C failures into Python exceptions.
8. Build and test on all target platforms.
9. Produce platform-specific wheels.
10. Decide whether the C library is embedded or shipped separately.

For your use case—agent-generated bindings and tests for reasonably conventional C—CFFI plus an intentionally designed C API is probably the best balance. It avoids writing CPython reference-counting code, remains ordinary C rather than requiring C++, and gives the Python layer enough control to present a proper Pythonic API.

---
---

## 📗 Bare Bindings

> [!NOTE] Prompt
> 
> What if I do not want to use CFFI as intermediary? There should be some templates with boilerplate, so that I could add a minimum wrapper code?

Yes. You can write the extension directly against the **CPython C API**. It remains pure C and produces a `.pyd` on Windows and `.so` on Linux/macOS.

For minimum maintenance, I would use:

* a small reusable C wrapper template;
* `PyType_FromSpec()` for Python classes;
* `PyArg_ParseTuple()` for argument conversion;
* a thin Python `__init__.py` for `IntEnum` definitions;
* setuptools only as the build driver;
* optionally the CPython Limited API to produce `abi3` wheels.

### Recommended division

Keep mechanical native operations in C:

```text
_native.add()
_native.Processor
_native.Processor.process()
```

Keep declarative Python constructs in Python:

```text
Mode(IntEnum)
```

This avoids constructing a Python `Enum` through cumbersome C API calls while eliminating CFFI entirely.

### Example C API

```c
/* mylib.h */

#ifndef MYLIB_H
#define MYLIB_H

typedef enum {
    MODE_FAST = 1,
    MODE_ACCURATE = 2
} Mode;

typedef struct Processor Processor;

int add(int a, int b);

Processor *processor_new(Mode mode);
void processor_free(Processor *processor);
double processor_process(Processor *processor, double value);

#endif
```

### Direct CPython wrapper

```c
/* python_wrapper.c */

#define PY_SSIZE_T_CLEAN
#include <Python.h>

#include "mylib.h"


typedef struct {
    PyObject_HEAD
    Processor *handle;
} PyProcessor;


/*
** Processor.__new__()
*/
static PyObject *
PyProcessor_new(PyTypeObject *type, PyObject *args, PyObject *kwargs)
{
    PyProcessor *self;

    self = (PyProcessor *)type->tp_alloc(type, 0);
    if (self != NULL) {
        self->handle = NULL;
    }

    return (PyObject *)self;
}


/*
** Processor.__init__(mode)
*/
static int
PyProcessor_init(PyProcessor *self, PyObject *args, PyObject *kwargs)
{
    static char *keywords[] = {"mode", NULL};
    int mode;

    if (!PyArg_ParseTupleAndKeywords(
            args,
            kwargs,
            "i:Processor",
            keywords,
            &mode
        )) {
        return -1;
    }

    if (mode != MODE_FAST && mode != MODE_ACCURATE) {
        PyErr_SetString(PyExc_ValueError, "invalid processor mode");
        return -1;
    }

    /*
    ** Handle an explicit second __init__ call safely.
    */
    if (self->handle != NULL) {
        processor_free(self->handle);
        self->handle = NULL;
    }

    self->handle = processor_new((Mode)mode);
    if (self->handle == NULL) {
        PyErr_NoMemory();
        return -1;
    }

    return 0;
}


/*
** Processor destructor.
*/
static void
PyProcessor_dealloc(PyProcessor *self)
{
    if (self->handle != NULL) {
        processor_free(self->handle);
        self->handle = NULL;
    }

    Py_TYPE(self)->tp_free((PyObject *)self);
}


/*
** Processor.process(value)
*/
static PyObject *
PyProcessor_process(PyProcessor *self, PyObject *args)
{
    double value;
    double result;

    if (self->handle == NULL) {
        PyErr_SetString(
            PyExc_RuntimeError,
            "Processor is not initialized"
        );
        return NULL;
    }

    if (!PyArg_ParseTuple(args, "d:process", &value)) {
        return NULL;
    }

    result = processor_process(self->handle, value);
    return PyFloat_FromDouble(result);
}


static PyMethodDef PyProcessor_methods[] = {
    {
        "process",
        (PyCFunction)PyProcessor_process,
        METH_VARARGS,
        PyDoc_STR("process(value) -> float")
    },
    {NULL, NULL, 0, NULL}
};


static PyType_Slot PyProcessor_slots[] = {
    {Py_tp_new, PyProcessor_new},
    {Py_tp_init, PyProcessor_init},
    {Py_tp_dealloc, PyProcessor_dealloc},
    {Py_tp_methods, PyProcessor_methods},
    {0, NULL}
};


static PyType_Spec PyProcessor_spec = {
    .name = "mypackage._native.Processor",
    .basicsize = sizeof(PyProcessor),
    .itemsize = 0,
    .flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
    .slots = PyProcessor_slots
};


/*
** Module-level add(a, b) function.
*/
static PyObject *
py_add(PyObject *module, PyObject *args)
{
    int a;
    int b;
    int result;

    (void)module;

    if (!PyArg_ParseTuple(args, "ii:add", &a, &b)) {
        return NULL;
    }

    result = add(a, b);
    return PyLong_FromLong(result);
}


static PyMethodDef module_methods[] = {
    {
        "add",
        py_add,
        METH_VARARGS,
        PyDoc_STR("add(a, b) -> int")
    },
    {NULL, NULL, 0, NULL}
};


static struct PyModuleDef module_definition = {
    PyModuleDef_HEAD_INIT,
    .m_name = "_native",
    .m_doc = "Native bindings for mylib.",
    .m_size = -1,
    .m_methods = module_methods
};


PyMODINIT_FUNC
PyInit__native(void)
{
    PyObject *module;
    PyObject *processor_type;

    module = PyModule_Create(&module_definition);
    if (module == NULL) {
        return NULL;
    }

    processor_type = PyType_FromSpec(&PyProcessor_spec);
    if (processor_type == NULL) {
        Py_DECREF(module);
        return NULL;
    }

    if (PyModule_AddObject(
            module,
            "Processor",
            processor_type
        ) < 0) {
        Py_DECREF(processor_type);
        Py_DECREF(module);
        return NULL;
    }

    if (PyModule_AddIntConstant(module, "MODE_FAST", MODE_FAST) < 0 ||
        PyModule_AddIntConstant(
            module,
            "MODE_ACCURATE",
            MODE_ACCURATE
        ) < 0) {
        Py_DECREF(module);
        return NULL;
    }

    return module;
}
```

This provides:

```python
from mypackage import _native

_native.add(2, 3)
processor = _native.Processor(_native.MODE_FAST)
processor.process(1.5)
```

### Python-facing API

```python
# src/mypackage/__init__.py

from enum import IntEnum

from ._native import (
    MODE_ACCURATE,
    MODE_FAST,
    Processor,
    add,
)


class Mode(IntEnum):
    FAST = MODE_FAST
    ACCURATE = MODE_ACCURATE


__all__ = ["Mode", "Processor", "add"]
```

The native constructor accepts an `IntEnum` because `IntEnum` is also an integer:

```python
from mypackage import Mode, Processor, add

assert add(2, 3) == 5

processor = Processor(Mode.FAST)
result = processor.process(1.5)
```

If you want stronger Python-level validation, subclass or wrap `_native.Processor` in Python. The C layer should still validate the raw integer because callers can access `_native` directly.

### Cross-platform build configuration

```toml
[build-system]
requires = ["setuptools>=77", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "0.1.0"
requires-python = ">=3.10"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```

A minimal `setup.py`:

```python
from setuptools import Extension, setup


native_extension = Extension(
    name="mypackage._native",
    sources=[
        "python_wrapper.c",
        "csrc/mylib.c",
    ],
    include_dirs=["csrc"],
)


setup(ext_modules=[native_extension])
```

Then:

```console
python -m pip install .
```

Setuptools selects the available compiler:

* MSVC on Windows;
* GCC or Clang on Linux;
* Apple Clang on macOS.

You do not normally need conditional compiler code unless the underlying C module itself has platform-specific requirements.

### Using the stable ABI

If you want one wheel to support several CPython versions on a given platform, build against the Limited API:

```c
#define Py_LIMITED_API 0x030A0000
#define PY_SSIZE_T_CLEAN
#include <Python.h>
```

And configure:

```python
native_extension = Extension(
    name="mypackage._native",
    sources=[
        "python_wrapper.c",
        "csrc/mylib.c",
    ],
    include_dirs=["csrc"],
    define_macros=[
        ("Py_LIMITED_API", "0x030A0000"),
    ],
    py_limited_api=True,
)
```

This can produce a wheel tagged approximately:

```text
mypackage-0.1.0-cp310-abi3-win_amd64.whl
```

That wheel can generally be used by CPython 3.10 and later on Windows x86-64. You still need distinct wheels for each OS and architecture.

Using `PyType_FromSpec()` is especially useful here because it is compatible with the Limited API, unlike direct manipulation of many `PyTypeObject` fields.

### What should go into a reusable template

Most projects only need to customize five sections:

1. The native handle stored after `PyObject_HEAD`.
2. Constructor argument parsing.
3. Native destructor call.
4. Method wrappers.
5. Module method and type tables.

A practical template can supply reusable conventions for:

* opaque native handles;
* owned versus borrowed pointers;
* constructor and destructor patterns;
* scalar, string, buffer, and array conversion;
* C error-to-Python-exception translation;
* enum constants;
* methods, properties, and class methods;
* build configuration;
* `abi3` compatibility;
* pytest tests for lifetime and failure behavior.

For a small conventional C API, the direct wrapper is often only around 150–300 lines. It has more manual code than CFFI, but no runtime binding dependency and gives complete control over object representation, validation, exceptions, and lifetime.
