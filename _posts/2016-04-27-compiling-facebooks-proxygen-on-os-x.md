---
tags: c++
title: Compiling Facebook's Proxygen on OS X
---
The Proxygen library, from Facebook, allows the C++ programmer to program on both ends of the HTTP protocol : client and server. The library runs on Linux and has been used in production by Facebook for a few years before it was open sourced in november 2014. This should be considered a serious option as it delivers incredible performance to process HTTP requests on the server side.

Proxygen's documentation isn't really great and it often seems easiest to just learn by looking at the sample code delivered along with the library. Don't raise your expectations, the aim of this article is not about improving the documentation. In fact, there's also another downside to proxygen's library in my opinion, it doesn't work out of the box on OS X even though it's a FreeBSD system. I have worked on a linux system for a few months to develop software using Proxygen and frustrated by the inferior tooling and the learning curves I had to go through, I decided to seriously get it to compile on OS X and that decision has been a success. Here's the recipe I've used.

## Dependencies
First of all, here's an overview of the dependencies we'll install on OS X 10.11 :
* Boost 1.60
* OpenSSL 1.0.2g
* libevent 2.0.22
* CMake 3.5.2
* glog 0.3.3
* gflags 2.1.2
* libdouble-conversion #56a0457
* Folly #4aa901d
* Wangle #9288f60

There are two dependencies from the linux build that have been completely removed. The first one, libatomic, is required when compiling with gcc. Because we are compiling with clang on the Mac (unless you are not?), the -latomic linker flag can be safely removed. The second one is libcap. This one is specific to linux and it seems to be used to drop privileges after creating sockets. This feature is clearly missing from the build on OS X so unless OS X support becomes official from the Proxygen authors, I strongly advise against hosting anything in production built on OS X. Use this build solely for developing/debugging/profiling purposes.
<p>Also, it is interesting to note that most of the dependencies to build Proxygen are inherited from folly. Once folly's dependencies are satisfied, there are only two libraries left to compile : wangle and then proxygen.</p>

## Boost 1.60
<p>Considering this build is for development purposes, I went with the latest stable Boost release at the current time which is 1.60. The dependency script (deps.sh) bundled with Proxygen for linux seems to require at least Boost 1.51.</p>
<p>When downloading a bundle from boost.org's web site, you'll need to grant execution privileges to the bootstrap script before executing it.</p>
<p>Boost is mostly a header-only library, but there are a few components that require some building. In order to speed up things a bit, we'll only build the subset of components are folly requires before installing everything to the default location (/usr/local/lib for binaries and /usr/local/include for the headers).</p>
```shell
chmod +x bootstrap.sh
./bootstrap.sh
./b2 --with-context --with-filesystem --with-program_options --with-regex --with-system --with-thread -j$(sysctl -n hw.ncpu)
sudo ./b2 install
```

<p>Here, the -j parameter allows us to specify the amount of parallel tasks that can be run. The sysctl command will return the number of logical CPUs on the current hardware in order to build as quickly as possible. This subcommand will be used repeatedly from now on.</p>

## OpenSSL 1.0.2g
<p>It has been a few years that OS X is not shipped with OpenSSL anymore, and even when it last was, it was an outdated version of OpenSSL (0.9.7) forked by Apple a short while before the Heartbleed vulnerability was introduced into the OpenSSL codebase. Thus we'll need to compile our own. Building OpenSSL is pretty straightforward, the only notable things are :</p>
<ul>
<li>the parameter we must supply to the Configure script : darwin64-x86_64-cc</li>
<li>the make depend command that the Configure script suggests we should execute…</li>
<li>… and the capital C used for the Configure script.</li>
</ul>
```shell
curl https://www.openssl.org/source/openssl-1.0.2g.tar.gz -o openssl-1.0.2g.tar.gz
tar -zxf openssl-1.0.2g.tar.gz
pushd openssl-1.0.2g
./Configure darwin64-x86_64-cc
make depend
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## libevent 2.0.22
<p>The libevent library has a dependency on OpenSSL and I have been having a hard time getting it to compile along with the installation of OpenSSL 1.0.2g done a little earlier. That is until I realized the dependency is meant for a library on which folly does not depend : libevent_openssl. The configure script offers a parameter to disable this optional feature and then there's no problem left compiling and installing this library.</p>
```shell
curl https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz -o libevent-2.0.22-stable.tar.gz
tar -zxf libevent-2.0.22-stable.tar.gz
pushd libevent-2.0.22-stable
./configure --disable-openssl
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## CMake 3.5.2
<p>CMake is a cross-platform build system that is required by some dependencies. It is also the latest stable release at the time of this writing. Fortunately, CMake is a breeze to install and you should not be having any trouble here.</p>
```shell
curl https://cmake.org/files/v3.5/cmake-3.5.2.tar.gz -o cmake-3.5.2.tar.gz
tar -zxf cmake-3.5.2.tar.gz
pushd cmake-3.5.2
./bootstrap
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## glog 0.3.3
<p>Google's logging library is another one that is easy and standard to compile and install.</p>
```shell
svn checkout https://google-glog.googlecode.com/svn/trunk/ google-glog
pushd google-glog
./configure
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## gflags 2.1.2
<p>Google's command line parsing library is, once again, standard and easy to compile and install.</p>
```shell
svn checkout https://google-gflags.googlecode.com/svn/trunk/ google-gflags
pushd google-gflags
./configure
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## libdouble-conversion #56a0457
<p>This is the first library we'll build using CMake so the steps are a little different. First, we create an empty directory where CMake can generate the files derived from the codebase. Then, we'll call CMake with a single parameter to specify a Release build. By default, CMake will generate Makefiles so that everything afterwards boils back down to what we're used to. Please note that I've included the revision number against which I have been writing this article.</p>
```shell
git clone https://github.com/floitsch/double-conversion.git double-conversion
pushd double-conversion
mkdir Release
cd Release
cmake -DCMAKE_BUILD_TYPE=Release ../
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## Folly #4aa901d
<p>Now that we've built all of folly's dependencies, we can now build the library itself. I believe there are some dependencies which are not bundled with OS X that were already installed on my machine that might still be missing for the reader. Namely, autotools. If that is the case, then please contact me with the details so that this guide can be improved for others. When you're done installing those dependencies, here's how to build folly.</p>
```shell
git clone https://github.com/facebook/folly
pushd folly/folly
autoreconf -ivf
LDFLAGS=&quot;-L/usr/local/ssl/lib -L/usr/local/lib&quot; CXXFLAGS=&quot;-I/usr/local/ssl/include -I/usr/local/include&quot; ./configure
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```
<p>Autoreconf will generate the configure script which is missing from folly's source. Then, some extra linker and compiler flags are required when configuring the build system so that the glog, gflags and OpenSSL libraries can be found. Finally, make as usual.</p>

