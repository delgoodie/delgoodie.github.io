---
sortIndex: 11
sidebar: ue4guide
---

Create a new package for the new asset:

1. UPackage\* NewPackage = CreatePackage(nullptr, NewPackageName);

1. Than duplicate the existing asset so that its Outer is the NewPackage:


1. UObject\* NewObject = StaticDuplicateObject(OldObject, NewPackage);

1. Than make any changes you want to NewObject and save the new package with:


1. SavePackageHelper(NewPackage, NewPackageName);

   *Reference From <https://udn.unrealengine.com/questions/366402/how-can-i-write-fassetdata-to-the-hard-disk.html>*

[The Cook, The Resave, His Garbage And Her Optimization – Unreal ...](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=8&ved=0ahUKEwjtx5j3pO3YAhVN8GMKHbeEAoMQFghVMAc&url=https%3A%2F%2Fcoconutlizard.co.uk%2Fnew%2Fprogramming%2Fthe-cook-the-resave-his-garbage-and-her-optimization%2F&usg=AOvVaw2DkrYGi8a6hcQIued0ssJ5)

*Reference From <https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=8&ved=0ahUKEwjtx5j3pO3YAhVN8GMKHbeEAoMQFghVMAc&url=https%3A%2F%2Fcoconutlizard.co.uk%2Fnew%2Fprogramming%2Fthe-cook-the-resave-his-garbage-and-her-optimization%2F&usg=AOvVaw2DkrYGi8a6hcQIued0ssJ5>*
