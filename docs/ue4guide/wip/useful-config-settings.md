---
sortIndex: 27
sidebar: ue4guide
---

```cpp
DefaultEngine.ini

; Enable BC6h/BC7 support & reduce shader compilation times by removing SM4 support

[/Script/WindowsTargetPlatform.WindowsTargetSettings]

-TargetedRHIs=PCD3D_SM4
```
