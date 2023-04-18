# MY GENTOO HANDBOOK

After 6 years of archlinux usage, i finally move to gentoo and give it a try. After several weeks, i still miss archlinux, since installing something is really cheap in arch, but often take minutes in gentoo. However, i learned more in gentoo and get used to compiling kernel. So, it is really worth to try gentoo.

Okay, another several weeks later, i got to say i love gentoo really, aka. It takes time to get used to gentoo, but once you make it, you will find that gentoo's way is elegant. 


## KERNEL 

### configure sound card

If we don't configure sound card, then with the default configuration we never gonna have sound. So we need to follow the [alsa page][9] and choose some of the card driver.

In fact i also don't know which one should i choose, i just picked `Realtic` and then compiled and reboot, and i get the sound.

### configure frame buffer

There are several times when i compiled a kernel, and reboot to a black screen, it only shows "loading kernel...", and this is something that configured in grub. So in a word, system cannot boot the kernel. And i am glad to say, i finally know the reason, that is caused by error configuration of frame buffer.

I miss one step in [kernel configuration][10] with `Xorg` ability. I didn't specify the firmware file(under `/lib/firmware`) which frame buffer depends on(compile gpu drive to kernel directly). 

We should follow the [firmware page][11] to configure.

![flamebuffer location](../.images/framebf.1.png)

We can write like this: `amdgpu/xxx.bin amdgpu/xxx.bin`, separated by space. However which one should i write? We need boot to a normal system, and run this command to find.
Or we can find in the [page][12].

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


### compile fuse as module

When i run appimage, it occurs an error shows need fuse(file system in userspace) device, and i have not compiled it to kernel, so if we want to use appimage, we need to compile it to kernel. And i choose to compile it as module.

```shell
--- ~/dl » ./Franz-5.9.2.AppImage 
fuse: device not found, try 'modprobe fuse' first

Cannot mount AppImage, please check your FUSE setup.
You might still be able to extract the contents of this AppImage 
if you run it with the --appimage-extract option. 
See https://github.com/AppImage/AppImageKit/wiki/FUSE 
for more information
open dir error: No such file or directory
```


### configure default gpu driver

Usually we can successfully login window manager without problem, but once we check the xorg log file `~/.local/share/xorg/Xorg.0.log`, we will find some error messages show cannot load ati/mesa gpu driver, although it is harmless(it will finally load the gpu we have), but this comes because we have not configured it yet.

We should specify the correct gpu driver we use in `/etc/X11/xorg.conf`.

```shell
Section "Device"
   Identifier   "My Graphics Card"
   Driver       "amdgpu"
EndSection
```


### configure console front

While using framebuffer, the console font size becomes too small, so there is a need to change the font size. I find [the way][12].

1. Find console font name in `/usr/share/consolefonts`.
2. Configure font in `/etc/conf.d/consolefont`.

   ```shell
   consolefont="sun12x22"
   ```

   > Note: remove `.psfu.gz` of the font name.

3. Add consolefont service to boot level.

   ```shell
   rc-update add consolefont boot
   ```

4. After reboot, the font size will be the right one.


## USAGE

### Turn Compile Dir To Tmpfs

When emerge builds package, it will use `/var/tmp/portage/` to store all compile files which will never be used again, so it is a waste of disk space, and we can [change this directory][26] to tmpfs.

We can add this lines to `/etc/fstab`. In a fact `/var/tmp` directory can also change to tmpfs.

```shell 
tmpfs /var/tmp         tmpfs rw,nosuid,noatime,nodev,size=4G,mode=1777 0 0
tmpfs /var/tmp/portage tmpfs rw,nosuid,noatime,nodev,size=4G,mode=775,uid=portage,gid=portage,x-mount.mkdir=775 0 0
```


### Change Keyboard Map

Change console keyboard map

```shell 
# edit /etc/conf.d/keymaps

extended_keymaps="dvorak"
```
Change X11 keyboard map

```shell 
# edit /etc/X11/xorg.conf

Section "InputClass"
	Identifier "Keyboard"
	MatchIsKeyboard "on"
	MatchDevicePath "/dev/input/event*"
	Driver "evdev"
	Option "XkbLayout" "us"
	Option "XkbVariant" "dvorak"
EndSection
```

### correctly configure gpu driver

If we only follow gentoo handbook to install and configure system with amdgpu, then we can get system work. If we run `neofetch`, we can see GPU is the correct one, but it just seems that the real amdgpu is not working, because its display a little slow and cannot set Xorg with *tearfree* and *accerate*.