## Wangle #9288f60
<p>Wangle is a library that provides client/server abstractions to ease the development of various services. Just like Folly, this library is aimed at Linux and does not guarantee OS X support. It turns out the CMakeLists.txt file used by CMake to generate the build system does not work out of the box. There are four problems with it :</p>
<ol>
<li>CMake cannot find the glog and gflags libraries [<a href="https://github.com/Dalzhim/wangle/commit/b54d2dce457299e9c78086f3e21fedc6ab288fea" target="_blank">Fixed</a>]</li>
<li>CMake is trying to find the libatomic library which we will not require on OS X [<a href="https://github.com/Dalzhim/wangle/commit/d63b7e3f8c816180512f599edec5389556fbfc65" target="_blank">Fixed</a>]</li>
<li>Even when specifying a custom path for OpenSSL, the header search paths are not adjusted accordingly [<a href="https://github.com/Dalzhim/wangle/commit/6e611173ee1bae30c0a7afe2926b1e7ee346a7dc" target="_blank">Fixed</a>]</li>
<li>The compilation flags specify C++0x but some pieces of code rely on a C++14 feature : deduced function return types [<a href="https://github.com/Dalzhim/wangle/commit/3581b3054232bcb38f8820e0f96cbcaef569a11d" target="_blank">Fixed</a>]</li>
</ol>
```shell
git clone https://github.com/facebook/wangle.git
cd wangle/wangle
mkdir Release
cd Release
cmake -DCMAKE_BUILD_TYPE=Release -DOPENSSL_ROOT_DIR=/usr/local/ssl ..
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## Proxygen #c8abbee
<p>Proxygen is the HTTP library we wanted to toy with all along! Just like both Folly and Wangle, it is aimed at Linux and does not guarantee OS X support. As a matter of fact, it does not work out of the box and requires some modifications in order to compile.</p>
<ol>
<li>Autoreconf fails to generate the configure script properly because of an unknown token : proxygen-config.h [<a href="https://github.com/Dalzhim/proxygen/commit/4008de5f45e44a5f7b21b0b16b0070bc615a25cb" target="_blank">Fixed</a>]</li>
<li>Autoreconf whines about a missing config : ACLOCAL_AMFLAGS [<a href="https://github.com/Dalzhim/proxygen/commit/bb7b9c57a99b0b5f6ec6666482665a571b2059aa" target="_blank">Fixed</a>]</li>
<li>An include seems to be missing for the gflags library's headers [<a href="https://github.com/Dalzhim/proxygen/commit/df1e178750bbd4621bae6cba3bef208a1a7ebc04" target="_blank">Fixed</a>]</li>
<li>The lib/ssl, lib/http and lib/services sublibraries fail to build properly [<a href="https://github.com/Dalzhim/proxygen/commit/9c79646d2250dfbd7dc1b742ce5684a3997e1abd" target="_blank">Fixed</a>]</li>
<li>The tests fail to build properly because they rely on wget to download gmock 1.7.2</li>
</ol>
<p>Finally, trying to build fails a little later when trying to build the tests because wget does not exist. All that is required is to download gmock at the path proxygen/lib/test, unzip it there and build it (without installing) and Proxygen can finally compile and install properly.</p>
```shell
git clone https://github.com/facebook/proxygen.git
pushd proxygen/proxygen
pushd lib/test
curl https://googlemock.googlecode.com/files/gmock-1.7.0.zip -o gmock-1.7.0.zip
unzip gmock-1.7.0.zip
pushd gmock-1.7.0
./configure
make -j $(sysctl -n hw.ncpu)
popd
popd
autoreconf -ivf
LDFLAGS=&quot;-L/usr/local/lib -L/usr/local/ssl/lib&quot; CPPFLAGS=&quot;-I/usr/local/include -I/usr/local/ssl/include&quot; ./configure
make -j $(sysctl -n hw.ncpu)
sudo make install
popd
```

## Conclusion
<p>Compiling Proxygen on OS X has been quite a challenge even though there were not too many modifications required. The main difficulty is that there is such a diversity of build systems that it takes a lot of time to get used to each one of them (makefiles, autotools, cmake). I will be trying to get pull requests incorporated into the main branch with those fixes and I hope to learn some more along the way. Once again, please contact me if you have anything to contribute concerning the installation of autotools. I'm pretty sure it isn't on OS X out of the box and I believe it wasn't exactly straightforward to install either when not using homebrew of macports. I hope this will help OS X support with Proxygen to move forward in the next weeks.</p>
