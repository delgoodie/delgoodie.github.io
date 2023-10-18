---
sortIndex: 4
sidebar: ue4guide
---

Then learn how to add component of any class, once you learn that you will be able place any component you like. you do this:

In header:

```cpp
UPROPERTY()
UClassOfTheComponent\* Component;
```

Then you want to create component:

```cpp
Component = ConstructObject&lt;UClassOfTheComponent>(UClassOfTheComponent::StaticClass(), GetOwner(), NAME_None, RF_Transient);
```

Then set up variables that set up the component (in case of UChildActorComponent you set ChildActorClass which will spawn actor, or ChildActor if you got ready one). If you deal with SceneComponent you need to attach it to something, so you can copy rama code here:

Component->AttachTo(this->ShipMesh, primaryWeaponSlots\[i].socketName, EAttachLocation::SnapToTarget);

And with every component you need to register it

Component->RegisterComponent();

*Reference From <https://answers.unrealengine.com/questions/221783/add-child-actor-component-in-c.html>*

Spawn/respawn/Create/recreate/modify Component at runtime:

```cpp
void AMyComponentSpawner::PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent)
{
//Get all of our components
TArray&lt;UActorComponent\*> MyComponents;
GetComponents(MyComponents);

//Get the name of the property that was changed
FName PropertyName = (PropertyChangedEvent.Property != nullptr) ? PropertyChangedEvent.Property->GetFName() : NAME_None;

// We test using GET_MEMBER_NAME_CHECKED so that if someone changes the property name
// in the future this will fail to compile and we can update it.
if ((PropertyName == GET_MEMBER_NAME_CHECKED(AMyComponentSpawner, MyMesh)))
{
FMultiComponentReregisterContext ReregisterContext(MyComponents);

for (UActorComponent\* Comp : MyComponents)
{
if (UStaticMeshComponent\* MeshComp = Cast&lt;UStaticMeshComponent>(Comp))
{
MeshComp->SetStaticMesh(MyMesh); // Update the component to the new mesh
}
}
}

// Call the base class version
Super::PostEditChangeProperty(PropertyChangedEvent);
}
#endif
```

*Reference From <https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/GameFramework/AActor/PostEditChangeProperty>*
