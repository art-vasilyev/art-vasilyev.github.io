---
layout: post
title: "Distributing python packages protected with Cython"
description: Describing how to protect and distribute python application    
date: 2019-11-25 09:00:00 +0300
categories: Programming Python
---

# Distributing python packages protected with Cython

One of the possible solutions to protect the source code of a python application is to use Cython. Cython translates source code into C/C++ code and compiles it. Resulting extensions still can be reverse-engineered, but not reversed to equivalent source code (like from byte-code). The problem with distributing compiled extensions is that they are platform-specific. We will use ``wheel`` as a packaging format to solve this issue.

Final solution is available on my [github page](https://github.com/art-vasilyev/demo-source-protect).

## Step 1. Sample application

Create a virtual environment for experiments:
```
$ virtualenv .venv --python=python3.6
$ source .venv/bin/activate
```

Create a simple hello-world application with the following structure:
```
.
├── app
|   ├── __init__.py
|   ├── core.py
│   └── main.py
└── setup.py
```

Our application is in the ``app`` directory. ``main.py`` is the entry point:
```python
from app.core import greeting

if __name__ == '__main__':
	greeting()
```

``core.py`` contain application logic, that we want to protect:
```python
def greeting():
	print("Hello world")
```

``setup.py`` is required to pack our application into a package. Without compilation, it can look like this:
```python
# coding: utf-8
import os

from setuptools import setup, find_packages

setup(
    name='app',
    version='0.1.0',
    packages=find_packages()
)
```

Let's build the package and look at what goes inside it:
```
$ python setup.py sdist
$ tar -xzf dist/app-0.1.0.tar.gz
$ tree app-0.1.0
app-0.1.0
├── app
│   ├── core.py
│   ├── __init__.py
│   └── main.py
├── app.egg-info
│   ├── dependency_links.txt
│   ├── PKG-INFO
│   ├── SOURCES.txt
│   └── top_level.txt
├── PKG-INFO
├── setup.cfg
└── setup.py

2 directories, 10 files

```

As you see, the package contains our package with py files. At the next step, we are going to compile python files.

## Step 2. Compilation

We need to install Cython to translate and compile python files:

```
$ pip install Cython
```

Let's update ``setup.py`` to add compilation:

```python
# coding: utf-8
import os

from setuptools import setup, find_packages

from Cython.Build import cythonize


EXCLUDE_FILES = [
    'app/main.py'
]


def get_ext_paths(root_dir, exclude_files):
    """get filepaths for compilation"""
    paths = []

    for root, dirs, files in os.walk(root_dir):
        for filename in files:
            if os.path.splitext(filename)[1] != '.py':
                continue

            file_path = os.path.join(root, filename)
            if file_path in exclude_files:
                continue

            paths.append(file_path)
    return paths


setup(
    name='app',
    version='0.1.0',
    packages=find_packages(),
    ext_modules=cythonize(
        get_ext_paths('app', EXCLUDE_FILES),
        compiler_directives={'language_level': 3}
    )
)

```
``get_ext_paths`` function returns a list of files that needs compilation. ``EXCLUDE_FILES`` is a list of files that we will include as-is. We exclude ``main.py`` from compilation because it is the application entry point.

Let's compile the application:
```
$ python setup.py build_ext --inplace
tree app
app
├── core.c
├── core.cpython-36m-x86_64-linux-gnu.so
├── core.py
├── __init__.c
├── __init__.cpython-36m-x86_64-linux-gnu.so
├── __init__.py
└── main.py

0 directories, 7 files
```

As you see, there are files ending with ``cpython-36m-x86_64-linux-gnu.so``. These extensions are platform-specific, and they won't run on another platform, so we need to create a separate package for each target platform.

## Step 3. Packaging

We will use wheel packaging format because wheels can contain platform information in package names. Let's try to build the application package and extract:
```
$ python setup.py bdist_wheel
$ unzip dist/app-0.1.0-cp36-cp36m-linux_x86_64.whl -d dist/app
$ tree dist/app
dist/app
├── app
│   ├── core.cpython-36m-x86_64-linux-gnu.so
│   ├── core.py
│   ├── __init__.cpython-36m-x86_64-linux-gnu.so
│   ├── __init__.py
│   └── main.py
└── app-0.1.0.dist-info
    ├── METADATA
    ├── RECORD
    ├── top_level.txt
    └── WHEEL

2 directories, 9 files
``` 

Extracted package contains compiled extensions, but there are also source files, and this is not what we wanted. The workaround for this is to override ``setuptools`` ``build_py``. ``build_py`` is called from ``bdist_wheel`` command and is responsible for collecting package files. Our custom command will filter ``.py`` files if there are compiled extensions with the same name.

Final version of ``setup.py``:
```python
# coding: utf-8
import os
import fnmatch
import sysconfig

from setuptools import setup, find_packages
from setuptools.command.build_py import build_py as _build_py

from Cython.Build import cythonize


EXCLUDE_FILES = [
    'app/main.py'
]


def get_ext_paths(root_dir, exclude_files):
    """get filepaths for compilation"""
    paths = []

    for root, dirs, files in os.walk(root_dir):
        for filename in files:
            if os.path.splitext(filename)[1] != '.py':
                continue

            file_path = os.path.join(root, filename)
            if file_path in exclude_files:
                continue

            paths.append(file_path)
    return paths


# noinspection PyPep8Naming
class build_py(_build_py):

    def find_package_modules(self, package, package_dir):
        ext_suffix = sysconfig.get_config_var('EXT_SUFFIX')
        modules = super().find_package_modules(package, package_dir)
        filtered_modules = []
        for (pkg, mod, filepath) in modules:
            if os.path.exists(filepath.replace('.py', ext_suffix)):
                continue
            filtered_modules.append((pkg, mod, filepath, ))
        return filtered_modules


setup(
    name='app',
    version='0.1.0',
    packages=find_packages(),
    ext_modules=cythonize(
        get_ext_paths('app', EXCLUDE_FILES),
        compiler_directives={'language_level': 3}
    ),
    cmdclass={
        'build_py': build_py
    }
)

```

**Note**: We used the ``cmdclass`` argument to override default commands. This trick won't work if you use ``pbr`` for collecting package information, in that case, place custom ``build_py`` in the separate module and put the reference to the ``setup.cfg`` configuration file.

Build the package:
```
$ rm -rf app.egg-info dist build
$ python setup.py bdist_wheel
```

Now we have a wheel package named ``app-0.1.0-cp36-cp36m-linux_x86_64.whl``. Let's check it and try to run:
```
$ unzip dist/app-0.1.0-cp36-cp36m-linux_x86_64.whl -d dist/app
$ tree dist/app
dist/app
├── app
│   ├── core.cpython-36m-x86_64-linux-gnu.so
│   ├── __init__.cpython-36m-x86_64-linux-gnu.so
│   └── main.py
└── app-0.1.0.dist-info
    ├── METADATA
    ├── RECORD
    ├── top_level.txt
    └── WHEEL

2 directories, 7 files
$ cd dist/app
$ python -m app.main
Hello world
```

Finally, it contains no source code and it works. This package is platform-specific, so you need to build packages for every target python version and upload them to PyPI server. When you try to install the package from PyPI, pip will choose package built for an appropriate platform or will fail if the platform doesn't support any of them.

**Note**: Take into account that Python can be built with UCS-2 and UCS-4 option (number of bytes required for unicode characters). In our example `36m` means that this package is built on Python 3.6 with UCS-2.

***

Feel free to contact me on [LinkedIn](https://www.linkedin.com/in/art-vasilyev/) or drop me an [email](mailto:artem.v.vasilyev@gmail.com).