If we run command `glxinfo -B`, we can see the truth, and it shows we are not using amdgpu but `llvmpipe`. The seem, if we run `glxgears`, only cpu usage increased.

```shell
OpenGL vendor string: Mesa/X.org
OpenGL renderer string: llvmpipe (LLVM 12.0.0, 256 bits)
```

The reason is that we missed one software, we need to follow [gentoo amdgpu chapter][24] to correctly set `VIDEO_CARDS` in `make.conf`.

```shell
VIDEO_CARDS="amdgpu radeonsi"
```

And then update world use flag.

```shell
emerge --ask --deep --changed-use @world
```

After reboot, we run `glxinfo -B` and find that everything is find.

```shell
name of display: :0
display: :0  screen: 0
direct rendering: Yes
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: AMD (0x1002)
    Device: AMD Radeon RX 6500 XT (navi24, LLVM 15.0.7, DRM 3.42, 5.15.88-gentoo) (0x743f)
    Version: 22.2.5
    Accelerated: yes
    Video memory: 4096MB
    Unified memory: no
    Preferred profile: core (0x1)
    Max core profile version: 4.6
    Max compat profile version: 4.6
    Max GLES1 profile version: 1.1
    Max GLES[23] profile version: 3.2
```


### add guru repository

There are so many [third part repository][15] with gentoo. And i will try guru this time.

[Guru][14] is a gentoo repository just like arch's aur. I tried to install with eselect-repository but failed, so i will configure it manually.

1. Create a repo file

   ```shell
   touch /etc/portage/repos.conf/guru.conf 

   [guru]
   location = /var/db/repos/guru
   sync-type = git
   sync-uri = https://github.com/gentoo-mirror/guru.git
   ```

2. syncing

   ```shell
   emerge --sync guru
   ```

> Note: now we cannot find any difference with `emerge search` which not like arch's aur. But we can find the difference in installation


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

I finally know that there are so many options for make to choose, and the one i need is `make defconfig`. It will produce an origin `.config` file. After that run `make menuconfig` to use gui interface to choose the items.


### remove sys-kernel/gentoo-kernel-bin

Because the first time i installed gentoo with this pre-compiled kernel, and change to gentoo-source after successfully boot, so there is no need to keep this binary kernel installed anymore, since every time upgrading system, it will upgrade too.

```shell
user$ doas emerge --depclean virtual/dist-kernel sys-kernel/gentoo-kernel-bin
```


### unmask a package or get the latest version

Sometimes, when we prepare to install a package with emerge, and we find we cannot install it, because it shows the package is masked. We can allow that by editing a configuration file `/etc/portage/package.accept_keywords`

```shell
# for example unmask ddcutil package
# put the line in package.accept_keywords

app-misc/ddcutil ~amd64
```

Then we will be allowed to install. If we want to install the latest version of package, we can configure it the same way.

```shell
# for example without any configure, allowed firefox version is 102.
# with configuration, we can install the newest verion 108

www-client/firefox ~amd64
```

### update configuration file

When we update or install or uninstall software, there is a chance we will face this information.

```markdown
 * IMPORTANT: 7 config files in '/etc' need updating.
 * See the CONFIGURATION FILES and CONFIGURATION FILES UPDATE TOOLS
 * sections of the emerge man page to learn how to update config files.
```

What this information it is telling us there is a need to merge new configuration to the old one, why would that happen? I am grade to say that's the graceful of gentoo, it will never gonna to change our configuration(default all config under `/etc`) unless our permit(adding to the path of `$CONFIG_PROTECT_MASK`). So we need to merge those changes, and there are [some tools][19] will help us.

- dispatch-conf  
- cfg-update 
- etc-update 

The new configuration file named `._cfgxxx` under the same folder where old configuration exists.

Before we merge, we can optimize a little of `dispatch-conf`, since it uses `diff` to compare two files without colors, we can change that.

```shell
user$ doas nvim /etc/dispatch-conf.conf

diff="diff --color=always -Nu '%s' '%s'"
```

Now, it is time to merge.

```shell
user$ doas dispatch-conf
```


### choose other repository

Gentoo has more than one [repository][17], we can add it by hand or use an old too called `layman`. But a modern way is using `eselect repository`.

1. First we need to install it.

   ```shell
   $user: doas emerge --ask app-eselect/eselect-repository
   ```

