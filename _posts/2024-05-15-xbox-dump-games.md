---
layout: post
title:  "Durango dumplings"
date:   2024-05-15 20:00:00 +0000
categories: games
---

# Bazinga! Dumping Games on a Retail

Update 2024.06.09 - Slightly updated the post with more clarifications.

## Intro

After achieving code execution on the Xbox in Retail mode (System OS: 10.0.22621.3446) and utilizing [CVE-2023-21768](https://github.com/xboxoneresearch/CVE-2023-21768-dotnet) for a kernel read-write primitive, there was a brief period were attempts to extract retail game data failed while in the System VM. While it's possible to mount XVCs into SytemOS, the OS will only let you read/dump the plaintext files in the container (Icons that show in dashboard, app manifest).

Further investigation for alternative methods were running slim, without having to work on furthering the chain for a Host OS exploit. Then an oversight was discovered within the Game OS (ERA)!

Let's look into that. Please note, details in regards to code execution and the entry point aren't shareable at the moment.

## Whoops, wrong call!

When you load a game on an Xbox, the System OS gathers the relevant data relevant to the title launch such as the path to the XVC on the HDD/SSD as well as providing which 'temp' XVD the launching title was allocated. This data is sent to the Host OS when starting the Game OS VM. Once the Game OS begins booting, a service known as `comsrvhost.exe` initializes the game mount and other additional setup using the data supplied from SRA, and validated by Host.

**But the main point of interest arrives when handling the 'temp' XVD mounting:**
![comsrv bug](/assets/durango_dumplings/comsrv_bug.png)

If you look at the highlighted line above, this function calls `XCrdMount` instead of `XCrdMountContentType`. Why does this matter? To keep things simple, making a call to `XCrdMount` does not restrict the content type of the target XVD you're mounting. Whereas with `XCrdMountContentType`, as implied by name, will mount based on the `XvdContentType` and ensure it matches with the target XVD header's `XvdContentType` - refer to here for more: <https://github.com/emoose/xvdtool/blob/master/LibXboxOne/XVD/XVDStructs.cs#L36>

So, based on seeing this occur - that would mean that if you replaced the `tempXX` XVD on the temp partition of the hard drive with a game, and loaded associated license with the game, that it would mount!

**We can check what temp xvd to use based on the stored registry information as displayed:**
![era-temp registry](/assets/durango_dumplings/eratemp_reg.png)

## But how would we read the mount data?

Fortunately, many games will actually use LUA with FFI for gameplay scripting and more. Such game, though there are more, will only be hinted at. But the save data that the game stores, allows the user to decompress and modify the scripts directly - allowing a little bit of execution.
Of course, you do need access to [ConnectedStorage](https://xboxoneresearch.github.io/wiki/games/savegames/) to mount/modify savegames, but as briefly mentioned in the beginning - that's already accessible with Code Execution in SystemOS! [XCrdUtil](https://xboxoneresearch.github.io/wiki/operating-system/xcrdutil/#examples) is your friend here.

NOTE: Lua FFI is not the only option for executing code in EraOS, it is just the most comfortable :D

## Almost there!

Next problem is the license. Every time you load a game on the Xbox, as well as apps excluding inbox apps, the Xbox will lookup for a valid license located in:

- `S:\clip\*.xml`

These licenses that are cached are primarily digital. [Disc-based games](https://xboxoneresearch.github.io/wiki/games/xbox-game-disc/) contain the license on there which is only loaded when inserted into the ODD. Anyways, the license is typically unloaded after the game exits. This means we need a way to force load the license ourselves.

The service responsible is known as **MicrosoftXboxSecurityClip** - located within System32 in the System OS. It's pretty straightforward to reverse and map out the calls (especially thanks to **AppServices**!)

In fact, only needed one function from the function table!

![xboxsecurityclip cs](/assets/durango_dumplings/xboxsecurityclip_cs.png)

## Bazinga!

Now as we launch our vulnerable game and given that the license for our target game is loaded - the game should successfully mount to the T:\ drive within the Game OS. From there, we can abuse the save data to run our own lua code. Unfortunately, no USB devices or other accessible XVD's are mounted so instead - we just send file data over socket to the Host PC.

![example hint](/assets/durango_dumplings/example_hint.png)

Voila!
