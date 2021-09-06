---
title: Single-file Python CLI Apps
date: 2021-08-05 13:04:22
tags:
    - python
    - tips-n-tricks
    - cli
---

When writing CLI applications I often find myself deciding on whether to use Go or Python - the former creating single-file (binary) executables (handy for distribution), and the latter being easy to develop. However, I recently discovered [zipapp](https://docs.python.org/3/library/zipapp.html); A part of the Python standard library that takes your Python files and shoves them into a single (zipped) file that you can execute - giving one of the benefits Go binaries have (with them otherwise being quite significantly different).

## How to use Zipapp

Zipapp is a Python stdlib module that takes your files in a package and zips 'em to a single file Python can interpret. Here's an example of a useless application packaged with zipapp:

```py greet/__main__.py
import sys

from greeter import greet

greet(sys.argv[1])
```

```py greet/greeter.py
def greet(name):
    print("Hello, " + name)
```

With this you should be able to run:

```sh
$ python3 greet world
Hello, world
```

Now to package the `greet` directory into a single binary:

```sh
$ mkdir bin
$ python3 -m zipapp greet # create the greet.pyz file
$ echo '#!/usr/bin/env python3' | cat - greet.pyz > bin/greet # add shebang to make the file executable
$ chmod u+x bin/greet # allow the file to execute itself
$ ./bin/greet world
Hello, world
```

### Including pip modules

Ok let's complicate this by using an external pip module. Zipapp won't automatically include referenced pip modules but there's a way to get around it; by installing modules directly to you app directory; Let's make our app even more useless with `cowsay`:

```txt requirements.txt
cowsay==4.0
```

```py greet/greeter.py
from cowsay import cow


def greet(string):
    cow("Hello, " + string)
```

Now to package our app:


```
$ pip3 install -r requirements.txt --target greet
$ python3 -m zipapp greet
$ echo '#!/usr/bin/env python3' | cat - greet.pyz > bin/greet
$ chmod u+x bin/greet
$ ./bin/greet world
  ____________
| Hello, world |
  ============
            \
             \
               ^__^
               (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||
```

Beautiful!

### Avoiding mixing your application code with modules

I would suggest you create the build from another directory than your application code to not get hell when creating your `.gitignore`. A simple build script could look something like this

```sh build.sh
#!/usr/bin/env sh
set -xeuo pipefail # never forget this in your shell scripts ;)

rm -rf target/*
mkdir -p bin target tmp
cp -r greet/ target
pip3 install -r requirements.txt --target target
python3 -m zipapp target -o tmp/greet.pyz
echo '#!/usr/bin/env python3' | cat - tmp/greet.pyz > bin/greet
chmod u+x bin/greet
```

```.gitignore .gitignore
/bin
/target
/tmp
```

For a full example visit [github.com/palmenhq/python-zipapp-example](https://github.com/palmenhq/python-zipapp-example)

## Conclusion

This is a really nifty way of building Python cli tools imo. However, note that it'll use the host's Python interpreter, so you can't run it if Python is not installed. However, most systems have python3 installed so this should be more of an advantage than a problem, as that can keep bundle file size down.

When building cli applications, my go-to option will from now on be Python unless I want the raw speed Go offers.
