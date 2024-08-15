---
layout: post
title:  "News recap v2"
date:   2024-08-15 00:00:00 +0000
categories: news
---

A few things have happened since the last news recap


## Kernel Exploit via Game Script

- The announced Kernel Exploit ([Collateral Damage](https://github.com/exploits-forsale/collateral-damage)) for Xbox One/Series got released by [carrot-c4k3](https://github.com/carrot-c4k3) and [landaire](https://github.com/landaire)!
- The Kernel exploit was patched, and the Game Script App was taken off the store and blacklisted by Microsoft.

Vulnerable to **Collateral Damage**:

- 10.0.25398.4478
- 10.0.25398.4908
- 10.0.25398.4909

Patched:
- 10.0.25398.4910 and above

There are two variants of Collateral Damage:

- **Local version** - This is meant to be deployed via a compatible file explorer application like [Adv File Explorer (Full Trust)](https://apps.microsoft.com/detail/9mvsvn9d3g5z)

- **Remote version** - Requires only the Game Script payload code to be entered (f.e. rubber ducky) or copy pasted from another app. The [solstice payload server](https://github.com/exploits-forsale/solstice/tree/main/crates) application will serve the rest of the exploit chain to the console by sideloading the exploit stages from the PC. 

For more technically detailed instructions, see [Collateral Damage](https://github.com/exploits-forsale/collateral-damage) and [Solstice](https://github.com/exploits-forsale/solstice).

## Durango Dumplings (v1)

While the LUA exploit is independent of the OS version, as it's a bug (or rather feature) in the game "Warhammer: Vermintide 2", the mounting of Games via the ERA Temporary XVD trick (aka. [Durango dumplings](https://xboxoneresearch.github.io/games/2024/05/15/xbox-dump-games.html)) has produced mixed results from users.

There are reports of games showing "phantom update messages" after attempting a dump.
Moving the game container off the console onto external media, deleting the container from internal storage, and moving it back to internal storage again fixes this issue.

Another problem was the minimum file size of 2GB, the standard file size of an Era Temp XVD.
If the game is smaller than that, Warhammer Vermintide 2 would simply regenerate the Temp XVD, making the dumping of the game impossible

Who can utilize Durango Dumplings v1?

- 10.0.25398.4478 - Vulnerable to Durango dumplings v1
- 10.0.25398.4908+ - Not vulnerable to v1

For users who are able to utilize game dumping, [Emma / @InvoxiPlayGames](https://github.com/InvoxiPlayGames) wrote two nice utilities to assist with this process:
- [LicenseClipFinder](https://github.com/InvoxiPlayGames/LicenseClipFinder), for finding out which game a license belongs to, and
- [OneDumpGame](https://github.com/InvoxiPlayGames/OneDumpgame), for actually dumping your games.

For anyone else interested in dumping their games on later firmwares, you are not out of luck! Introducing:

## Durango Dumplings v2 (Monosodium glutamate edition)

After the reports of the issues mentioned for the original Durango Dumplings method, we investigated once more in search of another method.

On both OS versions **10.0.25398.4908** and **10.0.25398.4909** we were still able to copy a game XVC to the temp. XVD location, and were also stil able to the appropriate license - so far so good.

However, when starting the game... The T:\ drive was just empty!

> Note that the following theory is just that - a theory. We were not able to confirm that this is the actual fix they implemented.
It seems like HostOS now performs some additional checks to ensure that only a single XVC with a __Content Type__ equivalent to 1, meaning a *Title* XVC, gets mounted into the game EraOS VM.

After realizing this, we decided to try some more outlandish ideas:

- EraOS also contains __xcrdapi.dll__, responsible for mounting and reading XVCs. Surely we need to be in an elevated context to be able to use it, just like in SystemOS, right? __Wrong!__
We can freely create XCrd contexts inside of the game process, without any authentication checks.

- What would happen if you unmounted the main game container, __while the game was still running?__ Turns out this does not actually crash or shutdown the container, but instead just happily does what you asked it to. Even the game still runs fine!

With knowledge of both of these new discoveries, we are proud to announce:
![Dumplings v2](/assets/news_recap_20240815/dump_v2.png)

By unmounting Vermintide 2 itself first, which has the __Content-Type__ 1 (as it is a Title), we can bypass the HostOS restriction of one-title-mounted-at-a-time and just mount another __Content-Type__ Title XVC! The best part about this is that we can also just reference locally stored XVCs, and are no longer limited to just the one temp XVD slot - No more fiddling with copying XVCs in SystemOS, which also means that the 2GB minimum file size limit is gone!

It is verified to be compatible with OS **10.0.25398.4478, .4908 and .4909**! Possibly, if another entrypoint is developed, it could also be compatible with the more recent versions.

For everyone interested in trying out this new dumping method, find the release [here, in the LuaFFI-CE repo](https://github.com/xboxoneresearch/LuaFFI-CE)!

Alongside the new Lua dumping script, we are also releasing a new dumping receiver! It features a fancy progress display, alongside some fixes for transferring large (4GB+) files.

![TwoDump](/assets/news_recap_20240815/twodump.png)

PS: Happy easter-egg hunting in exclusive titles!

[![Undertale easteregg](/assets/news_recap_20240815/undertale_eastergg.png)](https://x.com/vyletbunni/status/1823883515932381556)

## Wiki

As we are currently mostly busy with writing software to aid the usability of the retail exploits, we are sadly falling a bit behind on keeping the wiki up to date.

There are some great contributions happening in the community, with people writing scripts, tutorials and generally supporting other users. To gather all that knowledge in a central place, we would like to welcome everyone to submit their work to the [Xbox One Research Wiki!](https://xboxoneresearch.github.io/wiki)

If you have the time and are not afraid of writing a bit of Markdown documentation, please read the wiki's [contribution paragraph](https://xboxoneresearch.github.io/wiki/#contributing).
We appreciate everyone who contributes something to the wiki!

## PowerShell

Shortly after the exploit released, we realized that running PowerShell posed to be an issue. The main executable "pwsh.exe" and most of the main libraries are Microsoft-signed and, due to that, can be run without any issues. We did not get so lucky, however, with some of its third-party dependencies - They were not signed by Microsoft and as such failed to load - most prominently one of the most widely used .NET libraries in existence: `Newtonsoft.Json`.

Luckily, we were able to overcome this issue by utilizing the .NET SDK and its built-in `dotnet.exe msbuild` functionality. By feeding it an XML file containing a custom C# MSBuild task, we can execute our own arbitrary C# code - completely bypassing the code-signing requirement.

We were able to use this to create [SharpShell](https://github.com/xboxoneresearch/SharpShell), a custom assembly loader that bypasses the normal signature-verifying assembly loading, then loads and executes the PowerShell assembly itself.

Rejoice - no longer do you have to suffer with just cmd, for PowerShell is here to save the day.

## Solstice

The exploit loader/framework [Solstice](https://github.com/exploits-forsale/solstice) is coming along nicely.

The daemon component `solstice-daemon` is now capable of punching a hole through the Xbox firewall to allow incoming traffic and serve a nice, persistent ssh/sftp daemon.

[landaire](https://github.com/landaire) wrote a new blog post about the development of the PE loading feature: [Reflective PE Loader for Xbox](https://landaire.net/reflective-pe-loader-for-xbox/). Check it out, it's a very good read!

We are currently working on making it a bit fancier for the end user.

![rare achievement toast](/assets/news_recap_20240815/collat_achievement.webp)

## UWP Sideloading

UWP sideloading, after setting a few registry keys, works fine on non-series consoles. For more information, check out the [wiki article.](https://xboxoneresearch.github.io/wiki/development/deploying-uwp-apps/).

It still does not work on Xbox Series-consoles, however.

Short summary:

- Apps on Series S/X are deployed, and the icon and name do show in the dashboard. On start, the app always returns `0x80073cfc` which is a common error code for: __the package has been modified / is corrupted__. This error is mostly found in `AppXDeploymentServer.dll`.

- __ETW tracing__ has not been done yet to pinpoint the exact location where that error was thrown.

- As we are using the global test key for encrypting the content package, as the Xbox one family does contain it and allows it to be used by SP, there is the theory that for series console they don't support that specific key anymore.

TL;DR: still a work in progress.

## Xbox-Linux

[Shadow LAG](https://github.com/lllsondowlll) wrote a v86 emulated Linux system for use on the Xbox One/Series.

Recently, [SternXD](https://github.com/SternXD) helped out by creating the CI deployment pipeline for it.

You can find it here: [XBOX_Linux](https://github.com/xboxoneresearch/XBOX_Linux)

## New system update

On 2024/08/13 a new Systemupdate got released - Version `10.0.25398.4911`.

![OS 4911](/assets/news_recap_20240815/sysupdate_4911.png)

Source: [Support - XBOX](https://support.xbox.com/en-US/help/hardware-network/settings-updates/whats-new-xbox-one-system-updates)

## Releases

- 2024.07.28 - [SharpShell](https://github.com/xboxoneresearch/SharpShell) - A bootstrapper for running PowerShell on your Xbox One / Xbox Series console. By @XboxOneResearch
- 2024.07.21 - [LicenseClipFinder](https://github.com/InvoxiPlayGames/LicenseClipFinder) - Prints out Content-ID / Title name for S:\Clip\ files by @InvoxiPlayGames
- 2024.07.19 - [OneDumpGame](https://github.com/InvoxiPlayGames/OneDumpgame) - Lua payload to dump games via [Durango dumplings](https://xboxoneresearch.github.io/games/2024/05/15/xbox-dump-games.html) by @InvoxiPlayGames
- 2024.07.17 - [Interop repo](https://github.com/xboxoneresearch/Interop) - Xbox SystemOS specific interop code (powershell scripts, c# modules, msbuild tasks) by @XboxOneResearch
- 2024.07.15 - [Collateral Damage Exploit v1](https://github.com/exploits-forsale/collateral-damage) - Kernel exploit for Xbox SystemOS using CVE-2024-30088, in combination with [Solstice](https://github.com/exploits-forsale/solstice) by @carrot-c4k3 and @landaire
- 2024.07.09 - [LuaFFI-CE](https://github.com/xboxoneresearch/LuaFFI-CE) - Base Lua FFI (Warhammer Vermintide 2) savegame exploit by @XboxOneResearch
- 2024.05.05 - [XBOX Linux](https://github.com/xboxoneresearch/XBOX_Linux) - Emulated Linux environment in JavaScript / UWP  by @ShadowLAG