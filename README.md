# Multithreaded xxhsum (and hashing in general)

This is a version of xxhsum that adds a flag to operate in multithreaded
mode.

## Basic rationale

The rationale for this may require some explanation because its
practical and theoretical details are not immediately obvious.

Checking the integrity of data is of course, in principle, an extremely
parallelizable task; computing one hash does not interfere with
computing a separate hash.  The relative speeds of modern hash
algorithms, disks, processors, and filesystems have led to bottlenecks
that some users may find counterintuitive.  In particular, running a
hashing tool as fast as xxhsum against a typical filesystem (find . |
xargs xxhsum) is likely to perform rather poorly in the real world
because of the need for serial blocking on filesystem reads, even just
to traverse the filesystem's directory structure.  Particularly on
modern SSDs, which even at the consumer grade now can have a throughput
of 7 or 8 GB/sec, directory traversal, especially for small files, causes
significant performance degradation when run within a single thread.

Parallelization provides significant improvements on this type of setup,
as sophisticated system administrators have come over the last few years
to recognize.  For example, on Windows, no serious administrator would
copy a large set of files with Explorer rather than with robocopy /mt in
any situation where performance was even minimally important.

To be clear, the goal of parallelization here is a bit unusual; its
purpose is mainly to avoid serial blocking on filesystem operations.  To
say it more simply, the goal is not to saturate the processor (which is
unlikely, when any I/O is involved, with a hashing algorithm as speedy
as xxh) but to saturate the disk, network, or other I/O source.

As a simple casual example, on even a relatively small test filesystem
typical of user data (with about 200,000 files ranging from a few bytes
to several gigabytes each) and stored on a SSD connected over USB, all
running on low-end consumer hardware, my multithreaded version ran at
least twice as fast as the standard version in real-clock time using a
few threads.  The difference, again, appears to come not mainly from
parallelizing the hashing but from avoiding waiting for sequential
blocking filesystem operations to complete.  The speedup is
significantly greater on NVMe drives with higher throughput; it appears
to be possible to use the disks' full read speeds even on a filesystem
that contains many small files.

## Design

Under the traditional Unix philosophy, it may seem better to avoid
handling parallelization within individual tools and instead to address
the same problem at a more general level of abstraction.  For example,

    find . -type f -print0 | xargs -0 -P${nTHREADS} stdbuf -oL xxh128sum

is an adequate way to *produce* file hashes efficiently.  xargs -P is a
typical means of parallelizing workflows by means of multiple processes;
stdbuf here avoids interleaving the hash utility's output to a
checkfile.

The difficulty with the typical workflow of producing and verifying
hashes, however, is that it is inconvenient to parallelize hash checking
in a clean, general-purpose script.  The main possibilities are to use
split, followed by either a separate pipe to xargs or, in versions of
split that support it, using --filter to a command that is aware of its
multi-process environment and (for example) immediately puts itself into
the background.  Either solution is messy for interactive workflows,
requires scratch space on a filesystem or a separate script, etc.

Supporting multithreading in the hashing tool itself solves that 
problem, and it has two other advantages: (1) multithreading is more 
efficient than separate processes produced by xargs, which may matter in 
workflows using hash algorithms as efficient as XXH; (2) highlighting 
the benefits of parallel solutions is likely to be useful in showing 
users and administrators how quickly data can be checked for integrity 
on modern hardware and with modern hashing algorithms.

Tools like GNU parallel are an alternative, but given the high 
performance of XXH, internal multithreading is probably advantageous; 
GNU parallel may also not be available in all environments that use 
xxhsum.  One way to think about the problem is to consider a checkfile 
as a kind of little program to run and the checksum utility as an 
interpreter that runs it.  An internal implementation, too, can make 
sound default choices that suit the particular task; for example, in 
checkfiles covering a typical filesystem, large files to be checked are 
likely to be located near each other (e.g., in a directory containing 
multimedia or databases), which suggests that line-by-line 
parallelization is a better default than coarser chunking of the 
checkfile.

In view of these considerations, the current version of my code supports
multithreaded checking but not multithreaded hashing.  It would not be
difficult to support multithreaded hashing too, but I haven't bothered
to do that yet because, as I've explained above, there's much less of a
need for it.

## Implementation

I've tried to keep the implementation relatively simple and to avoid
changing the existing code as much as possible; I've also tried to format
the code in the existing style, even when that differs from my style.

The result is a little quick-and-dirty and is only the result of a few
hours of work -- I'm not always sure I struck the balance right between
simplicity and readability -- but it's at worst a proof of concept.  I
don't think what I've done will be too difficult to read or maintain.

I'm a law professor who (despite a deep background in development)
is a bit out of date, so I haven't attempted to integrate well into the
existing code's cross-platform support.  I don't think anything I've done
is idiosyncratic to a particular architecture, and at the very least any
incompatibilities should be relatively easy to address.

I also haven't fully integrated internally with libstdbuf yet, which is
probably the easiest way to avoid interleaved output.  (That could be
done manually with mutexes, too, if that seems cleaner.)  Anyway, doing
that should not be difficult either, and it shouldn't be relevant at all
when xxhsum is run with -cq.

-- Shawn Bayern
sbayern@law.fsu.edu
