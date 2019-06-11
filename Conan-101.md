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

### Search Server(s)

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

To create a new package from a template users can run `conan new Library/Version -t`, alternatively users can write a custom conanfile.py conforming to the following rules: https://docs.conan.io/en/latest/reference/conanfile.html#conanfile-reference 

### Settings

Each `conanfile.py` contains a `settings` attribute:

```
settings = ["os", "compiler", "build_type"]
```

The settings field defines the configuration of the different binary packages. 

Conan will necessarily treat a change in settings as a separate binary package. I.e. in the sample above, a change in `build_type` would cause a new binary package to be made.

Settings can be either provided to the package manager upon `conan install` using the `-s settings.name=value` paradigm, or otherwise the environment defaults are used from the `settings.txt`:

```

$ conan profile show default

Configuration for profile default:

[settings]
os=Linux
os_build=Linux
arch=x86_64
arch_build=x86_64
compiler=gcc
compiler.version=7
compiler.libcxx=libstdc++11
build_type=Release
[options]
[build_requires]
[env]
```

Settings are configurable. You can edit, add, remove settings or subsettings in your settings.yml file. See the settings.yml reference: https://docs.conan.io/en/latest/reference/config_files/settings.yml.html#settings-yml


### Creating

The `conan create` commands build a recipe into a binary package. If a `test_package` folder exists with a `conanfile.py` then it will automatically run during the `create` process.

```
conan new Hello/0.1 -t
conan create . demo/testing
...
[ 50%] Building CXX object CMakeFiles/example.dir/example.cpp.o
[100%] Linking CXX executable bin/example
[100%] Built target example
Hello/0.1@demo/testing (test package): Running test()
Hello World!

```

Under the hood, the `create` command is equivalent to:

```
conan export . demo/testing
conan install Hello/0.1@demo/testing --build=Hello
# package is created now, use test to test it
conan test test_package Hello/0.1@demo/testing

```

Users can opt to get finer granularity of control by manually calling the underlying `conan build`, `conan export-pkg` etc commands but often this is unnecessary.

More details on the package creation process can be found here: https://docs.conan.io/en/latest/creating_packages/understand_packaging.html

`conan create` and `conan install` take the same command line options, so it is possible to create and test packages for different configurations:

```
conan create . demo/testing -s build_type=Debug
conan create . demo/testing -s build_type=Release
conan create . demo/testing -s build_type=Release -s arch=x64
```

### Testing

`conan test` allows a `test_package/conanfile.py` to be tested. 

The commands installs dependencies, calls `conan build` and executes the `test()` method.

An example `test()` method is as follows:

```
    def test(self):
        if not tools.cross_building(self.settings):
            os.chdir("bin")
            self.run(".%sexample" % os.sep)
```

### Uploading

Packages can be uploaded to both public and private conan servers.

Recipe authors can upload a `recipe` and optionally, the corresponding `binary packages` for the relevant configurations (see #Settings).

The `conan upload` tool supports the following options:
- [none]: uploads the package recipe ONLY
- `--all` uploads the package recipe plus all binary packages
- `--packages` uploads the explicitly listed packages 
- `--query` uploads packages matching the user provided query

NB: If a package exists, it will not be re-uploaded unless explicitly forced.

## Build Policy

Authors can specify a build policy controlling whether clients can download pre-packaged binaries, and specify if local building of packages is permitted.

Clients can call `conan install --build` indicating their intent on where the package is built, and depending on the author's restrictions this may be honoured.

From https://docs.conan.io/en/latest/mastering/policies.html:  _By default, conan install command will search for a binary package (corresponding to our settings and defined options) in a remote. If it’s not present the install command will fail._

Users can use the `--build` option to change the default `conan install` behaviour:

    --`build some_package` will build only “some_package”.
    --`build missing` will build only the missing requires.
    --`build` will build all requirements from sources.
    --`build outdated` will try to build from code if the binary is not built with the current recipe or when missing binary package.
    --`build cascade` will build from code all the nodes with some dependency being built (for any reason). Can be used together with any other build policy. Useful to make sure that any new change introduced in a dependency is incorporated by building again the package.
    --`build pattern*` will build only the packages with the reference starting with “pattern”.

```
 class PocoTimerConan(ConanFile):
     settings = "os", "compiler", "build_type", "arch"
     requires = "Poco/1.7.8p3@pocoproject/stable" # comma-separated list of requirements
     generators = "cmake"
     default_options = {"Poco:shared": True, "OpenSSL:shared": True}
     build_policy = "always"
```

