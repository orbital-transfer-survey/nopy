# nopy

Install and run python dependencies in node.js project.

This project provides an alternative to a python virtual environment for node.js projects. It aims to make pip work more like npm. It supports pythons 2 and 3.

## Installation

First, nopy's commands are wrappers for python. Python must be installed already and on PATH.

Suppose we create an empty project with a package.json file:
```
/home/al$ mkdir myproject
/home/al$ cd myproject
/home/al/myproject$ npm init -y
Wrote to /home/al/myproject/package.json:

{
  "name": "myproject",
  ...
}
```

Then, to install nopy as a local dependency:
```
$ npm install nopy --save
...
```

## Using npip to install python packages

To install a python package in a node.js project:
```
$ node_modules/.bin/npip install toposort
Downloading/unpacking toposort
  Downloading toposort-1.5-py2.py3-none-any.whl
Installing collected packages: toposort
Successfully installed toposort
Cleaning up...
```

This creates a python_modules subdirectory, which contains installed python packages. Other pip commands are supported by npip:
```
$ node_modules/.bin/npip freeze
toposort==1.5
```

## Using nopy to run python

Invoke python using the nopy wrapper and it will find the python packages installed in python_modules.
```
$ node_modules/.bin/nopy
Python 3.6.0
Type "help", "copyright", "credits" or "license" for more information.
>>> from toposort import toposort
>>> list(toposort({2: {11},
...                9: {11, 8, 10},
...                10: {11, 3},
...                11: {7, 5},
...                8: {7, 3},
...               }))
[{3, 5, 7}, {8, 11}, {2, 10}, {9}]
```

Here, when python was invoked interactively, it looked in the python_modules subdirectory to find the locally installed toposort dependency.

To direct python to invoke a python program at a particular path:
```
$ cat - >some_program.py
from toposort import toposort
print(list(toposort({2: {11},
                     9: {11, 8, 10},
                     10: {11, 3},
                     11: {7, 5},
                     8: {7, 3},
                    }))
$ node_modules/.bin/nopy some_program.py
[{3, 5, 7}, {8, 11}, {2, 10}, {9}]
```

Python interpreter command line options and arguments that will be visible to the python program via `sys.argv` are passed through. To invoke a module, pass the `-m` option through to python.
```
$ node_modules/.bin/nopy -m site --user-base
/home/al/myproject/python_modules
```

Similarly, to pass python code on the command line, use the `-c` option:
```
$ node_modules/.bin/nopy -c 'import sys; x=float(sys.argv[1]); print x*x' 3
9.0
```

What magic does nopy use to find the `python_modules` directory? When nopy is invoked with the path to a python program to run, it first inspects the directory containing that program to see if it has a `package.json` file. If it's there, then it will try to use a `python_modules` directory alongside it. If `package.json` isn't found, it works up the directory hierarchy until it reaches the root. If a `package.json` file is found anywhere along the way, it tries to use a sibling `python_modules` directory. This resembles the mechanism that node.js uses to resolve modules.

If nopy is invoked with no path to a program to run, e.g. if it is invoked interactively or with `-m` or `-c`, the process is quite similar. Instead of starting at a directory given by the path, it starts in the current working directory.

## Documenting python dependencies

Python dependencies should be listed in the package.json file in the same way as for node.js dependencies, except the section is called "pythonDependencies" rather than "dependencies". Here is an example package.json file:
```
{
  "name": "myproject",
  ...
  "dependencies": {
    "nopy": "^1.0.0"
  },
  "pythonDependencies": {
    "toposort": ">=1.5"
  }
}
```

A version may be specified for python dependencies, which follows the same format as used by [pip requirement specifiers](https://www.python.org/dev/peps/pep-0508/). The version may be an empty string.

This serves not only to document the python dependencies; if `npip install` is invoked with no requirements, its default action is to install all the python dependencies given in package.json.
```
$ node_modules/.bin/npip install
Requirement already satisfied: toposort
```

It might be desirable to install the python dependencies at the same time as the other dependencies. This can be accomplished with an npm install script. While we're at it, it would be convenient to be able to invoke nopy and npip through npm scripts too.
```
{
  "name": "myproject",
  ...
  "scripts": {
    "install": "npip install",
    "npip": "npip",
    "nopy": "nopy"
  },
  "dependencies": {
    "nopy": "^1.0.0"
  },
  "pythonDependencies": {
    "toposort": ">=1.5"
  }
}
```

Now both the node.js and python dependencies can be installed with a single command:
```
$ npm install
```

And now if we want to install another python dependency, we can use an npm script:
```
$ npm run npip -- install pytz
```

## Using nopenv to run programs in the local python environment

The nopenv tool is similar to nopy. Rather than passing python source files to python for execution, it is used to run executable programs in the context of an environment that includes the locally installed python modules. For example, it is useful for running executable scripts installed by some python packages.

As one example, let's say the [alembic](https://pypi.python.org/pypi/alembic) python package was installed locally. To start a database migration:
```
$ node_modules/.bin/nopenv alembic revision -m "create inventory table"
Generating /path/to/yourproject/alembic/versions/1975ea83b712_create_invent
```

Alternatively, it's less typing to make an npm script in package.json to quickly invoke the script:
```
{
  "name": "myproject",
  ...
  "scripts": {
    "alembic": "nopenv alembic",
    "install": "npip install"
  },
  "dependencies": {
    "nopy": ""
  },
  "pythonDependencies": {
    "alembic": ""
  }
}
```

Then to run alembic using the npm script:
```
$ npm run alembic -- revision -m "create inventory table"
Generating /path/to/yourproject/alembic/versions/1975ea83b712_create_invent
```

## Global installation

nopy can be installed globally so that nopy and npip are on PATH. For example:
```
$ npm install nopy -g
...
$ npip install <package name>
...
```

However, I prefer to make do with only a local installation. npm scripts are often a solution when repeatedly typing 'node_modules/.bin' becomes cumbersome.

## Gotchas

When a python program is invoked with nopy, its PYTHONPATH environment variable is modified to point to the node.js project directory, i.e. he one containing package.json. This allows modules to be imported relative to the project directory itself, not just from python packages that have been installed python_modules. In order to isolate the project's python dependencies, any previous value of PYTHONPATH is overridden.

Behind the scene, nopy uses python's [per user site-packages directory](https://www.python.org/dev/peps/pep-0370/) mechanism. Specifically, when python or pip are invoked indirectly by way of the nopy or npip wrappers, the `PYTHONUSERBASE` environment variable is modified to reference the python_modules directory contained in the node.js project, the one alongside package.json. This tells python to look there for installed pythonpackages. Additionally, pip is invoked with the `--user` option, which causes it to install packages in python_modules.

A caveat though is, because python's per user site-packages directory is overridden, any other such directory, such as one residing in the user's home directory, is no longer be visible to python. If the goal is to isolate the project's python dependencies within the project, this is a feature. It's definitely a potential gotcha, though!

npip ships with a version of pip that it can install locally into the node.js project site directory. It does this automatically when the first python package is installed but only if there is not a more recent version of pip installed globally. You can see which version of pip is used by npip using the --version option.
```
$ node_modules/.bin/npip --version
pip 9.0.1 from /home/al/myproject/python_modules/lib/python2.7/site-packages (python 2.7)
```

The oldest versions of python that nopy has been tested with are 2.7.6 and 3.6.0.

## TODO

These are some things that might happen in the future.
* There is a node.js API to programatically spawn python programs. This could be documented and made "official".
* Integrate JSON-RPC or other protocol over channel between node.js process and python child processes.