2. Make sure there already has a `/etc/portage/repo.conf` directory.
3. Make sure there already has a wget config file `wgetrc` in `~/.config`.

   ```shell
   $user: cp /etc/wgetrc ~/.config/
   ```

4. Run the command show all repositories.

   ```shell
   $user: eselect repository list

   # or only show currently configured repository
   $user: eselect repository list -i
   ```

   > Note:
   > 1. Installed, enabled repositories are suffixed with a * character.
   > 2. Repositories suffixed with #, need their sync information updated (via disable/enable) or were customized by the user.
   > 3. Repositories suffixed with @ are not listed by name in the official, published list.

5. Add an new ebuild repository

   ```shell
   # for example add guru
   $user: doas eselect repository enable guru
   ```

6. Sync new repository

   ```shell
   $user: doas emaint sync -r guru
   ```


### use [gentoolto][18]

Gentoolto is a overlay instruction help us to optimize packages, shrink compile and launch time but get bigger package size.

1. Add repository

   ```shell
   $user: doas eselect repository enable mv lto-overlay
   ```

2. Sync repository

   ```shell
   $user: doas emaint sync -r mv
   $user: doas emaint sync -r lto-overlay
   ```

3. Install ltoize

   Unmask ltoize first

   ```shell
   # add this to /etc/portage/package.accept_keywords

   sys-config/ltoize ~amd64
   ```

   ```shell
   $user: doas emerge --ask sys-config/ltoize
   ```

4. Configure `make.conf.lto`

   Just uncomment the one github page shows, but `CPU_FLAGS_X86` need ourselves to add

   ```shell
   # run cpuid2cpuflags to show all matched flag 
   $user: cpuid2cpuflags

   # then add this flag to config file with same format, for my cpu it is
   CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3"
   ```

5. Rebuild system with lto

   ```shell
   $user: doas emerge -e --keep-going @world
   ```


### remove unused agetty

Every time i login system and launch htop, find there are agetty2...agetty6, in a fact, i never gone use the other agetty, so i decide to remove those. And it is quiet simple.

Edit `/etc/inittab`, under TERMINAL section, comment those unused(c2...c6).

```shell
# TERMINALS
#x1:12345:respawn:/sbin/agetty 38400 console linux
c1:12345:respawn:/sbin/agetty --noclear 38400 tty1 linux
#c2:2345:respawn:/sbin/agetty 38400 tty2 linux
#c3:2345:respawn:/sbin/agetty 38400 tty3 linux
#c4:2345:respawn:/sbin/agetty 38400 tty4 linux
#c5:2345:respawn:/sbin/agetty 38400 tty5 linux
#c6:2345:respawn:/sbin/agetty 38400 tty6 linux
```


### use eix

Okay, what is [eix][20]? It is a better version of `emerge --search`, much more faster and flexible.

```shell
user$ doas emerge --ask app-portage/eix
```

After installation, we need to update the cache to index all packages on the system.

```shell
user$ doas eix-update
```

### setting cursor theme

Because i don't wanna to build my gentoo with any gtk or qt library, so there is no need to configure cursor theme for gtk or qt. And i find [arch wiki][21] is much better than gentoo wiki, since it introduce the XDG specification which is very important for me to configure cursor theme for firefox.

So, i only need to configure XDG and Xresources.

I don't want to create `.icons` under home directory, so i need put cursor theme in `/usr/share/icons`.

- XDG configuration

  The config should be put in `/usr/share/icons/default/index.theme`.

  ```shell
  mkdir /usr/share/icons/default
  touch /usr/share/icons/default/index.theme
  ```

  Then add, make sure the cursor_theme_name is the theme directory name.

  ```shell
  [icon theme] 
  Inherits=cursor_theme_name
  ```

- Xresources

  It is quite simple. Add to `~/.Xresources` or other use defined position.

  ```shell
  Xcursor.theme: oreo_red_cursors
  Xcursor.size: 16
  ```

### prevent package auto update or build a local repo

If we update the system and there is a new version of kernel, then the system will auto update kernel even the one `gentoo-source` which needs manually configured. But i don't expect that happened, so i need a way to fix that. 

There are [several ways][23] to implement it:

1. write package to `package.provided`
2. add excluding package to `rsync_excludes`
3. create a local ebuild repository

---
According to [this page][22], we can write the package into `/etc/portage/profile/package.provided` file to prevent it auto update.

