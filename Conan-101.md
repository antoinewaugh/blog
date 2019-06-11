# Conan 101

Conan is a c/c++ package manager: https://conan.io/

## Servers

In order to use a library or package a recipe must exist on a server.

Servers can be public (conan.io) or private. The local user cache acts as a server, too.

### List Remote Servers

To see what remote servers are available to the user:

```
$ conan remote list

conan-center: https://conan.bintray.com [Verify SSL: True]
```

### Add Remote Server

To add a remote server:

```
$ conan remote add bincrafters https://api.bintray.com/conan/bincrafters/public-conan

$ conan remote list

conan-center: https://conan.bintray.com [Verify SSL: True]
bincrafters: https://api.bintray.com/conan/bincrafters/public-conan [Verify SSL: True]

```

## Search Server(s)

Users can search for a package locally:

```
$ conan serach "Poco*"

Existing package recipes:

Poco/1.9.0@pocoproject/stable

```

And remotely:

```
conan search "Poco*" --remote=conan-center

Existing package recipes:

Poco/1.7.8p3@pocoproject/stable
Poco/1.7.9@pocoproject/stable
Poco/1.7.9p1@pocoproject/stable
Poco/1.7.9p2@pocoproject/stable
Poco/1.8.0.1@pocoproject/stable
Poco/1.8.0@pocoproject/stable
Poco/1.8.1@pocoproject/stable
Poco/1.9.0@pocoproject/stable

```

## Packaging

If a package is not available on a remote server for a particular library, a conan-recipe needs to be created.

Conan recipes can build from source, or alternatively can package existing binaries.

Package recipes often follow a directory structure as follows:

```
conanfile.py
test_package
  CMakeLists.txt
  conanfile.py
  example.cpp
 ```

The `conanfile.py` provides the build information for the package, whereas the contents of the `test_package` directory verify if a package has been correctly built.

### 
