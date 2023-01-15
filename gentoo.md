# GENTOO HANDBOOK


## INSTALL

### KERNEL 

#### configure sound card

If we don't configure sound card, then in the default configuration we never gonna have sound. So what we need is to follow the [alsa page][9] choose some of the card driver.

In fact i also don't know while one should i choose, but with some likely choose such as `Realtic`, then compiled and reboot, i get the sound.

#### configure frame buffer

There are several times when i compiled a kernel, and reboot to a black screen, it only shows "loading kernel...", and this is what configured in grub. So in a word, system cannot boot the kernel. And i am glad to say, i finally knows the reason, that is caused by frame buffer.

I don't know what it is will going on at the vary beginning if we never configure `Xorg`. But i just modified some [kernel configuration][10] with `Xorg` ability. And it turns wrong, because i missing one step. I didn't specify the firmware file(under `/lib/firmware`) that gpu will uses. May be the page doesn't make it clear.

We should follow the [firmware page][11] to configure. And at the page, we will get the correct location to configure.

![flamebuffer location](../.images/framebf.1.png)

We can write like this: `amdgpu/xxx.bin amdgpu/xxx.bin`, separated by space. However which one should i write? We need boot to a normal system, and run this command to find.

```shell
--- › » dmesg -t | grep amdgpu | grep firmware
Loading firmware: amdgpu/beige_goby_sos.bin
Loading firmware: amdgpu/beige_goby_ta.bin
Loading firmware: amdgpu/beige_goby_smc.bin
Loading firmware: amdgpu/beige_goby_pfp.bin
Loading firmware: amdgpu/beige_goby_me.bin
Loading firmware: amdgpu/beige_goby_ce.bin
Loading firmware: amdgpu/beige_goby_rlc.bin
Loading firmware: amdgpu/beige_goby_mec.bin
Loading firmware: amdgpu/beige_goby_mec2.bin
Loading firmware: amdgpu/beige_goby_sdma.bin
Loading firmware: amdgpu/beige_goby_dmcub.bin
Loading firmware: amdgpu/beige_goby_vcn.bin
amdgpu 0000:03:00.0: amdgpu: Will use PSP to load VCN firmware
```

![location to write](../.images/framebf.2.png)
![location to write](../.images/framebf.3.png)




### configure default gpu driver

If everything configured and work fine, we can successfully login window manager without problem, but when we see the xorg log file `~/.local/share/xorg/Xorg.0.log`, we can still find some error messages show cannot load ati/mesa gpu driver, although it is harmless, it will finally loads the gpu we have, but this comes because we have not configured it yet.

We should specify the correct gpu driver we use in `/etc/X11/xorg.conf`.

```shell
Section "Device"
   Identifier   "My Graphics Card"
   Driver       "amdgpu"
EndSection
```


## SKILL

### compile kernel with correct file

Since i decide to use gentoo, how to configure a minimalist kernel is always a challenge for me. Last failed time i compile kernel, i got confused a lot, because the first time compiled failed, i want to recompile the kernel, but when i run `make menuconfig`, i got the configuration the same as i changed before, even when i removed `.config` file. What i really want is a clean origin setting. And i finally find there are ways to [realized that][8]

```xml
Configuration targets:                  
  config              - Update current config utilising a line-oriented program
  nconfig             - Update current config utilising a ncurses menu based program       
  menuconfig          - Update current config utilising a menu based program               
  xconfig             - Update current config utilising a QT based front-end
  gconfig             - Update current config utilising a GTK based front-end
  oldconfig           - Update current config utilising a provided .config as base         
  localmodconfig      - Update current config disabling modules not loaded                 
  localyesconfig      - Update current config converting local mods to core                
  silentoldconfig     - Same as oldconfig, but quietly, additionally update deps           
  defconfig           - New config with default from ARCH supplied defconfig               
  savedefconfig       - Save current config as ./defconfig (minimal config)                
  allnoconfig         - New config where all options are answered with no                  
  allyesconfig        - New config where all options are accepted with yes                 
  allmodconfig        - New config selecting modules when possible                         
  alldefconfig        - New config with all symbols set to default                         
  randconfig          - New config with random answer to all options                       
  listnewconfig       - List new options                                                   
  olddefconfig        - Same as silentoldconfig but sets new symbols to their default value
  kvmconfig           - Enable additional options for guest kernel support                 
  tinyconfig          - Configure the tiniest possible kernel                              
```

I finally know that there are so many options for make to choose, and what i want is `make defconfig`. It will produce an origin `.config` file. After that run `make menuconfig` to use gui interface to choose the items.


### unmask a package || get the latest version

Sometimes, when we prepare to install a package with emerge, and we find we cannot install it, because it shows the package is masked. We can allow that by editing a configuration file `/etc/portage/package.accept_keywords`

```shell
# for example unmask ddcutil package
# put the line in package.accept_keywords

app-misc/ddcutil ~amd64
```

Then we will be allowed to install. Same way, if we want to install the latest version of package, we can configure it the same way.

```shell
# for example without any configure, allowed firefox version is 102.
# with configuration, we can install the newest verion 108

www-client/firefox ~amd64
```


## ISSUE

### Startx cannot open /dev/tty0

In gentoo, after we installed, and we try to login window manager with `startx` command as a none root user, we will got this error message:

```shell
user$ startx

parse_vt_settings: Cannot open /dev/tty0 (permission denied)
```

