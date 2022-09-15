# Rocksmith with RS_ASIO on the Steam Deck

## Please note

This is a modified copy of the great guide at https://github.com/theNizo/linux_rocksmith specifically targeted at the Steam Deck. Hopefully this repository is temporary and can be integrated with the original source at some point.

This guide is **untested**, I have retraced all the steps I took after a lot of trial and error. So if you find inconsistencies/unexpected results when following the guide please get back to me so it can be improved, as I can't be sure whether I missed a step.

## Table of contents

- [Rocksmith with RS_ASIO on the Steam Deck](#rocksmith-with-rs_asio-on-the-steam-deck)
	- [Please note](#please-note)
	- [Table of contents](#table-of-contents)
	- [Install necessary stuff](#install-necessary-stuff)
		- [steamos-readonly disable](#steamos-readonly-disable)
		- [Packages](#packages)
	- [wineasio](#wineasio)
		- [Setting up the game's prefix/compatdata](#setting-up-the-games-prefixcompatdata)
		- [Installing RS_ASIO](#installing-rs_asio)
		- [Set up JACK](#set-up-jack)
		- [Starting the game](#starting-the-game)
			- [Command](#command)



## Install necessary stuff

### steamos-readonly disable

```shell
sudo steamos-readonly disable

sudo pacman-key --init
sudo pacman-key --populate archlinux
```

### Packages

Note: Install qpwgraph through Discover (Discover comes preinstalled on Steam Deck)

```shell
sudo pacman -S realtime-privileges wine-staging
# These packages are already on SteamOS so I did not install them:
# pipewire-alsa pipewire-pulse pipewire-jack lib32-pipewire-jack pavucontrol

# the groups should already exist, but just in case
sudo groupadd audio
sudo groupadd realtime
sudo usermod -aG audio $USER`
sudo usermod -aG realtime $USER`
```

Log out and back in.

<details><summary> How to check if this worked correctly</summary>
For the packages, do `pacman -Q <packages here>`. Should output the names and versions without errors.

	For the groups, run `groups`. This will give you a list, which should contain "audio" and "realtime".
</details>

## wineasio

Installing `base-devel` is very useful for using the AUR and compiling in general.

On SteamOS the following additional packages were required to compile wineasio:

```shell
sudo pacman -S base-devel glibc linux-headers linux-api-headers libtool binutils lib32-glibc
# note about these two packages: they are in conflict with lib32-pipewire-jack pipewire-jack
# pacman can remove these packages for you and we can reinstall them once wineasio is compiled
sudo pacman -S lib32-jack2 jack2
```

The official source for wineasio is [wineasio/wineasio](https://github.com/wineasio/wineasio), however I could not get it to work with pipewire-jack.

I took the fixed version from here [TobiasKozel/wineasio](https://github.com/TobiasKozel/wineasio) and modified it slightly to get it to compile on the Steam Deck, see [here](https://github.com/BWagener/wineasio/commit/8e24a15801b5980c9245ebf4fe30a722857f7e40) for the modification I made.

```shell
# retrieve fixed version:
git clone https://github.com/BWagener/wineasio.git
cd wineasio
# build
rm -rf build32
rm -rf build64
make 32
make 64

# Install on normal wine
sudo cp build32/wineasio.dll /usr/lib32/wine/i386-windows/wineasio.dll
sudo cp build32/wineasio.dll.so /usr/lib32/wine/i386-unix/wineasio.dll.so
sudo cp build64/wineasio.dll /usr/lib/wine/x86_64-windows/wineasio.dll
sudo cp build64/wineasio.dll.so /usr/lib/wine/x86_64-unix/wineasio.dll.so
```

`wineasio` is now installed on your native wine installation.

<details>
	<summary>How to check if it's installed correctly</summary>

	find /usr/lib/ -name "wineasio.dll"
	find /usr/lib/ -name "wineasio.dll.so"
	find /usr/lib32/ -name "wineasio.dll"
	find /usr/lib32/ -name "wineasio.dll.so"

This should output 4 paths (ignore the errors).
</details>

To make Proton use wineasio, we need to copy these files into the appropriate locations:

```shell
# !!! WATCH OUT FOR VARIABLES !!!
# SteamOS Proton Experimental Installation:
PROTON='/home/deck/.local/share/Steam/steamapps/common/Proton - Experimental/files'
cp /usr/lib32/wine/i386-unix/wineasio.dll.so "$PROTON/lib/wine/i386-unix/wineasio.dll.so"
cp /usr/lib/wine/x86_64-unix/wineasio.dll.so "$PROTON/lib64/wine/x86_64-unix/wineasio.dll.so"
cp /usr/lib32/wine/i386-windows/wineasio.dll "$PROTON/lib/wine/i386-windows/wineasio.dll"
cp /usr/lib/wine/x86_64-windows/wineasio.dll "$PROTON/lib64/wine/x86_64-windows/wineasio.dll"
```

After successfully compiling wineasio reinstall lib32-pipewire-jack pipewire-jack

```shell
# remove lib32-jack2 jack2 when prompted
sudo pacman -S lib32-pipewire-jack pipewire-jack
```

### Setting up the game's prefix/compatdata

Note: WIP I was prompted to install wine-mono after running the following commands. I did so using [packer](https://archlinux.org/packages/community/x86_64/packer/) and the [wine-mono](https://archlinux.org/packages/community/any/wine-mono/) package from the [AUR](https://wiki.archlinux.org/title/Arch_User_Repository)

However, I don't know whether this step is necessary so if you like please try to skip it and report back.

```shell
# SteamOS library on internal storage:
STEAMLIBRARY='/home/deck/.steam/steam'
```

1. Delete or rename `$STEAMLIBRARY/steamapps/compatdata/221680`, then start Rocksmith and stop the game once it's running.
1. `WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx regsvr32 /usr/lib32/wine/i386-windows/wineasio.dll` (Errors are normal, should end with "regsvr32: Successfully registered DLL [...]")

I don't know a way to check if this is set up correctly. This is one of the first steps I'd redo when I have issues.

### Installing RS_ASIO

[Download](https://github.com/mdias/rs_asio/releases) the newest release, unpack everything to the root of your Rocksmith installation (`$STEAMLIBRARY/steamapps/common/Rocksmith2014/`)

Edit RS_ASIO.ini: fill in `WineASIO` where it says `Driver=`. Do this for `[Asio.Output]` and `[Asio.Input.0]`. If you don't play multiplayer, you can comment out Input1 and Input2 by putting a `;` in front of the lines.

### Set up JACK

Open pavucontrol ("PulseAudio Volume Control"), go to "Configuration" and make sure there's exactly one input device and one output device enabled.

All available devices will automatically be tied to Rocksmith, and the game doesn't like you messing around in the patchbay (= it would crash).

### Starting the game

Delete the `Rocksmith.ini` inside your Rocksmith installation. It will auto-generate with the correct values. The only important part is the `LatencyBuffer=`, which has to match the Buffer Periods.

Steam needs to be running.

If we start the game from Steam, the game cant connect to wineasio (you won't have sound and will get an error message). So there's two ways around that:

#### Command

```shell
STEAMLIBRARY='/home/deck/.steam/steam'
PROTON='/home/deck/.local/share/Steam/steamapps/common/Proton - Experimental/files'

cd $STEAMLIBRARY/steamapps/common/Rocksmith2014

PIPEWIRE_LATENCY=256/48000
WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx
WINEDLLOVERRIDES="wineasio=n,b"
"$PROTON/bin/wine" $STEAMLIBRARY/steamapps/common/Rocksmith2014/Rocksmith2014.exe

```

`PIPEWIRE_LATENCY`: Rocksmith needs a sample rate of 48000. 256 refers to the buffer size. This number worked great for me, but you can experiment with others, of course.

(You don't always have to do this, but this give me the most reliable experience) As soon as you see the games window, take the focus away from it, eg. on a different window. Don't focus Rocksmith until the logos start to appear (it's usually the same amount of time). At this point, RS_ASIO is initialized and you can start playing.
