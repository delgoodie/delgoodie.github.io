---
sidebar: ue4guide
---


==============================

UnrealEdSrv.cpp/EditorServer.cpp/UEditorEngine/UUnrealEdEngine is good place to see commands and what things need to be broadcast

- Also useful for most editor actions (like building map, rebuilding volumes, cooking, replaceactors, etc etc )

- edact prefixed things are editor actions (e.g. edactCopySelected, edactPasteSelected, etc)

ConvertLightActors/ReplaceActors/ConvertActors

SetPreviewMeshMode/UpdatePreviewMesh/CyclePreviewMesh

CopySelectedActorsToClipboard/PasteSelectedActorsFromClipboard

SyncBrowserToObjects/SyncToContentBrowser/SyncActorLevelsToContentBrowser

FindSelectedActorsInLevelScript

FEditorFileUtils::SaveCurrentLevel/GetDirtyWorldPackages/GetDirtyContentPackages/SaveDirtyPackages/SaveMap

EditorLevelUtils::MakeLevelCurrent( LevelToMakeCurrent );

UEditorLevelUtils::MoveSelectedActorsToLevel()

MoveActorInFrontOfCamera

MoveViewportCamerasToActor

MoveViewportCamerasToComponent

SnapObjectTo

MoveViewportCamerasToBox

GetPIEViewport/GetActiveViewport

SpawnPlayFromHereStart

ParseMapSectionIni/LoadMapListFromIni

PackageUsingExternalObjects

FindReferencesToActorFromLevelScript/ModifyActorReferencedGraphNodes

FindActorsThatReferenceActor/GetActorReferenceMap/ReplaceAllActorRefrences

**FBlueprintEditorUtils:** Lots of useful blueprint nodes

GetGraphFunctionMetaData/GetEntryNode/GetEntryAndResultNodes/CheckIfGraphHasLatentFunctions

OpenReparentBlueprintMenu

===========================

FSelfRegisteringExec for POD structs to register for console commands

FStringOutputDevice Archive;

const FExportObjectInnerContext Context;

UExporter::ExportToOutputDevice(&Context, Object, NULL, Archive, TEXT("copy"), 0, PPF_Copy | PPF_DebugDump, false);

Archive.Log(TEXT("\\r\\n\\r\\n"));

FString ExportedText;

ExportedText = Archive;

**========================================**

AHUD::HandleBugScreenShot() override for bug screenshots and use bugscreenshotwithhudinfo to screenshot

**==================**

GetCurrentViewportPlacementTransform

Add Slate Window To Game:

​ FSlateApplication::Get().AddWindow

​ (

​ SNew(SWindow)

​ .ClientSize(FVector2D(400,600))

​ .Title( FText::FromString( Object->GetName() ) )

​ \[

​ SNew(SBorder)

​ .BorderImage(FEditorStyle::GetBrush("ToolPanel.GroupBorder"))

\[

​ SNew(SVerticalBox)

​ +SVerticalBox::Slot()

​ .FillHeight(1)

​ \[

​ DetailsView.ToSharedRef()

​ ]

​ ]

​ ]

​ );

Create Detail View of all properties in an object:

FPropertyEditorModule& PropertyModule = FModuleManager::LoadModuleChecked&lt;FPropertyEditorModule>("PropertyEditor");

​ TSharedPtr&lt;IDetailsView> DetailsView = PropertyModule.CreateDetailView(Args);

​ DetailsView->SetIsPropertyVisibleDelegate(FIsPropertyVisible::CreateStatic(&Local::IsPropertyVisible, bShouldShowNonEditable));

​ DetailsView->SetObject(Object);

​

​ // create Slate property window

​ FSlateApplication::Get().AddWindow

​ (

​ SNew(SWindow)

​ .ClientSize(FVector2D(400,600))

​ .Title( FText::FromString( Object->GetName() ) )

​ \[

​ SNew(SBorder)

​ .BorderImage(FEditorStyle::GetBrush("ToolPanel.GroupBorder"))

​ \[

​ SNew(SVerticalBox)

​ +SVerticalBox::Slot()

​ .FillHeight(1)

​ \[

​ DetailsView.ToSharedRef()

​ ]

​ ]

​ ]

​ \\;

================

// HOW TO GET AN IN-DEPTH PERFORMANCE ANALYSIS OF SLATE

//

// Step 1)

// Set WITH_VERY_VERBOSE_SLATE_STATS to 1.

//

// Step 2)

// When running the game (outside of the editor), run these commandline options

// in order and you'll get a large dump of where all the time is going in Slate.

//

// stat group enable slateverbose

// stat group enable slateveryverbose

// stat dumpave -root=stat_slate -num=120 -ms=0