The solution is [enabling elogind service][7] at boot time.

```shell
user$ doas rc-update add elogind boot
user$ /etc/init.d/elogind start
```

### `/etc/profile` doesn't take effect

Cannot auto source `/etc/profile` when changing default shell from bash to zsh.

In archlinux, once i change default shell to zsh, everything works fine. But in gentoo, i got some problems, i find everything in `/etc/profile` doesn't take effect.

After searching, i finally find [the reason][3], zsh shell doesn't read `/etc/profile` but `/etc/zsh/zprofile`.

So the solution is writing everything we need to `/etc/zsh/zprofile` for system range, or `~/.config/zsh/zprofile` for user range.


### no sound by default

Alsa stands for advanced linux sound architecture which obtains the sound ability of kernel. In fact, we don't need other tools to gain the sound, but the sound system is muted by default, so we need install `media-sound/alsa-utils` which contains [some useful tools][5] help us to unmute sound and adjust it.

- Unmute sound system

  We can use command `alsamixer` a ncurse tool to unmute sound first. But the system will still have no sound until we configured.

- [Find sound device][6]

  Find the sound devices

  ```shell
  user$ aplay -L 

  ...
  front:CARD=Generic_1,DEV=0
  HD-Audio Generic, ALCS1200A Analog
  Front output / input
  ...
  ```

  Find the part of front, that is the information we need.

- Configure sound

  There are there places we can put our configuration:

  - `/etc/asound.conf`
  - `~/.asoundrc`
  - `~/.config/alsa/asoundrc`

  As my case, i need this configuration to be put.

  ```shell
  defaults.pcm.!card Generic_1
  defaults.pcm.!device 0
  defaults.ctl.!card Generic_1
  ```


### cannot find libnss.so

When i start clash with `./cfw`, it shows error message for libnss.so library missing. If use emerge to search libnss, we cannot find the correct one. In a fact, the correct one is called `nss`.

```shell
emerge --ask dev-libs/nss
```

After installing, i can successfully launch the clash application.



## COMPILED FROM SOURCE


### ddcutil

If we get a external monitor, we cannot adjust brightness with `xbrightness` command, so we need a tool called `ddcutil`, however in gentoo it is masked, so the best way is downloading the source code and compile it manually.

- Download [ddcuitl source file][1] from github
- Follow [this page][2] to compile ddcutil

  ```shell
  #cd to unpacking directory
  user$ ./autogen.sh 
  user$ ./configure
  user$ make 
  user$ doas make install
  ```

  Then ddcutil will be installed in `/usr/local/bin/ddcutil`.

- Load `i2c-dev` module

  By default, we cannot run ddcutil command, cause `i2c-dev` device missing.

  ```shell
  user$ doas ddcutil detect                                       1 ↵
  No /dev/i2c devices exist.
  ddcutil requires module i2c-dev.
  ```

  Load missing module

  ```shell
  user$ doas modprobe i2c-dev

  #check whether loaded
  user$ lsmod |rg -i i2c
  i2c_dev                28672  0
  ```

- Detect display device

  Most of the time it show device 10

  ```shell
  user$ doas ddcutil getvcp ALL |rg -i brightness
  VCP code 0x10 (Brightness                    ): current value =    25, max value =   100
  ```

- Set/get brightness with ddcutil

  ```shell
  user$ doas ddcutil setvcp 10 + 5
  user$ doas ddcutil getvcp 10
  VCP code 0x10 (Brightness                    ): current value =    25, max value =   100
  ```


### picom

Yeah, i love picom, but in gentoo repo i cannot find [pijulius/picom] but the original one, so i need to compile it from source.

- Download [source code][4]
- Download needed dependcies
  
  One trick is that we don't need to install all the dependcies from what github page says. We can only install the dependcies from what gentoo repo's picom needs.

  ```shell
  #find the needed dependcies
  user$ emerge --ask --pretend x11-misc/picom

  #install the need dependcies
  user$ doas emerge --ask uthash libev libconfig
  ```

- Compile source file

  ```shell
  user$ cd picom-implement-window-animations
  user$ git submodule update --init --recursive
  user$ meson --buildtype=release . build 
  user$ ninja -C build
  ```

  Then the binary file will be installed in `~/.local/bin/picom`.

- Run picom

  Make sure the configuration file is placed in proper location.

  ```shell
  user$ picom --experimental-backends --config ~/.config/picom/picom.conf -b
  ```




[1]: https://github.com/rockowitz/ddcutil
[2]: https://www.ddcutil.com/building/
[3]: https://unix.stackexchange.com/questions/537637/sshing-into-system-with-zsh-as-default-shell-doesnt-run-etc-profile
[4]: https://github.com/pijulius/picom
[5]: https://github.com/alsa-project/alsa-utils
[6]: https://wiki.gentoo.org/wiki/ALSA#Configuration_2
[7]: https://wiki.gentoo.org/wiki/Non_root_Xorg
[8]: https://unix.stackexchange.com/questions/224887/how-to-script-make-menuconfig-to-automate-linux-kernel-build-configuration
[9]: https://wiki.gentoo.org/wiki/ALSA
[10]: https://wiki.gentoo.org/wiki/Xorg/Guidehttps://wiki.gentoo.org/wiki/Xorg/Guide
[11]: https://wiki.gentoo.org/wiki/Linux_firmware#Kernels_prior_to_v4.18
