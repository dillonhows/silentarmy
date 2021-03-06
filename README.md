# SILENTARMY

SILENTARMY is a [Zcash](https://z.cash) miner for Linux written in OpenCL with
multi-GPU support. The
[Stratum](https://github.com/str4d/zips/blob/77-zip-stratum/drafts/str4d-stratum/draft1.rst) protocol is implemented for connecting to mining pools. It runs
best on AMD GPUs but has also been reported to work on other OpenCL devices such
as Xeon Phi, Intel GPUs, and through OpenCL CPU drivers. (Nvidia GPUs are not
currently supported due to an
[issue](https://github.com/mbevand/silentarmy/issues/6).)

After compiling SILENTARMY, list the available OpenCL devices:

```
$ silentarmy --list
```

Start mining with two GPUs (ID 2 and ID 5) on a pool:

```
$ silentarmy --use 2,5 -c stratum+tcp://us1-zcash.flypool.org:3333 -u t1cVviFvgJinQ4w3C2m2CfRxgP5DnHYaoFC
```

When run without options, SILENTARMY mines with the first OpenCL device, using
my donation address, on flypool:

```
$ silentarmy
Connecting to us1-zcash.flypool.org:3333
Stratum server sent us the first job
Mining on 1 device
Total 0.0 sol/s [dev0 0.0] 0 shares
Total 43.9 sol/s [dev0 43.9] 0 shares
Total 46.9 sol/s [dev0 46.9] 0 shares
Total 44.9 sol/s [dev0 44.9] 1 share
[...]
```

Usage:

```
$ silentarmy --help
Usage: silentarmy [options]

Options:
  -h, --help            show this help message and exit
  -v, --verbose         verbose mode (may be repeated for more verbosity)
  --debug               enable debug mode (for developers only)
  --list                list available OpenCL devices by ID (GPUs...)
  --use=LIST            use specified GPU device IDs to mine, for example to
                        use the first three: 0,1,2 (default: 0)
  --instances=N         run N instances of Equihash per GPU (default: 2)
  -c POOL, --connect=POOL
                        connect to POOL, for example:
                        stratum+tcp://example.com:1234
  -u USER, --user=USER  username for connecting to the pool
  -p PWD, --pwd=PWD     password for connecting to the pool
```

# Equihash solver

SILENTARMY also provides a command line Equihash solver (`sa-solver`)
implementing the CLI API described in the
[Zcash open source miner challenge](https://zcashminers.org/rules).
To solve a specific block header and print the encoded solution on stdout, run
the following command (this header is from
[mainnet block #3400](https://explorer.zcha.in/blocks/00000001687e89e7e1ce48b349e601c89c70dd4c268fdf24b269a3ca4140426f)
and should result in 1 Equihash solution):

```
$ sa-solver -i 04000000e54c27544050668f272ec3b460e1cde745c6b21239a81dae637fde4704000000844bc0c55696ef9920eeda11c1eb41b0c2e7324b46cc2e7aa0c2aa7736448d7a000000000000000000000000000000000000000000000000000000000000000068241a587e7e061d250e000000000000010000000000000000000000000000000000000000000000
```

If the option `-i` is not specified, `sa-solver` solves a 140-byte header of all
zero bytes. The option `--nonces <nr>` instructs the program to try multiple
nonces, each time incrementing the nonce by 1. So a convenient way to run a
quick test/benchmark is simply:

`$ sa-solver --nonces 100`

Note: due to BLAKE2b optimizations in my implementation, if the header is
specified it must be 140 bytes and its last 12 bytes **must** be zero. For
convenience, `-i` can also specify a 108-byte nonceless header to which
`sa-solver` adds an implicit nonce of 32 zero bytes.

Use the verbose (`-v`) and very verbose (`-v -v`) options to show the solutions
and statistics in progressively more and more details.

# Performance

* 47.5 Sol/s with one R9 Nano
* 45.0 Sol/s with one R9 290X
* 41.0 Sol/s with one RX 480 8GB

Note: the `silentarmy` **miner** automatically achieves this performance level,
however the `sa-solver` **command-line solver** by design runs only 1 instance
of the Equihash proof-of-work algorithm causing it to underperform. One must
manually run 2 instances of `sa-solver` (eg. in 2 terminal consoles) to
achieve the same performance level as the `silentarmy` **miner**.

Troubleshooting performance issues:
* By default SILENTARMY mines with only one device/GPU; make sure to specify
  all the GPUs in the `--use` option, for example `silentarmy --use 0,1,2`
  if the host has three devices with IDs 0, 1, and 2.
* If some GPUs have less than ~2.4 GB of GPU memory, run
  `silentarmy --instances 1` (2 instances use ~2.4 GB of GPU memory,
  1 instance uses ~1.2 GB of GPU memory.)
* If you are using an AMD GPU with the **Radeon Software Crimson Edition**
  driver, as opposed to the **AMDGPU-PRO** driver, then edit param.h and set
  `OPTIM_FOR_FGLRX` to 1. This will improve performance by +5% and reduce
  GPU memory usage from 1.2 GB per instance to 805 MB per instance. But do
  **not** set it if you are using the AMDGPU-PRO driver or else it will
  degrade performance by -15% or more.
* If 1 instance still requires too much memory, edit `param.h` and set
  `NR_ROWS_LOG` to `19` (this reduces the per-instance memory usage to ~670 MB)
  and run with `--instances 1`.

# Dependencies

SILENTARMY has primarily been tested with AMD GPUs on 64-bit Linux with
the **AMDGPU-PRO** driver (amdgpu.ko, for newer GPUs) and the **Radeon Software
Crimson Edition** driver (fglrx.ko, for older GPUs). Its only build
dependency is an OpenCL implementation.

Installation of the drivers and SDK can be error-prone, so below are
step-by-step instructions for the AMD OpenCL implementation (**AMD APP SDK**),
for Ubuntu 16.04 as well as Ubuntu 14.04 (beware: the `silentarmy` miner makes
use of Python's `ensure_future()` which requires Python 3.4.4, however Ubuntu
14.04 ships 3.4.3, therefore only the `sa-solver` tool is usable on Ubuntu
14.04.)

## Ubuntu 16.04

1. Download the [AMDGPU-PRO Driver](http://support.amd.com/en-us/kb-articles/Pages/AMDGPU-PRO-Install.aspx)
(as of 30 Oct 2016, the latest version is 16.40)

2. Extract it:
   `$ tar xf amdgpu-pro-16.40-348864.tar.xz`

3. Install (non-root, will use sudo access automatically):
   `$ ./amdgpu-pro-install`

4. Add yourself to the video group if not already a member:
   `$ sudo gpasswd -a $(whoami) video`

5. Reboot

6. Download the [AMD APP SDK](http://developer.amd.com/tools-and-sdks/opencl-zone/amd-accelerated-parallel-processing-app-sdk/)
(as of 27 Oct 2016, the latest version is 3.0)

7. Extract it:
   `$ tar xf AMD-APP-SDKInstaller-v3.0.130.136-GA-linux64.tar.bz2`

8. Install system-wide by running as root (accept all the default options):
  `$ sudo ./AMD-APP-SDK-v3.0.130.136-GA-linux64.sh`

9. Install compiler dependencies which you will need to compile SILENTARMY:
  `$ sudo apt-get install build-essential`

## Ubuntu 14.04

1. Install the official Ubuntu package:
   `$ sudo apt-get install fglrx`
   (as of 30 Oct 2016, the latest version is 2:15.201-0ubuntu0.14.04.1)

2. Follow steps 5-9 above.

## Arch Linux

1. Install the [silentarmy AUR package](https://aur.archlinux.org/packages/silentarmy/).

# Compilation and installation

Compiling SILENTARMY is easy:

`$ make`

You may need to specify the paths to the locations of your OpenCL C headers
and libOpenCL.so if the Makefile does not find them:

`$ make OPENCL_HEADERS=/path/here LIBOPENCL=/path/there`

Self-testing the command-line solver (solves 100 all-zero 140-byte blocks with
their nonces varying from 0 to 99):

`$ make test`

For more testing run `sa-solver --nonces 10000`. It should finds 18681
solutions which is less than 1% off the theoretical expected average number of
solutions of 1.88 per Equihash run at (n,k)=(200,9).

For installing, just copy `silentarmy` and `sa-solver` to the same directory.

# Implementation details

The `silentarmy` Python script is actually mostly a lighteight Stratum
implementation and job dispatcher that sends Equihash work items to 1 or more
instances of `sa-solver --mining` which initializes the solver in a special
"mining mode" so it can be controled via stdin/stdout. By default 2 instances
of `sa-solver` are launched for each GPU (this can be changed with the
`silentarmy --instances N` option.) 2 instances per GPU usually results in the
best performance.

The `sa-solver` binary invokes the OpenCL kernel which contains the core of the
Equihash algorithm. My implementation uses two hash tables to avoid having to
sort the (Xi,i) pairs:

* Round 0 (BLAKE2b) fills up table #0
* Round 1 reads table #0, identifies collisions, XORs the Xi's, stores
  the results in table #1
* Round 2 reads table #1 and fills up table #0 (reusing it)
* Round 3 reads table #0 and fills up table #1 (also reusing it)
* ...
* Round 8 (last round) reads table #1 and fills up table #0.

Only the non-zero parts of Xi are stored in the hash table, so fewer and fewer
bytes are needed to store Xi as we progress toward round 8. For a description
of the layout of the hash table, see the comment at the top of `input.cl`.

Also the code implements the notion of "encoded reference to inputs" which
I--apparently like most authors of Equihash solvers--independently discovered
as a neat trick to save having to read/write so much data. Instead of saving
lists of inputs that double in size every round, SILENTARMY re-uses the fact
they were stored in the previous hash table, and saves a reference to the two
previous inputs, encoded as a (row,slot0,slot1) where (row,slot0) and
(row,slot1) themselves are each a reference to 2 previous inputs, and so on,
until round 0 where the inputs are just the 21-bit values.

A BLAKE2b optimization implemented by SILENTARMY requires the last 12 bytes of
the nonce/header to be zero. When set to a fixed value like zero, not only the
code does not need to implement the "sigma" permutations, but many 64-bit
additions in the BLAKE2b mix() function can be pre-computed automatically by
the OpenCL compiler.

Managing invalid solutions (duplicate inputs) is done in multiple places:

* Any time a XOR results in an all-zero value, this work item is discarded
as it is statistically very unlikely that the XOR of 256 or fewer inputs
is zero. This check is implemented at the end of `xor_and_store()`
* When the final hash table produced at round 8 has many elements
that collide in the same row (because bits 160-179 are identical, and
almost certainly bits 180-199), this is also discarded as a likely invalid
solution because this is statistically guaranteed to be all inputs repeated
at least once. This check is implemented in `kernel_sols()` (see
`likely_invalids`.)
* Finally when the GPU returns potential solutions, the CPU also checks for
invalid solutions with duplicate inputs. This check is implemented in
`verify_sol()`.

Finally, SILENTARMY makes many optimization assumptions and currently only
supports Equihash parameters 200,9.

# Author

Marc Bevand -- [http://zorinaq.com](http://zorinaq.com)

Donations welcome: t1cVviFvgJinQ4w3C2m2CfRxgP5DnHYaoFC

# Thanks

I would like to thank these persons for their contributions to SILENTARMY,
in alphabetical order:
* nerdralph

# License

The MIT License (MIT)
Copyright (c) 2016 Marc Bevand

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
