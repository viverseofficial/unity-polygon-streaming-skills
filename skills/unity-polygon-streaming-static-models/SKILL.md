---
name: unity-polygon-streaming-static-models
description: Stream static (non-animated) GLB / FBX models with the Viverse Polygon Streaming Unity SDK — StreamController + StreamingModel wiring, custom materials, runtime URL swap, collider mesh fetch, add/remove events.
prerequisites: [Unity 2021+, com.viverse.polygon-streaming installed, StreamController in the scene, XRG model URL]
tags: [polygon-streaming, viverse, unity, xrg, static, streaming, glb, fbx]
---

# Polygon Streaming Unity SDK — Static Models

Use this skill for streaming static (non-animated) models — architectural / product / engineering GLB & FBX converted to XRG. For scene / install wiring, use `unity-polygon-streaming-quickstart` first.

## When To Use This Skill

- Streaming a static XRG model into a scene.
- Overriding the SDK's default opaque / transparent materials.
- Swapping the streamed model URL at runtime.
- Fetching a collider mesh for physics / raycasting.
- Subscribing to scene-level model add / remove events.

## Prerequisites

- Package `com.viverse.polygon-streaming` installed (see the quickstart skill).
- One `StreamController` in the scene with **Viewer Camera** assigned.
- Layer `PolygonStreaming` present when occlusion culling is enabled.
- XRG URL from <https://stream.viverse.com/console>.

## Mandatory Compliance Gates (MUST PASS)

1. **MUST** assign the `StreamController` reference on the `StreamingModel` component. Auto-discovery does not happen.
2. **MUST NOT** call `Load()` — no such method. Loading starts automatically on the first `Update` after the component is enabled.
3. **MUST NOT** offset the streamed geometry by moving child transforms — the `StreamingModel` GameObject's own Transform is the model root.
4. **MUST** register `SetLoadedCallback` **before** enabling the component if you depend on load timing.
5. To re-load a new URL, **MUST** toggle `enabled = false → set sourceUrl → enabled = true`. Setting `sourceUrl` alone does not restart the load.

## StreamingModel — Field Reference

Inspector-visible public fields (assign in Editor or via code):

| Field | Type | Default | Purpose |
| --- | --- | --- | --- |
| `streamController` | `StreamController` | — | Required scene-level manager. |
| `sourceUrl` | `string` | — | XRG URL. |
| `hashCode` | `string` | — | Optional cache validation hash. |
| `lightProbeUsage` | `bool` | `false` | Sample Unity light probes on the streamed mesh. |
| `customMaterials` | `bool` | `false` | Override default materials with your own. |
| `material` | `Material` | — | Opaque override (used when `customMaterials` is true). |
| `transparentMaterial` | `Material` | — | Transparent override. |
| `qualityPriority` | `int` | 1 | Relative quality weight vs other models. |

## Minimum Working Example

```csharp
using UnityEngine;

public class StaticModelExample : MonoBehaviour
{
    public StreamingModel streamingModel;

    void Start()
    {
        streamingModel.SetLoadedCallback(OnLoaded);
        streamingModel.SetLoadFailedCallback(OnFailed);
    }

    void OnLoaded()
    {
        Debug.Log("[PS] Ready");
    }

    void OnFailed(System.Exception ex)
    {
        Debug.LogError($"[PS] {ex.Message}");
    }
}
```

The model begins streaming automatically on the first frame after the component is enabled. Do **not** call `Load()` — there is no such method.

## Custom Materials

Toggle **Custom Materials** and assign an opaque and/or transparent material.

```csharp
streamingModel.customMaterials = true;
streamingModel.material = myOpaqueMaterial;
streamingModel.transparentMaterial = myTransparentMaterial;
```

Set these **before** the model loads; changing them after load requires a full disable/enable cycle.

## Swapping the URL at Runtime

```csharp
streamingModel.enabled = false;                     // unregister + discard current geometry
streamingModel.sourceUrl = "https://example.com/new.xrg";
streamingModel.enabled = true;                      // re-register + start fresh load
```

Just reassigning `sourceUrl` while the component is enabled has no effect.

## Changing the Viewer Camera at Runtime

```csharp
streamController.streamControllerMonobehaviourConfiguration.viewerCamera = myNewCamera;
```

The frustum cache is cleared automatically.

## Listening for Model Registration

```csharp
streamController.OnAddStreamingModel    += m => Debug.Log($"[PS] +{m.sourceUrl}");
streamController.OnRemoveStreamingModel += m => Debug.Log($"[PS] -{m.sourceUrl}");
```

Useful for scene-wide LOD / analytics tracking.

## Fetching a Collider Mesh

Physics colliders are **not** created automatically. Fetch a mesh separately if you need one:

```csharp
async void SetupCollider()
{
    Mesh mesh = await StreamingModel.FetchColliderMesh("https://example.com/model.xrg");
    if (mesh != null)
    {
        var mc = gameObject.AddComponent<MeshCollider>();
        mc.sharedMesh = mesh;
    }
}
```

This is a **static** method — call it independently of any `StreamingModel` instance. The returned mesh is a lightweight collider representation, not the render mesh.

## Occlusion Culling Notes

- One `StreamingOcclusionCulling` instance per process, shared across all controllers.
- Destroying the last `StreamController` clears the shared culling state.
- Disable **Occlusion Culling** on the controller if the raycast cost is problematic on your target platform (e.g. very low-end mobile).

## Verification Checklist

- [ ] `SetLoadedCallback` fires within a few seconds of Play.
- [ ] Model appears at the `StreamingModel` GameObject's Transform (not at the world origin).
- [ ] `StreamController.Viewer Camera` is assigned — no warning in the Console.
- [ ] Layer `PolygonStreaming` exists when Occlusion Culling is enabled.
- [ ] Custom materials (if used) are set **before** the first load or before an enable-cycle re-load.

## Gotchas

| Symptom | Cause | Fix |
| --- | --- | --- |
| Model never appears | `streamController` reference is null | Assign it in the Inspector. |
| Model at world origin instead of GameObject position | Reading `streamingModel` transform from a child | Position the `StreamingModel` GameObject itself. |
| New URL has no effect | Set `sourceUrl` without toggling `enabled` | Cycle `enabled = false; sourceUrl = …; enabled = true;`. |
| Custom material ignored | Assigned after load | Enable **Custom Materials** and assign before load, or cycle `enabled`. |
| Warning: "viewerCamera must be set" | Camera not assigned | Assign to `StreamController`; the SDK falls back to `Camera.main` otherwise. |
| Model appears low-poly and never improves | Global triangle budget exhausted by other models | Raise `Triangle Budget` on `StreamController` or increase `qualityPriority`. |
| Raycasts hit nothing | `MeshCollider` never added | Use `StreamingModel.FetchColliderMesh` and attach it manually. |

## References

- In-package doc: `USAGE_STATIC_MODELS.md`
- In-package Gitbook: `Gitbook/SubPage3.md` (StreamingModel component), `Gitbook/SubPage4.md` (static models)
- Runtime source: `Runtime/Scripts/StreamController.cs`, `Runtime/Scripts/StreamingModel.cs`
- Model console: <https://stream.viverse.com/console>
