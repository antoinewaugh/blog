# ABI Part II Still Learning

As mentioned [in this post](https://github.com/antoinewaugh/blog/blob/master/ABI-Ill-learn-about-that-thing-later.md) given I am devloping on fedora to deploy to centos I need to add the `-D_GLIBCXX_USE_CXX11_ABI=0` when compiling my project.

Recently, I needed to add a dependency [cryptopp](https://www.cryptopp.com/).

Following their instructions, the build process was as follows:

```
$ wget https://www.cryptopp.com/cryptopp700.zip
$ unzip cryptopp700.zip
$ make
$ sudo make install
```

So far so good.

# Linking the libcryptocpp.a

After building, I simply addded the link command to my CMakelist.txt

```
target_link_libraries(mytarget libcryptocpp.a)
```

# Undefined symbols

After linking and rebuilding, the following link error occurred

```
$ cmake .. && make
$ ~/include/cryptopp/filters.h:1366: undefined reference to `CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::string const&, bool)
```

The weird thing was, when running `nm` I was able to see the listed symbol in the libcryptopp.a

```
nm ../lib/libcryptopp.a | c++filt | grep CryptoPP::BufferedTransformation::TransferAllTo2
0000000000002c80 T CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)
                 U CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)
                 U CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)
                 U CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)
                 U CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)
                 U CryptoPP::BufferedTransformation::TransferAllTo2(CryptoPP::BufferedTransformation&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, bool)

```

# Culprit

The culprit was (yet again) the `-D_GLIBCXX_USE_CXX11_ABI=0` flag used to build the project which was linking against `libcryptopp.a`.

Effectively the std::string from c++11 onwards is not compliant with ABI=0, and therefore the building of the libcryptopp should have also included the ABI flag.

# The fix

Rebuild the libcryptopp with the necessary flags

```
$ wget https://www.cryptopp.com/cryptopp700.zip
$ unzip cryptopp700.zip
$ export CXXFLAGS=-D_GLIBCXX_USE_CXX11_ABI=0 # <--- added 
$ make
$ sudo make install
```

# Conclusion

Remember that if ABI=0 is being used to compile, then all subsequent libraries (static and dynamic) which are linking against the said project also need to have been compiled with the same flag.
