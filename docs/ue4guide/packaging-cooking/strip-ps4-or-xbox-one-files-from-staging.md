---
sortIndex: 8
sidebar: ue4guide
---

n CopyBuildToStagingDirectory.Automation.cs, a function is called to copy UFS and NonUFS folders to the staging folder (\`SC.StageFiles(...)\`):

The final argument passed to the function is a bool called bStripFilesForOtherPlatforms that is determined using:

```cpp
!Params.UsePak(SC.StageTargetPlatform)\
```

So basically if you aren't using a pakfile, bStripFilesForOtherPlatforms will be true and you're in danger of getting the same behavior I describe.

```cpp
 public bool UsePak(Platform PlatformToCheck)
 {
 return Pak || PlatformToCheck.RequiresPak(this) == Platform.PakType.Always;
 }
```

When this boolean is set, there's some code in DeploymentContext.cs::StageFiles that looks at the full path of the content you are including. If this content contains a folder whose name matches any platforms that you are **not** building for then the content **gets ignored** - as in not copied.

For future generations, strategically use platform names in your build folders: Win32 Win64 WinRT WinRT_ARM UWP Mac XboxOne PS4 IOS Android HTML5 Linux AllDesktop

*Reference From <https://answers.unrealengine.com/questions/241947/additional-asset-directories-not-copied-to-package.html>*
