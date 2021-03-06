# The general setup
While the build guides below (wrt the actual TF configuration and build) are fairly system-invariant for Linux OS, the build guides below assume the following setup.

#### :warning:**Disclaimer**:warning::
There are very likely to be assumptions about your installation or setup that I have not made explicit. Every fresh install I do of \*buntu comes with a litany of library and binaries setup, so it's possible there are more than a few boxes checked by the time I build TF that have not been mentioned here. YMMV.

* * *

## \*Buntu 18.04
Install your preferred flavor of Ubuntu 18.04. Buntu has native support from all the libraries and software you need for your deep-learning env.

I recommend [Xubuntu](https://xubuntu.org/download#lts). It's what all my machines use; clean, fast, lightweight enough for my westmere thinkpad, and without the Ubuntu bloat.


### Properly configured python environment
*Detailed steps [below](#setting-up-your-python-environment).*

You need to have your python environment setup before you build TF. This guide is predicated on a virtualenv setup through [pyenv](https://github.com/pyenv/pyenv) and building for **Python 3**. If you already have your python setup, and understand library pathing, then you should be fine.

Using system site-packages and libraries (stuff rooted in /usr) for python is STRONGLY DISCOURAGED.

### Bazel
*Detailed steps [below](#installing-bazel)*

Bazel is required for building tensorflow. Installation is straightforward, and you have a few options, but make sure you install Bazel `0.21` (`0.22` is not supported by tf).

### Latest TensorFlow source
Clone the latest TensorFlow from the [official repo](https://github.com/tensorflow/tensorflow/) (either selecting the release branch you would like, or simply cloning the entire repo).

```bash
# Clone the entire repo (which includes release branches)
git clone https://github.com/tensorflow/tensorflow.git

# Clone the specific release you want (eg v1.13.1)
git clone https://github.com/tensorflow/tensorflow.git --branch v1.13.1 --depth=1
```


### Optional Setup
The following software need not be installed to build tensorflow from source, but is *highly* recommended if you have supporting hardware.
- [CUDA, cuDNN, TensorRT](#installing-cuda-cudnn-tensorrt): If you have a supported Nvidia GPU, I would say you must configure these libraries to accelerate TensorFlow, as well as many other ML frameworks.
- [MKL](#installing-mkl): Intel Math Kernel Library for Deep Neural Networks. Improves speed of DNN ops when running on CPU.
  - NB: While my GPU wheels are built with support for MKL, in reality, if tensorflow is bult for GPU, it does not actually use MKL. I do it anyway :sunglasses:.

* * *


# Building TensorFlow from source
Why? You probably got those warnings about tensorflow not being optimized for certain instruction sets from tf when you initialized your tf session. Or maybe you wanted support for other services, platforms, or libraries. Maybe you just want a bespoke tensorflow for your exact setup.

The build process is not trivial, but if you've managed to setup all the dependencies and have gone through the build process a few times, it is fairly straightfoward.


## Understanding optimization flags
> The TensorFlow Library wasn't compiled to use X instructions, but these are available on your machine and could speed up CPU computations


**SSE4.1, SSE4.2, AVX, AVX2, FMA** are all extensions to the x86 instruction set architecture for CPUs.\
They all enhance or expand existing instruction set functionality. There are a whole bunch of other ISA (instruction set architecture) extensions, but the ones listed are the most significant and the only ones you'll be specifying to TF. (If you have a Kaby Lake or later Xeon, or a Cannon Lake processor, you also have AVX-512, which is another signficant extension.)

## So how can you build tensorflow to support these extensions?
#### **TL;DR**: Don't do anything. Just use the default `-march=native` for `--config=opt`

### Most popular: the "I want extra work for myself" way
The most common method you will see when googling around is explicitly passing the instructions you want to `--config=opt`. eg:
- Dispensing with `./configure` altogether and just specifying everything to bazel build:
>`bazel build -c opt -copt=-msse4.1 --copt=-msse4.2 --copt=-mavx --copt=-mavx2 --copt=-mfma //tensorflow/tools/pip_package:build_pip_package`

- Or, specifying the config optimization flags in `./configure`:
> `Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: -msse4.1 -msse4.2 -mavx -mavx2 -mfma`

### The "just letting the build default do it for you automatically" way
There's no reason you need to specify any of those optimization flags to the build configuration (**even if you aren't building for your machine/`march`**, more on that below).

#### Understanding `-march=native`
Check your own `march`: `gcc -march=native -Q --help=target | grep march`

*`-march=native`* is the compiler option referring to your native x86_64 microarchitecture--in other words, the instruction set supported by your CPU.\
You do not need to specify or configure your `march` to be native. It already is.

In fact, the only way Tensorflow knows it was not compiled to use those instruction sets is because it checks the capabilities of your architecture by looking at what instructions are supported by `march=native` So if you got warnings for `SSE4.1, SSE4.2, AVX, AVX2, FMA`, it's because your `march=native` says you can do them, but the vanilla TF package was not optimized for them.

However, if your GCC version is older than v5, and you have a CPU newer than Broadwell, your `native` may be set to `broadwell`. I have included a section on how to make sure your [GCC is up to date](#GCC-setup).
 - NB: `broadwell` and `skylake` **have the same optimization flags**, so even if your GCC is a bit dated, and your native is `broadwell`, tensorflow will still build correctly.

**so just leave --config=opt alone. It will automatically build for your instructions by it's default -march=native**

* * *

### What about GPU builds? Do those change `--config` or `-march`?
If you are building for GPU, then you will specify that in an earlier question, and `cuda` will be automatically included in `--config` by bazel. If you specify that again in the `./configure` process, or to `bazel build` like `bazel build --config=opt --config=cuda //tensorflow/tools/pip_package:build_pip_package`, you will even get a warning saying it has already been specified and that duplicate commands may mess things up.

### So, is there any reason you would ever specify other config flags?
Yes. **In my experience**, there are two situations in which you will specify additional config flags, though keep in mind that others who need more features or support or have special constraints will probably have their own `config` specs.

- **MKL**: If you have `mkl` installed on your system, you will want to specify that the bazel build like, eg: `bazel build --config=opt --config=mkl //tensorflow/tools/pip_package:build_pip_package` I believe the bazel std:out even mentions this.
- **a non-native -march**: If you are building tensorflow for a different machine, perhaps one that does not have the same processor architecture as the build machine, then you will want to specify that `march`.
  - For instance, I also build tensorflow for two other machines, my thinkpads, which have older processors. So, my typical build configs for my thinkpads look like this:
    - **ThinkPad T430, with a Core i5-3320M (Ivybridge) processor**:
    > `bazel build --copt=-march="ivybridge" --config=mkl -c opt //tensorflow/tools/pip_package:build_pip_package`

    - **ThinkPad X201, with a Core i5-540M (Westmere) processor**:
    > `bazel build --copt=-march="westmere" -c opt //tensorflow/tools/pip_package:build_pip_package`

* * *

# Library installation and environment setup notes

## GCC setup
It's not necessary to update GCC, at least for building tensorflow. But if your CPU is newer than Broadwell, your GCC -march=native is likely configured for broadwell (which doesn't actually matter, see below). Only `gcc-7` and above support newer architectures. You should be on `gcc-8`. But when you are building tensorflow, make sure you switch to `gcc-7`, as `gcc-8` is not yet a supported compiler for tensorflow.

I say it's not really necessary to update GCC for building tensorflow because, unless you are on a skylake or later Xeon (which has an additional flag `avx512`), because--as mentioned earlier--`-march=broadwell` and `-march=skylake` **use the same optimization flags**.

Also note that **there is no `kabylake` march**. If you are running a Kaby Lake processor, you use `skylake`. There is no Kaby Lake specific optimization (nor Coffee Lake for that matter). The only reason Cannon Lake is a separate march from skylake is because it supports AVX-512 by default (whereas the others had to be special chips, like Xeon, for that set).

If you want to know what your `march=native` is considered by GCC and what that supports, type `gcc -march=native -Q --help=target` in your console.

If you are interested in what instruction sets are supported by what architectures in GCC, check out the GCC docs:
- [gcc-4.9.4 march options docs](https://gcc.gnu.org/onlinedocs/gcc-4.9.4/gcc/i386-and-x86-64-Options.html#i386-and-x86-64-Options) The `gcc` version you most likely have. Note that `broadwell` is the latest intel arch.
- [gcc-7 options docs](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/gcc/x86-Options.html#x86-Options) The latest `gcc` version supported by bazel/tensorflow (can't remember which) for building. Note `skylake` is the latest intel arch.
- [gcc-8 march options docs](https://gcc.gnu.org/onlinedocs/gcc-8.2.0/gcc/x86-Options.html#index-march-12) The current version of GCC (but not supported by bazel)



## Follow these instructions to install both GCC versions, and to be able to switch between them.

```bash
# If you are *not* on 18.04, you must add this PPA to get gcc7/8
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update
sudo apt full-upgrade

#==== Install gcc version
sudo apt install gcc-7 g++-7 gcc-8 g++-8

# slaving g++ to gcc version insures that the g++ version always matches current gcc, so
#  you don't have to switch both
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 100 --slave /usr/bin/g++ g++ /usr/bin/g++-8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 60 --slave /usr/bin/g++ g++ /usr/bin/g++-7

# To switch between gcc-7 and gcc-8:
sudo update-alternatives --config gcc
```

* * *

# Setting up your python environment
### Use [`pyenv`](https://github.com/pyenv/pyenv).
If you got yourself setup with another python environment manager, and understand python system-site packages pathing, then you can probably skip this part.

```bash
#==== First, dependencies
sudo apt install -y curl apt-transport-https cmake make git-core autoconf automake build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget llvm libncurses5-dev xz-utils tk-dev

#==== Installing Pyenv
curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash;

#==== Update your shell config script with the lines suggested by `pyenv-installer`
export PATH="/home/my_user_name/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

# Then source your shell config (you may need to start a new terminal session)
source ~/.zshrc # or .bashrc or whatever
pyenv install 3.7.2
pyenv virtualenv 3.7.2 my_virtualenv_name
pyenv local my_virtualenv_name
pip install -U pip setuptools
pip install wheel numpy scipy keras # MUST install wheel and keras for tf to build successfully

#==== Confirming installation
# If everything has been configured properly, you should see the python pathing has changed
$ which python
/home/$USER/.pyenv/shims/python

$ python -V
Python 3.7.2

$ python
Python 3.7.2 (default, Jan 30 2019, 21:09:18)
[GCC 8.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> print('All set!')
All set!
```

* * *

# Installing Bazel
Consult the [Bazel install guide for Ubuntu](https://docs.bazel.build/versions/master/install-ubuntu.html) and select which installation method you prefer. However, **make sure you install Bazel v0.21** and NOT the latest version, as of TF 1.13.1, `v0.23` or `0.22` as those versions are not yet supported by TF.

I used the binary installer option, which I think is easiest. I install Bazel to `/opt/bazel`.

```bash
# Install prerequisites (super common packages omitted)
sudo apt install pkg-config zlib1g-dev

# Download the v0.21 binary installer from GitHub
wget https://github.com/bazelbuild/bazel/releases/download/0.21.0/bazel-0.21.0-installer-linux-x86_64.sh

# Make installer executable
chmod +x bazel-0.21.0-installer-linux-x86_64.sh

# Install bazel to /opt
sudo ./bazel-0.22.0-installer-linux-x86_64.sh --prefix=/opt/bazel

# Symlink binary
sudo ln -sf /opt/bazel/bin/bazel /usr/local/bin

```
I would recommend against Bazel's installation guide installing to `--user`. The `--user` flag installs Bazel to `$HOME/bin`, which I think is a deadly sin. I don't like packages installing to a top-level home dir.

If you do want to install Bazel for `user`, then I would recommend the following:

```bash
# Bazel's recommended cmd (DONT DO THIS!)
./bazel-<version>-installer-linux-x86_64.sh --user

# Install bazel bin to proper $USER bin, and lib to .bazel
./bazel-<version>-installer-linux-x86_64.sh --bin=$HOME/.local/bin --prefix=$HOME/.bazel

# $HOME/.local/bin should already be on $PATH by default, but if not
# add the following to your shell config file (.bashrc or .zshrc):
echo 'export PATH="${PATH:+${PATH}:}$HOME/.local/bin"' >> ~/.zshrc

```

* * *

# Installing CUDA, cuDNN, TensorRT
Great, now that you have your python environment setup, let's setup your GPU enviroment if you have one. I'd love to include the CUDA, cuDNN, and TensorRT files to this repo as releases, but given cuDNN and TensorRT both lie behind an account authentication wall, I'd rather not risk some licence/legal stuff.

## Cuda
I am assuming a clean build here (from a fresh install).\
If you have a preexisting CUDA installation, or nvidia drivers, the process is somewhat more complicated and you should consult the installation guides.

Remember to use your own file names! The ones listed below were simply the current versions at the time of writing.

###  Installation

```bash
#==== Install CUDA deb
# Make sure you download the network package, NOT the local!
#  The local package does not always support the latest kernel!

sudo dpkg -i cuda-repo-ubuntu1804_10.0.130-1_amd64.deb
sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
sudo apt update

# WARNING: if you install the normal `cuda` package instead of the
#  specific version package, the package manager will automatically
#  update your CUDA installation. Very rarely do CUDA-accelerated
#  libraries support new CUDA versions at release, so you want to lock
#  in your cuda version
sudo apt install cuda-10-0

# Now, add the following lines to your shell config (.zshrc, etc)
export PATH=/usr/local/cuda-10.0/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-10.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}

# Reboot machine to finish installation
reboot
```

#### Now that CUDA should have installed, let's verify our installation
You want to enter `nvcc -V` (nvidia driver) and `nvidida-smi` into your command line. Here's what the output should look like (your case will of course be different).

```bash
#==== Verify nvidia driver
$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2018 NVIDIA Corporation
Built on Sat_Aug_25_21:08:01_CDT_2018
Cuda compilation tools, release 10.0, V10.0.130

#===== Verify GPU processes
$ nvidia-smi
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 410.79       Driver Version: 410.79       CUDA Version: 10.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 1070    Off  | 00000000:01:00.0  On |                  N/A |
|  0%   33C    P8     6W / 151W |    146MiB /  8110MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0       835      G   /usr/lib/xorg/Xorg                           140MiB |
+-----------------------------------------------------------------------------+

```
Great! CUDA's all set!

## cuDNN
cuDNN installation always comes **after** you've installed CUDA.
cuDNN can be a bit trickier than the CUDA installation--I had trouble with this at first--but I think the trick here is to use the tar files instead of the other formats (.deb, .local, or whatever).

You also have to sign up and do a silly survey to get access to the cuDNN (and TensorRT) downloads. After you've done that, grab the cuDNN tar file supporting your CUDA version.

```bash
#==== cuDNN
# tar file method:
tar -xzvf cudnn-10.0-linux-x64-v7.3.1.20.tgz # whatever ver you got
sudo cp cuda/include/cudnn.h /usr/local/cuda/include
sudo cp cuda/lib64/libcudnn* /usr/local/cuda/lib64
sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*

# Now verify cuDNN installation was successful
# For cuDNN 7.5, you should see CUDNN_MAJOR 7, CUDNN_MINOR 5
cat /usr/local/cuda/include/cudnn.h | grep CUDNN_MAJOR -A 2
#define CUDNN_MAJOR 7
#define CUDNN_MINOR 5
#define CUDNN_PATCHLEVEL 0
--
#define CUDNN_VERSION (CUDNN_MAJOR * 1000 + CUDNN_MINOR * 100 + CUDNN_PATCHLEVEL)

#include "driver_types.h"

```
cuDNN, Done!

## TensorRT
While CUDA and cuDNN are required for any GPU-accelerated ML framework, TensorRT is an optional library. But if you've come this far, why not squeeze a little more from your GPU? Plus, it's the easiest installation out of the three.
```bash
#==== TensorRT
# First install CUDA. Then get the latest TensorRT supporting your
#  CUDA version. eg, at the time of writing:
sudo dpkg -i nv-tensorrt-repo-ubuntu1804-cuda10.0-trt5.0.2.6-ga-20181009_1-1_amd64.deb
sudo apt update
sudo apt install tensorrt
sudo apt install uff-converter-tf

# verify:
dpkg -l | grep TensorRT
## Should see something like:
ii  graphsurgeon-tf                                             5.0.2-1+cuda10.0                    amd64        GraphSurgeon for TensorRT package
ii  libnvinfer-dev                                              5.0.2-1+cuda10.0                    amd64        TensorRT development libraries and headers
ii  libnvinfer-samples                                          5.0.2-1+cuda10.0                    all          TensorRT samples and documentation
ii  libnvinfer5                                                 5.0.2-1+cuda10.0                    amd64        TensorRT runtime libraries
ii  tensorrt                                                    5.0.2.6-1+cuda10.0                  amd64        Meta package of TensorRT
ii  uff-converter-tf                                            5.0.2-1+cuda10.0                    amd64        UFF converter for TensorRT package


```
TensorRT is good!

All done with the GPU environment! :clap:

* * *

# Installing MKL
**The following MKL installation guide was copied, verbatim, from TinyMind (https://github.com/mind) at https://github.com/mind/wheels#mkl. That's where I got my wheels before I started building my own. Make sure to check them out!**

> MKL is [Intel's deep learning kernel library](https://github.com/01org/mkl-dnn), which makes training neural nets on CPU much faster. If you don't have it, install it like the following:

```sh
# If you don't have cmake
sudo apt install cmake

git clone https://github.com/01org/mkl-dnn.git
cd mkl-dnn/scripts && ./prepare_mkl.sh && cd ..
mkdir -p build && cd build && cmake .. && make
sudo make install

echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib' >> ~/.bashrc
```

* * *
