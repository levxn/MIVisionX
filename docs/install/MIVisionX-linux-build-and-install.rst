.. meta::
  :description: MIVisionX building and installing on Linux
  :keywords: MIVisionX, ROCm, installation, Linux, source, build

*************************************************************
Building and installing MIVisionX on Linux from source code
*************************************************************

MIVisionX ``4.0`` and later are built on top of the **ROCm Core SDK** and require **ROCm 7.13 or later** for GPU builds. Before building MIVisionX, install the ROCm Core SDK and the remaining :doc:`prerequisites <./MIVisionX-prerequisites>`. The Core SDK provides the HIP and OpenCL runtimes, the ``amdclang++`` compiler, the ``half`` library, and ``RPP``.

Clone the MIVisionX repository:

.. code:: shell

    git clone https://github.com/ROCm/MIVisionX.git
    cd MIVisionX

HIP backend (default)
=====================

.. code:: shell

    mkdir build-hip && cd build-hip
    cmake ../
    make -j$(nproc)
    sudo make install

OpenCL backend
==============

.. code:: shell

    mkdir build-ocl && cd build-ocl
    cmake -DBACKEND=OCL ../
    make -j$(nproc)
    sudo make install

.. note::

    If both the HIP and OpenCL backends will be installed on the same system, use ``-DCMAKE_INSTALL_PREFIX`` to install each into a separate directory so they do not overwrite each other. For example: ``cmake -DCMAKE_INSTALL_PREFIX=/opt/mivisionx-ocl/ -DBACKEND=OCL ../``

.. note::

    **OpenCL platform enumeration failures (ICD loader workaround)**

    On some ROCm releases, OpenCL applications (including ``runvx -affinity:GPU`` and the OpenVX conformance suite) fail with ``clGetPlatformIDs(-1001)`` or report zero platforms, even though the AMD OpenCL driver is installed and a supported GPU/APU is present. This has two causes, both seen in practice:

    * The OpenCL driver is not registered in ``/etc/OpenCL/vendors/``, or the ``.icd`` vendor file shipped by ROCm contains only a relative path (``libamdocl64.so``) that a generic ICD loader cannot resolve. This is a ROCm packaging gap tracked upstream at `ROCm/TheRock#4750 <https://github.com/ROCm/TheRock/pull/4750>`_.
    * Binaries may pick up ROCm's own bundled ``libOpenCL.so`` (via RPATH) instead of the system ``ocl-icd`` loader; the bundled loader does not always honor ``OCL_ICD_VENDORS`` correctly.

    Until the upstream packaging fix ships, work around it with a local vendor file using an absolute path, combined with prioritizing the system loader:

    .. code:: shell

        mkdir -p ~/.config/OpenCL/vendors
        echo "${ROCM_PATH:-/opt/rocm}/lib/opencl/libamdocl64.so" > ~/.config/OpenCL/vendors/amdocl64.icd
        export OCL_ICD_VENDORS=~/.config/OpenCL/vendors
        export LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH

    Verify with ``clinfo``; it should report at least one ``AMD Accelerated Parallel Processing`` platform. This workaround is temporary and should no longer be needed once ``ROCm/TheRock#4750`` lands in a ROCm release.

After installation, MIVisionX files are found under ``/opt/rocm/`` unless ``-DCMAKE_INSTALL_PREFIX`` was specified.

To run tests after building:

.. code:: shell

    make test

CPU-only build
==============

MIVisionX can be built without ROCm or an AMD GPU. Only a C++17 compiler and CMake 3.10 or later are required. Pass ``-DGPU_SUPPORT=OFF`` to CMake:

.. code:: shell

    mkdir build-cpu && cd build-cpu
    cmake -DGPU_SUPPORT=OFF ../
    make -j$(nproc)
    sudo make install
