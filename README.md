# VR Linux curses
This document is a compilation of issues that I have run into while using SteamVR and ChilloutVR on Linux. Some things have solutions, some have workarounds, and some are just cursed.

# SteamVR
## Useful resources
- [Getting my Valve Index to run flawlessly on ArchLinux with i3wm - WebFreak001](https://gist.github.com/WebFreak001/f8d5852c795e3b5b1fddfc1911e6babb)
- [Virtual reality - Arch Wiki](https://wiki.archlinux.org/title/Virtual_reality)
- [SteamVR for Linux bug tracker](https://github.com/ValveSoftware/SteamVR-for-Linux/issues/)


## First launch
Every first launch of SteamVR crashes more or less when the first frame is rendered. Sometimes it is fine until I grab the headset and make it wake up, but it usually triggers by itself if I just wait 10 seconds. You don't have to wait for it to crash, closing it manually immediately and starting it again works too. Very rarely it doesn't crash even after starting to render, but then it will have an incredibly low framerate (<10 FPS), and it crashes after a few seconds of that.

## Index microphone with pulseaudio
By default the Valve Index microphone will probably not work.

Add `default-sample-rate = 48000` to `~/.config/pulse/daemon.conf` **or** `/etc/pulse/daemon.conf`, then run `pulseaudio -k` to restart pulse.

See also:
- https://github.com/ValveSoftware/SteamVR-for-Linux/issues/215#issuecomment-526791835
- https://gist.github.com/WebFreak001/f8d5852c795e3b5b1fddfc1911e6babb/

## Game crashes
If a VR game crashes, SteamVR will not properly shut down the session, and starting any game at that point will fail (the game will fail to create a VR session and crash). SteamVR must be restarted to fix this.

## Delayed overlay rendering
All overlays, controller models and the SteamVR void (everything that SteamVR or the compositor renders) were several frames behind. This is nauseating in the SteamVR void, and just annoying when you are in a game. Game rendering is otherwise unaffected.

On Some Systems™️ this manifests as persistent crashes instead.

Fixed by adding `"enableLinuxVulkanAsync" : false` under `"steamvr" : {` in `~/.steam/steam/config/steamvr.vrsettings`..

## Rendering artifacts at edge of vision
https://github.com/ValveSoftware/SteamVR-for-Linux/issues/500
https://github.com/ValveSoftware/SteamVR-for-Linux/issues/480

The edge of the screens were flickering grey garbage data. It seems to be the part of the screen that is normally supposed to be completely black, as it's barely visible.

ePutting `RADV_DEBUG=zerovram %command%` in the SteamVR launch options fixes this.


## SteamVR desktop overlay
### The X11 bug (old)
[libx11 issue 176](https://gitlab.freedesktop.org/xorg/lib/libx11/-/issues/176)
For several months after december 16, 2022, there was a bug in the latest version (in the arch repos) of libx11, that caused the SteamVR desktop overlay to cause crashes.

After opening the desktop a third time, or showing it for enough time, Steam would crash. Quite often SteamVR would also crash at that point, taking the open game with it.

#### Fun ChilloutVR side effect
If Steam crashed but SteamVR and ChilloutVR stayed open, it seemed fine (apart from the desktop overlay not working) until someone joined or left the world, or I switched world. It seems like when changing the number of connected users, ChilloutVR tries to tell Steam about it to change the rich presence (even if that is disabled), and then crashes if Steam is no longer running.


### Reliability
Sometimes the desktop overlay just fails to start (says "no desktop found"), usually works to just try again a few times. Many people have this issue always and cannot use it at all.

### Multi-monitor usability
Only one monitor is visible at a time, depending on the cursors location.
The cursor position is not mapped to where you are pointing, instead the full all-screen area (the rectangle containing all screens) is scaled down to be the same width as the overlay, and cursor position is mapped on that area. This means movement is effectively multiplied by the number of screens you have (depending on their relative sizes) as well as heavily offset from where the laser points.

### Full-body trackers
If any full-body trackers are on and connected, launching SteamVR will *always* fail to start the desktop view. This is especially annoying since Tundra trackers turn themselves on when you charge them.

### Alternatives
https://github.com/crispyPin/sinpin-vr/
https://github.com/galister/WlxOverlay/


# ChilloutVR
## Video players
- Video players do not work for all formats.
- They render upside-down.
- Attempting to pause only stops the video, not the audio.
- The audio plays in its own channel so the volume can't be controlled in-game, and it will always start at the default volume (100%, which is loud)
- Skipping in the video does nothing

Video players can be disabled completely with `--disable-videoplayers` in the ChilloutVR launch options.

## MelonLoader
On Some Systems™️, MelonLoader (and the modded game) will not start unless you put `--melonloader.hideconsole` in the launch options. (I have been told not everyone needs this)

[Official MelonLoader Linux installation instructions for Proton games](https://melonwiki.xyz/#/README?id=linux-instructions)

<!--
(These instructions are out of date, use the official ones above instead.)
### Installation
- Download [MelonLoader.x64.zip](https://github.com/LavaGang/MelonLoader/releases/)
- Install [protontricks](https://github.com/Matoking/protontricks/#installation) (aur `protontricks`)
- run `protontricks 661130 winecfg`, click **Libraries**, type in `version` and click **Add**
- run `protontricks 661130 dotnet48`
 -->

## The 2-minute crash
Several things can cause ChilloutVR to crash about 2 minutes in to the session, with nothing useful in the logs. This can happen in VRChat too, but was less predictable there and seemed to come and go with updates.

### config files
On Some Systems™️, having *any* config files in the games config directory (see below) will cause this crash. Removing them or replacing all of them with symlinks to `/dev/null` works, but then you no longer have settings saved. I made [a mod (ConfigHack)](https://github.com/CrispyPin/CVR-ConfigHack) that loads the main settings from a different file instead, and this works (for me) for some reason.

The config directory is usually `~/.steam/steam/steamapps/compatdata/661130/pfx/drive_c/users/steamuser/AppData/LocalLow/Alpha\ Blend\ Interactive/ChilloutVR/`

#### in combination with the Zen kernel
Since I switched to `linux-zen`, having config files no longer caused the 2-minute crash. **However**, ChilloutVR started leaking around 0.3GB per minute, and the kernel seemed to be using another 10GB for no apparent reason. Thus I reverted to using `/dev/null` symlinks and the ConfigHack mod.

### UIExpansionKit
On Some Systems™️, using the UIExpansionKit mod (which many mods depend on) would cause the crash. Switching to the `linux-zen` kernel and [setting the file descriptor limit (on Arch)](https://wiki.archlinux.org/title/Limits.conf#nofile) made this no longer happen.

I first tried only setting the fd limit, which did not have any noticable effect, then switching to zen did fix it. It's possible you only need zen, I have not tried reverting the fd limit.


# OVR Advanced Settings
[OpenVR-AdvancedSettings source](https://github.com/OpenVR-Advanced-Settings/OpenVR-AdvancedSettings/)

Since switching to arch, the Steam release segfaulted on startup (this may have been fixed since). Using the windows version through Proton worked somewhat, but inverted the Y position of the cursor when navigating the menu.

Solution in the end was to compile it myself, this worked, possibly because it's running a normal binary instead of an AppImage.
