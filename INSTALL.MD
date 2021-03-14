
Intro

Anbox provides a userland rootfs, glue logic and kernel modules to run android applications on GNU/Linux.

By default it supports x86_64 architectures. I wanted to run some android APKs on my Raspberry PI 4, which comes with an armv7l kernel.

After a bit of tinkering I got it “running”, though it was quite unstable.

Don’t expect to run your favorite games with this on the PI, not yet anyway.
Prerequisites

    A Raspberri Pi 4
    A fast internet connection
    A desktop/server to build android.img
    Loads of patience

Chapter 1: PI Kernel from source

This is needed to get to get the matching kernel headers / Modules.symvers, so we can build binder/ashmem

Get Dependencies

    $ sudo apt-get install git bc bison flex libssl-dev dkms raspberrypi-kernel-headers libncurses5-dev

    Get the matching kernel using rpi-source

    $ sudo wget https://raw.githubusercontent.com/RPi-Distro/rpi-source/master/rpi-source -O /usr/local/bin/rpi-source && sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update
    
    $ rpi-source

    Extract the current kernel config with

    cd ~/linux
    $ zcat /proc/config.gz > .config

    Make the kernel with

    $ make -j4 zImage modules dtbs

    Install the new kernel with

    $ sudo cp arch/arm/boot/zImage /boot/kernel7l.img
    $ sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
    $ sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
    $ sudo cp arch/arm/boot/dts/*.dtb /boot/

    Reboot the PI and check that the new kernel is loaded

    $ uname -a # Check the build timestamp in the output

Chapter 2: binder_linux && ashmem kernel modules

    Clone anbox-modules
    
    $ git clone https://github.com/anbox/anbox-modules.git
    
    Add the following line on top of binder/binder.c

    #define CONFIG_ANDROID_BINDER_IPC_32BIT

    Follow the install instructions in anbox-modules

Chapter 3: Anbox from source

Now we clone.

    $ git clone https://github.com/anbox/anbox.git --recurse-submodules
    $ cd anbox

Delete the following lines from CMakeLists.txt.

Line 21-25:

    set(C_AND_CXX_WARNINGS "-pedantic -Wno-error=pedantic -Wall -Wextra")

    # Some additional warnings not included by the general flags set above.
    set(EXTRA_C_WARNINGS "-Wcast-align -Wcast-qual -Wformat -Wredundant-decls -Wswitch-default")
    set(EXTRA_CXX_WARNINGS "-Wnon-virtual-dtor -Wold-style-cast")

15 lines down, give or take...

    if ("${cmake_build_type_lower}" STREQUAL "release" OR "${cmake_build_type_lower}" STREQUAL "relwithdebinfo")
      option(Werror "Treat warnings as errors" ON)
    else()
      option(Werror "Treat warnings as errors" OFF)
    endif()

    if (${Werror})
        message(STATUS "Treat warnings as errors")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
      if ("${cmake_build_type_lower}" STREQUAL "release" OR "${cmake_build_type_lower}" STREQUAL "relwithdebinfo")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-error=deprecated-declarations")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=deprecated-declarations")
      endif()
    endif()

Replace uint64_t with uint32_t in src/anbox/input/device.cpp, struct CompatEvent.

Around line 56

    change:    std::uint64_t sec;
    to:        std::uint32_t sec;
    change:    std::uint64_t usec;
    to:        std::uint32_t usec;

This is needed to support 32bit android userland

Now to compile anbox.

   First, we need dependencies... again...

You will encounter an error during apt install stating it cannot find a file if you don't do this first. These files were changed, but here is a link to those files and instructions on what to do with them. I also included these files in the "Extras" folder in this repository.

    $ wget http://ramses.wh2.tu-dresden.de/pub/mirrors/raspbian/raspbian/pool/main/s/systemd/libsystemd-dev_241-7~deb10u5+rpi1_armhf.deb
    $ wget http://ramses.wh2.tu-dresden.de/pub/mirrors/raspbian/raspbian/pool/main/s/systemd/libudev-dev_241-7~deb10u5+rpi1_armhf.deb
    $ dpkg -i libsystemd-dev_241-7~deb10u5+rpi1_armhf.deb
    $ dpkg -i libudev-dev_241-7~deb10u5+rpi1_armhf.deb

    $ sudo apt install build-essential cmake cmake-data debhelper dbus google-mock libboost-dev libboost-filesystem-dev libboost-log-dev libboost-iostreams-dev libboost-program-options-dev libboost-system-dev libboost-test-dev libboost-thread-dev libcap-dev libexpat1-dev libsystemd-dev libegl1-mesa-dev libgles2-mesa-dev libglm-dev libgtest-dev liblxc1 libproperties-cpp-dev libprotobuf-dev libsdl2-dev libsdl2-image-dev lxc-dev pkg-config protobuf-compiler python3-minimal
    
Compile from source:

    $ mkdir build
    $ cd build
    $ cmake ..
    $ make -j4
    $ sudo make install

Chapter 4: Anbox/android.img for armv7

This downloads ~ 40GB of sources and uses ~ 100GB of disk space. You need a powerful desktop for this step. It will take hours.
I did this on a windows machine with virtualbox running debian 10. Good luck on this part. I might add a new file with instructions specifically for this, as it was a royal pain in the {censored}.

    Open a terminal and type

    $ export LC_ALL=C

    Follow instructions in https://github.com/anbox/anbox/blob/master/docs/build-android.md
    Extract android.img with

    unsquashfs android.img

    Modify the heap size from dalvik.vm.heapsize=512m to dalvik.vm.heapsize=128m

    $ nano squashfs-root/system/build.prop # use ctrl-w to search for heapsize

    Repack android.img with

    $ rm android.img
    $ mksquashfs ./squashfs-root android.img

Chapter 5: Run anbox

    Create /var/lib/anbox
    Copy android.img to /var/lib/anbox
    Run anbox container-manager

    $ sudo ANBOX_LOG_LEVEL=debug anbox container-manager --daemon --privileged --data-path=/var/lib/anbox

    Run anbox session-manager

    $ anbox session-manager --single-window --window-size=1024,768

    Watch the logs in /var/lib/anbox/logs

    $ sudo tail -f  /var/lib/anbox/logs/console.log 

Chapter 6: Enjoy the crashes

You should now have anbox running on your pi!

You can connect to it via and and install APKs you want to test

    $ sudo apt install adb
    $ wget http://my-apk....
    $ adb install my-apk....