```shell
user$ doas touch /etc/portage/profile/package.provided
user$ echo "sys-kernel/gentoo-sources-5.15.88" >> /etc/portage/profile/package.provided
```

Just make sure the package's version need to be specified also.

---
In order to disable sync new ebuild, we can either remove `rsync-verify` USE flag of `portage` or set `sync-rsync-verify-metamanifest=on`. I think the second is better.

```shell
# edit /etc/portage/repos.conf/gentoo.conf and change
sync-rsync-verify-metamanifest = no
```

Then exclude the package and modify `make.conf`

```shell
# create file
user$ touch /etc/portage/rsync_excludes

# write package to this file
user$ echo "gentoo-sources" >> /etc/portage/rsync_excludes

# configure in make.conf
PORTAGE_RSYNC_EXTRA_OPTS="--exclude-from=/etc/portage/rsync_excludes"
```

---
Create standard repo structure. Let's call this repo with `localrepo`.

```shell
user$ mkdir -p /var/db/repos/localrepo/{metadata,profiles} 
user$ chown -R portage:portage /var/db/repos/localrepo
user$ echo 'localrepo' > /var/db/repos/localrepo/profiles/repo_name
```

Specify no sync needed for this repo.

```shell
# edit /var/db/repos/localrepo/metadata/layout.conf 
masters = gentoo
auto-sync = false
```

Enable/announce this repo in system.

```shell
# edit /etc/portage/repos.conf/localrepo.conf 
[localrepo]
location = /var/db/repos/localrepo
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

*Cannot auto source `/etc/profile` when changing default shell from bash to zsh.*

In archlinux, once i change default shell to zsh, everything works fine. But in gentoo, i got some problems, i find everything in `/etc/profile` doesn't take effect.

After searching, i finally find [the reason][3], zsh shell doesn't read `/etc/profile` but `/etc/zsh/zprofile`.

So the solution is writing everything we need to `/etc/zsh/zprofile` for system wide, or `~/.config/zsh/zprofile` for user wide.


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

When i start clash with `./cfw`, it shows some error messages: "libnss.so library missing". If use emerge to search libnss, we cannot find the correct one. In a fact, the correct one is called `nss`.

```shell
emerge --ask dev-libs/nss
```

After installing, i can successfully launch the clash application.


### update system error

```shell
$user: emerge --sync

gpg: refreshing 4 keys ...
gpg: keyserver refresh failed...
```

This problem maybe remote server problem, and a way to fix is use other sync method.

```shell
$user: doas emaint --auto sync
```

### dbus error

I use stack to install xmonad and xmobar since long time ago, cause it is a great way without system to install plenty of haskell modules preventing starting xmonad failure each time even one haskell module has been updated.

But this time in gentoo linux, maybe xmobar version has been updated, i can install xmobar, but failed to start it, and shows:

```shell
--- ~ » xmobar 
xmobar: SocketError {socketErrorMessage = "Network.Socket.connect: <socket: 16>: does not exist (No such file or directory)", socketErrorFatal = True, socketErrorAddress = Just (Address "unix:path=/run/user/1000/bus")}
```

After searching, i tried this method(which turns to be a horrible mistake):

```shell
# add this two lines in /etc/environment
DBUS_SYSTEM_BUS_ADDRESS=unix:path=/run/dbus/system_bus_socket
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/dbus/system_bus_socket
```

Yes, it really fixed the problem of xmobar, it can be launched successfully, but causes problems more than that. When i try to send a notification or create a new tab in firefox by scripts, all fail.

```shell
--- ~ » notify-send "hello"                                           1 ↵
GDBus.Error:org.freedesktop.DBus.Error.ServiceUnknown: The name org.freedesktop.Notifications was not provided by any .service files
--- ~ » firefox-bin --search --new-tab "ls"                           1 ↵
[Parent 5156, Main Thread] WARNING: g_dbus_proxy_new: assertion 'G_IS_DBUS_CONNECTION (connection)' failed: 'glib warning', file /builds/worker/checkouts/gecko/toolkit/xre/nsSigHandlers.cpp:167

(firefox:5156): GLib-GIO-CRITICAL **: 04:00:23.853: g_dbus_proxy_new: assertion 'G_IS_DBUS_CONNECTION (connection)' failed
[Parent 5156, Main Thread] WARNING: g_dbus_proxy_new: assertion 'G_IS_DBUS_CONNECTION (connection)' failed: 'glib warning', file /builds/worker/checkouts/gecko/toolkit/xre/nsSigHandlers.cpp:167

