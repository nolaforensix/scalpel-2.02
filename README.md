Note: This is the source code for Scalpel 2.02, a fast
header/footer-based file carver.  While version 1.60 is the most
widely used public release, this version is multithreaded, has a much
better I/O subsystem, and should generally be quite a bit faster.

Neither this release nor 1.60 are being actively developed and if
you're simply doing file carving to retrieve deleted files of common
types, you probably should be using photorec instead, since photorec
has moved beyond the simplistic header/footer-based approach used by
Scalpel 1.60 and 2.02.  On the other hand, if you are developing
patterns to recover esoteric file types, you may find that Scalpel is
still useful.

While there was a plan to integrate this version of Scalpel into the
Sleuthkit, where a different license was required, plans for that fell
through due to some technical issues on their side (C++ and Java are
used there, while Scalpel is written entirely in C).  This version,
which contains no Sleuthkit integration, is released under the GPL.

scalpel3 is now under development and supports not only
header/footer-based file carving, but also much more sophisticated
recovery options and high-performance recovery of fragmented files.  A
public repo for scalpel3 will appear when it's ready.

** If you're having trouble with the libtre dependency when attempting
   to install on Linux, see the DEPENDENCIES discussion near the end
   of this document **

Cheers,

--Golden

Original documentation in README follows.

-------------------------------------------------------------------------

Scalpel is a file carving and indexing application that runs on Linux,
Mac, and Windows systems.  The first version of Scalpel, released in
2005, was based on Foremost 0.69. There have been a number of internal
releases since the last public release, 1.60, primarily to support our
own research.  The newest public release v2.02, has a number of
additional features, including:

o minimum carve sizes.

o multithreading for quicker execution on multicore CPUs.

o asynchronous I/O that allows disk operations to overlap with pattern
matching--this results in a substantial performance improvement.

o regular expression support for headers/footers.

o embedded header/footer matching for better processing of structured
file types that may contain embedded files.

** GPU acceleration and blockmap features were previously offered but are
currently deprecated and removed **

Scalpel performs file carving operations based on patterns that
describe particular file or data fragment "types".  These patterns may
be based on either fixed binary strings or regular expressions.  A
number of default patterns are included in the configuration file
included in the distribution, "scalpel2.conf".  The comments in the
configuration file explain the format of the file carving patterns
supported by Scalpel.

Important note: The default configuration file, "scalpel2.conf", has
many supported file patterns commented out--you must edit this file
before running Scalpel to activate the patterns you want to use.
Resist the urge to simply uncomment all file carving patterns; this
wastes time and will generate a huge number of false positives.
Instead, uncomment only the patterns for the file types you need.

Scalpel options are described in the Scalpel man page, "scalpel2.1".
You may also execute Scalpel w/o any command line arguments to see a
list of options.

NOTE: Compilation is necessary on Unix platforms and on Mac OS X.  For
Windows platforms, a precompiled scalpel2.exe is provided.  If you do
wish to recompile Scalpel on Windows, you'll need a mingw (gcc)
setup. Scalpel will not compile using Visual Studio C compilers.  Note
that our compilation environment for Windows is currently 32-bit; we
haven't tested on the 64-bit version of mingw, but will address this
in the future.

COMPILE INSTRUCTIONS ON SUPPORTED PLATFORMS:

Linux/Mac OS X:    ./configure and then make

Windows:           cd to src directory and then:

	           mingw32-make -f Makefile.win

and enjoy.  If you want to install the binary and man page in a more
permanent place, just copy "scalpel2" (or "scalpel2.exe") and
"scalpel2.1" to appropriate locations, e.g., on Linux, "/usr/local/bin"
and "/usr/local/man/man1", respectively.  On Windows, you'll also need
to copy the pthreads and tre regular expression library dlls into the
same directory as "scalpel2.exe".


OTHER SUPPORTED PLATFORMS

We are not currently supporting Scalpel on Unix variants other than
Linux. Go ahead and try a ./configure and make and see what happens,
but be sure to do thorough testing before using Scalpel in production
work.  If you are interested in supporting a version of Scalpel on an
alternate platform, please contact us.  

LIMITATIONS:

Carving Windows physical and logical device files (e.g.,
\\.\physicaldrive0 or \\.\c:) isn't currently supported because it
requires us to rewrite some portions of Scalpel to use Windows file
I/O functions rather than standard Unix calls.  This may be supported
in a future release, but it's unlikely.

Block map features are disabled in this release as are GPU features.

DEPENDENCIES:

Scalpel uses the POSIX threads library.  On Win32, Scalpel is
distributed with the Pthreads-win32 - POSIX Threads Library for Win32,
which is Copyright(C) 1998 John E. Bossom and Copyright(C) 1999,2005
by Pthreads-win32 contributors.  This library is licensed under the GPL.

For Windows, Scalpel needs the tre regular expression library and is
distributed with tre-0.7.5, which is licensed under the LGPL.

For other platforms, libtre must be installed manually.  It can be
obtained from http://laurikari.net/tre/.  On Linux, make install for
this library installs into /usr/local/lib.  If LD_LIBRARY_PATH doesn't
include /usr/local/lib, put a line like:

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib in your .bashrc
or equivalent.


SUGGESTIONS:

Bug reports, comments, complaints, and feature requests should be
directed to the author via goldenrichard@gmail.com.


Cheers,

--Golden G. Richard III and Vico Marziale.
