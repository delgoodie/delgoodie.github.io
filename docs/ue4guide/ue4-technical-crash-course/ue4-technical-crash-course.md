---
sortIndex: 1
sidebar: ue4guide
---

# Overview Of Engine

- <https://docs.unrealengine.com/latest/INT/Programming/Introduction/index.html>
- [GDC Europe 2014: Unreal Engine 4 for Programmers - Lessons Learned & Things to Come:](http://www.slideshare.net/GerkeMaxPreussner/gdc-europe-2014)

# Deep Dive Technical Course

- <http://nikoladimitroff.github.io/Game-Engine-Architecture>
- <https://www.blaenkdenum.com/notes/unreal-engine>
- <https://jip.dev/notes/unreal-engine/>
- <https://github.com/jbtronics/UE4-Cheatsheet>

## Modules

**Types:**

- Developer: Used by Editor & Programs, but not games
- Editor: Used by UnrealEditor Only
- Runtime: Used by Editor, Games, & Programs
- ThirdParty: External third party libs/code
- Plugins: Extensions for Editor and/or Games. Should not have dependencies on other plugins
- Programs: Standalone apps & tools

**Important Modules:**

- Core: fundamental types & functions
- CoreUObject: Implements Uobject reflection system
- Engine: Game & engine framework classes
- OnlineSubsystem: Online & social networking features
- Slate: Widget library & high level UI functionality

**Modules for Advanced Functionality:**

- DesktopPlatform: Modules for OS function calls (e.g. filesystem, etc)
- DetailCustomization: Editor Detail panel customizations
- Launch: Main loop classes & functions
- Messaging: Message passing subsystem
- Sockets: Network socket implementation
- Settings: Editor & Project settings API
- SlateCore: Low level UI functionality
- TargetPlatform: Platform abstraction layer
- UMG: WYSIWYG UI system (Unreal Motion Graphics)
- UnrealEd: Unreal Editor main frame & features
- Analytics: Analytics functionality
- AssetRegistry: Asset database functionality for UnrealEd
- JsonUtilities & XmlParser: Parsing json/xml files

# Rendering

- How Unreal Renders A Frame: <https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame>
- UE4 Rendering Overview: <https://medium.com/@lordned/unreal-engine-4-rendering-overview-part-1-c47f2da65346>
- <http://gregory-igehy.hatenadiary.com/entry/2018/02/24/023251>
- <http://gregory-igehy.hatenadiary.com/entry/2017/12/28/002645>

# Python

## Fast ramp up to UE4Python

These documents are great starting points which are located in the [UE4 Python Repo](https://github.com/kitelightning/UnrealEnginePython/)

- README.MD
- UOBJECT_API.md
- SnippetsForStaticAndSkeletalMeshes.md
- Examples\\\*.md

## How to find stuff

- Look at the \*Snippets.py files
- Use Find all in BuildAutomation vscode project
- Use Visual assist symbol search "py\_ ..."

## Useful Python Starting Points

- <https://www.toptal.com/python/top-10-mistakes-that-python-programmers-make>
- <http://book.pythontips.com/en/latest/index.html>
- <http://ozkatz.github.io/improving-your-python-productivity.html>
- <https://treyhunner.com/2019/05/python-builtins-worth-learning>