(firefox:5156): GLib-GIO-CRITICAL **: 04:00:23.853: g_dbus_proxy_new: assertion 'G_IS_DBUS_CONNECTION (connection)' failed
```

![firefox dbs error](../.images/firefox-dbus-error.png)

At first, i thought it was the problem of dbus software itself, there must be some problems of communication of process. Yeah, there really are problems of communication of process, but it caused by myself. Because i add that two dbus environment variables in `/etc/environment`.

**Solution:**

1. solve the problem of dbus communication by removing dbus variable in `/etc/environment`.
2. solve the problem of xmobar by adding a softlink in `~/.xinitrc`

   ```shell
   #fix xmobar cannot find /run/user/1000/bus
   ln -sf /run/dbus/system_bus_socket /run/user/1000/bus
   ```

It is a happy ending again, i am so happy to see that, cause i weak up in 3:00 AM(weak up by something of a mouse noise, i finally know this rent house has some mouses) and find the solution(i don't know why i just realized where the problem is at this time, but it happened)!


### When Upgrade Gentoo, it needs to install golang

When i try to upgrade gentoo, i find it tries to auto install golang which is a very large size package. I cannot find the reason, why i need to install golang?

So, i write a post in [reddit][25] and get lots of help and finally get the reason and fixed the problem.

```shell 
# use --tree to see all dependencies as a tree.
doas emerge --ask --verbose --update --deep --newuse --tree @world

[nomerge       ] x11-terms/kitty-0.27.1::gentoo [0.26.5-r1::gentoo] USE="X -test -verify-sig -wayland" PYTHON_SINGLE_TARGET="python3_10 -python3_9 -python3_11" 
[ebuild  N     ]  dev-lang/go-1.20.2:0/1.20.2::gentoo  CPU_FLAGS_X86="sse2" 25,566 KiB
[ebuild  N     ]   dev-lang/go-bootstrap-1.18.6::gentoo  USE="(-big-endian)" 139,654 KiB
[ebuild     U  ]  x11-terms/kitty-shell-integration-0.27.1::gentoo [0.26.5::gentoo] 0 KiB
```

It shows kitty depends on golang, so after uninstalled kitty, there is no need of golang again

```shell 
doas emerge --depclean x11-terms/kitty
```


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

  > Note: we need to make sure i2c-dev already compiled in kernel.

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

  Then the binary file will be installed in `./build/src/picom`.

- Run picom

  Make sure the configuration file is placed in proper location.

  ```shell
  user$ picom --experimental-backends --config ~/.config/picom/picom.conf -b
  ```

### xcolor

In gentoo, i cannot find xcolor package, the great tool, so i have to install it from source.

1. Download the [source code][16].
2. Extra package and compile.

   ```shell
   tar -xf Soft-xcolor-0.5.1-2-g969d652.tar.gz -C ~/.local/apps 
   cd ~/.local/apps 
   cargo install xcolor
   ```

3. It's binary file located in `~/.local/share/cargo/bin`, the better way add this path to environment variable.


### links

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
[12]: https://wiki.gentoo.org/wiki/Fonts#Console_font
[13]: https://wiki.gentoo.org/wiki/AMDGPU
[14]: https://wiki.gentoo.org/wiki/Project:GURU/Information_for_End_Users
[15]: https://qa-reports.gentoo.org/output/repos/
[16]: https://github.com/Soft/xcolor
[17]: https://wiki.gentoo.org/wiki/Eselect/Repository
[18]: https://github.com/InBetweenNames/gentooLTO
[19]: https://wiki.gentoo.org/wiki/Dispatch-conf
[20]: https://wiki.gentoo.org/wiki/Eix#Searching_for_packages
[21]: https://wiki.archlinux.org/title/Cursor_themes#XDG_specification
[22]: https://wiki.gentoo.org/wiki//etc/portage/profile/package.provided
[23]: https://wiki.gentoo.org/wiki/Handbook:AMD64/Portage/CustomTree#Defining_a_custom_ebuild_repository
[24]: https://wiki.gentoo.org/wiki/AMDGPU
[25]: https://www.reddit.com/r/Gentoo/comments/12ep98w/when_upgrade_gentoo_i_find_it_tries_to_install/
[26]: https://wiki.gentoo.org/wiki/Portage_TMPDIR_on_tmpfs
