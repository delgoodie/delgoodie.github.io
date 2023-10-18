---
sortIndex: 5
sidebar: ue4guide
---

The main idea here is that we want to move the SkeletalMesh render data into the Derived Data Cache (DDC) in a similar way to how StaticMesh works. This makes it much easier to make changes to the render buffer layout. To this end we split FSkeletalMeshResource (which was both the 'source' and the 'derived' data) into FSkeletalMeshModel (source data, only available in editor builds) and FSkeletalMeshRenderData (the actual data needed at runtime for rendering). For each LOD we now have a FSkeletalMeshLODModel (editor only data, within the FSkeletalMeshModel) and a FSkeletalMeshLODRenderData (derived render data, within the FSkeletalMeshRenderData). These are the replacement for the old, confusingly named FStaticLODModel. The derived data is transient - it is re-derived any time the source data changes.

So when you are updating your code, the main question is - "is this runtime or editor time code". If you want to work on the source data in the editor, you need to work on FSkeletalMeshModel. If your code is supposed to run in cooked game builds, then you can only work with FSkeletalMeshRenderData. So for example, the mesh merging code all works with FSkeletalMeshRenderData because it has to work at runtime. All the editor code now operates on FSkeletalMeshModel, and triggers a rebuild of the derived data so you can see the results (you can see USkeletalMesh::PostEditChangeProperty calls InvalidateRenderData which updates the GUID on the source data and re-generates the derived data for rendering.

I hope that helps!

*Reference From <https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1446998-what-happened-to-uskeletalmesh-in-ue-4-19>*
