# fydeos_os_developer_challenge



Here're steps I've taken to finish FydeOS OS developer challenge.



## 0. Connectivity, tools & infrastructure readiness

**Network part**

I've already have 2 VPS running on Vultr, setup by me but owned by my friends. And I use v2ray as the proxy client.

The bandwidth ISP provided  is only 100Mbps, looks like I cannot get good ping and speed. I didn't tweak those vps before because I just need google to search.

![](./proxy_ping.png)

As I tested, the speed of download from google source site is not stable, keeps changing from 50KB to 200KB. But I really don't want to upgrade my ISP service and network devices for just a test. So I kept using my previous servers without registering a new one on Digital Ocean as recommended.

I usually like to setup socks/http/https proxy explicitly with config files in my home environment because it's clear when things goes wrong. My previous company provided AnyConnect vpn, so I didn't need to setup the transparent router. If it's necessary, it could be done with an OpenWRT router for this purpose.



**Local development environment**

I'm using an old Thinkpad T460p with 32G memory and 512G SSD, running windows 11. I use WSL2 as the linux part, with 20G memory and 256G vm disk by default, running Ubuntu 20.04.



## 1.  Get Chromium OS code and start building

**Config proxy in WSL2**

For the wsl2 proxy part, need to [disable windows firewall on interface `vEthernet(WSL)` first](https://github.com/microsoft/WSL/issues/4139#issuecomment-850071837).

And setup `~/.gitconfig `  and `~/.curlrc` to use proxy running on windows.

Then setup environment variables: `http_proxy/https_proxy/all_proxy`

Generate a `~/.boto` file with proxy set, looks like it's for `gclient`, not very sure if it's mandatory.



**Prepare before chroot**

Just as google document said. I run the following command lines in terminal:

``` shell
# setup needed apps for the build process
sudo apt-get install git gitk git-gui curl xz-utils python3-pkg-resources python3-virtualenv python3-oauth2client

# prepare depot_tools and setup the path (NEED REPLACEMENT) in ~/.bashrc
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=/path/to/depot_tools:$PATH

# setup locales to en_US.UTF-8
sudo apt-get install locales
sudo dpkg-reconfigure locales

# configure git, NEED TO REPLACE email and name to my own
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

# init work directory
mkdir -p ~/chromiumos
cd ~/chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest -b release-R96-14268.B
repo sync -j4

# create chroot environment and enter
cros_sdk
```

Actually I did `repo sync` for main branch 2 days before. So I just switched branch by: `repo forall -c 'git checkout -b release-R96-14268.B' `.



**Build image inside chroot**

> Note: When building main branch previously, I got a `Permission Error: [Errno 13]`. Found [this post](https://thelinuxcluster.com/2021/03/05/python-errno-13-permission-denied/) as reference and solved.
>

``` shell
# Note: this part works only inside chroot, under ~/chromiumos/src/scripts

# setup board
export BOARD=amd64-generic
setup_board --board=${BOARD}
./set_shared_user_password.sh

# for the SEMAPHORE permission error problem
sudo chmod 777 /dev/shm
sudo chmod +t /dev/shm

# build packages fastest as commented in script, not sure if I missed something
./build_packages --board=${BOARD} --nowithautotest --noworkon

# build test image
./build_image --board=${BOARD} --noenable_rootfs_verification test

# TODO: convert image to qemu raw image
mkdir ~/chromiumos/images
./image_to_vm.sh --from=/mnt/host/source/src/build/images/${BOARD}/latest/ --test_image
```



Met an error during the build_package process, looks like network error:

![](./build_packages_error.png)



The build process stopped outputting later, no response when pressing enter key, not sure if it's caused by the previous network error. I broke the process and re-executed original `./build_image` script. It really took some time.

Here's the build_packages result.

![](./build_packages_result.png)

And the build_image result.

![](./build_image_result.png)

Generated vm image at: `/mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_30_1804-a1/chromiumos_qemu_image.bin`.

To be honest, it's really NOT difficult in this part. But it took longer than I thought. I think it will be better with more powerful PC with ubuntu installed directly (not via WSL2), and network speed affects the build time obviously as well.



**Run image in qemu**

As learned from [Google's document](https://chromium.googlesource.com/chromiumos/docs/+/2efe4b73ea2109870480a3d6148024686faf1e6e/cros_vm.md). VM could be started with this command.

``` shell
# (inside)
export BOARD=amd64-generic
cros_vm --image-path /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_30_1804-a1/chromiumos_qemu_image.bin --start --board ${BOARD} --qemu-args " -vnc :0"
```

It will start a vm which could be connected via VNC Viewer in Windows. But it keeps restarting at `Booting the kernel.` line.

![](./vnc_boot_hang.png)



I haven't any experience using qemu. So I tried to install qemu for windows then double check the corresponding image path in windows, it is:  `\\wsl.localhost\Ubuntu\home\cliff\Workspace\chromiumos\src\build\images\amd64-generic\R96-14268.84.2022_05_30_1804-a1\chromiumos_qemu_image.bin`.

Then I started qemu for windows with this command:

``` shell
# windows power shell, under C:\Program Files\qemu
.\qemu-system-x86_64.exe -drive file="\\wsl.localhost\Ubuntu\home\cliff\Workspace\chromiumos\src\build\images\amd64-generic\R96-14268.84.2022_05_30_1804-a1\chromiumos_qemu_image.bin,format=raw,index=0,media=disk"
```

Qemu for windows hangs at `Booting the kernel.` line.

![](./qemu_boot_hang.png)



I guess I got a wrong image. Maybe need to re-compile.



## 2. Kernel replacement

Did a quick search and found [this article](https://colinxu.wordpress.com/2020/08/06/build-chromium-os%E2%80%8E-for-qemu/). As it commented, looks like build file for kernel 5.10 is already in ` ~/trunk/src/third_party/chromiumos-overlay/sys-kernel/chromeos-kernel-5_10/`, so I think just need to do as following sequence (NO TIME TO TEST YET):

1. modify the kernel version in `~/trunk/src/overlays/overlay-amd64-generic/profiles/base/make.defaults`, change kernel-4_14 to kernel-5_10
2. unemerge old kernel
3. build new kernel
4. build image again

Command to be used in step 2/3/4:

``` shell
# (inside)
export BOARD=amd64-generic
emerge-amd64-generic –unmerge sys-kernel/chromeos-kernel-4_14
emerge-amd64-generic sys-kernel/chromeos-kernel-5_10
./build_image --board=${BOARD} --noenable_rootfs_verification test
```

Then use the new image for test.



## 3. CrOS devserver in docker
`start_devserver` will report permission error, need to specify writable `static_dir`. For example:

``` shell
# (inside)
start_devserver --static_dir=./static
```

I got another warning, guess that's because of "python code not up-to-date".

```
[30/May/2022:22:02:36] ENGINE Listening for SIGTERM.
INFO:cherrypy.error:[30/May/2022:22:02:36] ENGINE Listening for SIGTERM.
[30/May/2022:22:02:36] ENGINE Listening for SIGHUP.
INFO:cherrypy.error:[30/May/2022:22:02:36] ENGINE Listening for SIGHUP.
[30/May/2022:22:02:36] ENGINE Listening for SIGUSR1.
INFO:cherrypy.error:[30/May/2022:22:02:36] ENGINE Listening for SIGUSR1.
[30/May/2022:22:02:36] ENGINE Bus STARTING
INFO:cherrypy.error:[30/May/2022:22:02:36] ENGINE Bus STARTING
Exception in thread Thread-1:
Traceback (most recent call last):
  File "/usr/lib64/python3.6/threading.py", line 916, in _bootstrap_inner
    self.run()
  File "/usr/lib64/python3.6/threading.py", line 864, in run
    self._target(*self._args, **self._kwargs)
  File "/usr/lib64/devserver/health_checker.py", line 60, in func_require_psutil
    return func(*args, **kwargs)
  File "/usr/lib64/devserver/health_checker.py", line 141, in _refresh_io_stats
    prev_disk_io_counters = psutil.disk_io_counters()
  File "/usr/lib64/python3.6/site-packages/psutil/__init__.py", line 2032, in disk_io_counters
    rawdict = _psplatform.disk_io_counters(**kwargs)
  File "/usr/lib64/python3.6/site-packages/psutil/_pslinux.py", line 1117, in disk_io_counters
    for entry in gen:
  File "/usr/lib64/python3.6/site-packages/psutil/_pslinux.py", line 1090, in read_procfs
    raise ValueError("not sure how to interpret line %r" % line)
ValueError: not sure how to interpret line '   1       0 ram0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0\n'

[30/May/2022:22:02:37] ENGINE Serving on http://:::8080
INFO:cherrypy.error:[30/May/2022:22:02:37] ENGINE Serving on http://:::8080
[30/May/2022:22:02:37] ENGINE Bus STARTED
INFO:cherrypy.error:[30/May/2022:22:02:37] ENGINE Bus STARTED
```

![](./local_dev_server.png)

Although it's running, I'm not sure what it serves, and if it works properly. I think I may need to tweak some parameters to "link" the "dev_server" with the environment or running instance.

The document looks not up-to-date with the actual script, for example, the `-t` parameter is already marked as deprecated when I checked the output of `start_devserver --help`, but it's still listed in the [document](https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/master/docs/devserver.md).



## 4. Connecting the dots





## 5. Documentation and other

Every time I start `build_packages` process, it took hours on my machine, much longer than the 90 minutes stated in document. Sometimes it just keeps outputting something not that informative like this:

``` shell
>>> 22:03:41 Still building (chromeos-base/chrome-icu-96.0.4664.209_rc-r1:0/96.0.4664.209_rc-r1::chromiumos, ebuild scheduled for merge to '/build/amd64-generic/') after 1:04:59.943283
```

I don't know if that's because my network is poor? or my PC is not powerful enough? To answer that question, I need some comparison which I cannot do right now.



I have to admit that I haven't used or compiled chromium os several days before, my understanding to that is still very poor.
