# Use GCC to PGO Clang

Clang is faster to compile than GCC, but It Is Said that GCC PGOs better than Clang.
So let's build a monster.

## Methodology

I'm building a DEBUG version of this cset:

~~~
commit 6fa252d9b66aa88294aeef1f97160411f2f6f85f
Merge: bd1a9cc986c6 1688c0df862d
Author: Ciure Andrei <aciure@mozilla.com>
Date:   Fri Mar 30 01:06:18 2018 +0300

    Merge inbound to mozilla-central. a=merge
~~~

All build times are for `./mach build` after running `./mach configure` untimed beforehand.

The machine is a 2xE5-2670, so 2x2x8=32 hardware threads at 2.60GHz base.
I'm running everything with the default of -j32.
GCC version: 7.3.1
LLVM version: 6.0.0

I only run one trial of each config, but the error bars on builds seem to be less than
+/-5s, so I'm not too concerned.

## Install GCC

There's no need to build GCC yourself.

## Clone and checkout

~~~
git clone https://git.llvm.org/git/llvm.git/
git checkout origin/release_60

cd llvm/tools
git clone https://git.llvm.org/git/clang.git/
git checkout origin/release_60

cd ../..
~~~


## Build with -fprofile-generate

~~~
cd llvm-obj-inst

prof="-fprofile-generate=`pwd`/fprofiles"
flags="-march=native -O3 $prof"
CFLAGS="$flags" CXXFLAGS="$flags" LDFLAGS="$prof" cmake -G Ninja -DCMAKE_BUILD_TYPE=Release ../llvm

cmake --build .
~~~

This takes a while even at -j32:

~~~
real    14m36.881s
user    433m33.101s
sys     17m14.185s
~~~


## Compile the thing you care about

We add these to our gecko .mozconfig:

~~~
export CC=/path/to/llvm-obj-inst/bin/clang
export CXX=/path/to/llvm-obj-inst/bin/clang++
ac_add_options --disable-warnings-as-errors
~~~

Optional:

~~~
ac_add_options --disable-tests
~~~

`./mach build`: 18m09s (+6m50s from non-pgo)


## Build with -fprofile-use

~~~
cd pgo-llvm

prof="-fprofile-use=`pwd`/fprofiles"
flags="-march=native -O3 $prof"
CFLAGS="$flags" CXXFLAGS="$flags" LDFLAGS="$prof" cmake -G Ninja -DCMAKE_BUILD_TYPE=Release ../llvm

cmake --build .
~~~

This was faster for me:

~~~
real    10m58.746s
user    327m20.485s
sys     14m47.751s
~~~


## Switch over!

~~~
export CC=/path/to/pgo-llvm/bin/clang
export CXX=/path/to/pgo-llvm/bin/clang++
ac_add_options --disable-warnings-as-errors
~~~

`./mach build`: 10m30s (vs 12m24s for my system clang)
Note that at these times, linking dominates the build time. (about 5m alone)
It looks like lld defaults to using --threads, but we do spend most of the time pegging a
single core.
-30s isn't really worth it at 10m total, but might be more attractive if we could get link
time down.


## Establish baseline build times (optional)

To take a baseline, it's useful to see what we'd get without PGO or other options.

~~~
CFLAGS="$flags" CXXFLAGS="$flags" cmake -G Ninja -DCMAKE_BUILD_TYPE=Release ../llvm
cmake --build .
~~~

- Arch Linux `clang` 6.0.0-1: 12m24s
- `flags=''`: 12m00s (-24s from system binary)
- `flags='-O3'`: 12m02s (+2s from no-flags)
- `flags='-march=native'`: 12m05s (+5s from no-flags)
- `flags='-O3 -march=native'`: 11m55s (-5s from no-flags)
- With PGO: 10m30s (-1m30s from no-flags, -1m54s from system binary)

My guess is that no-flags clang builds with good enough optimization flags that it's not
worth supplying your own flags.
The spread in times here for different flags are really close to the error bars.

## Conclusion

It's probably worth it if you clobber a lot, particularly on a low-core-count machine.

If that's too much work, just building 'default clang' from source seems to help.
