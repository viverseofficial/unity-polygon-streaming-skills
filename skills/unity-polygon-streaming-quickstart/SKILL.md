---
name: unity-polygon-streaming-quickstart
description: Install the Viverse Polygon Streaming Unity SDK and get an XRG model streaming into a scene. Covers .tgz install, StreamController + StreamingModel wiring, the PolygonStreaming layer, and prefabs.
prerequisites: [Unity 2021+, an XRG model URL from https://stream.viverse.com/console, a scene camera]
tags: [polygon-streaming, viverse, unity, xrg, streaming, quickstart, setup]
---

# Polygon Streaming Unity SDK — Quick Start

Use this skill to bring the Polygon Streaming Unity SDK into a project for the first time, or to wire up a fresh scene. Once a model streams, jump to the domain-specific skill (`static-models`, `animated-glb`, or `vrm`).

## When To Use This Skill

- Fresh install of `com.viverse.polygon-streaming` into a Unity project.
- Setting up the scene-level `StreamController` for the first time.
- Debugging "the model never appears" symptoms.
- Adding the required `PolygonStreaming` layer.

## Prerequisites

- Unity 2021 or newer (BRP / URP / HDRP all supported).
- The SDK `.tgz` file (downloaded from the Viverse Polygon Streaming product page).
- An XRG model URL from <https://stream.viverse.com/console>.
- A scene camera (used for LOD & occlusion decisions).

## Preflight Checklist

- [ ] Unity project is 2021+.
- [ ] `.tgz` file is downloaded locally (do **not** copy the SDK folder into `Assets/` — install as a package).
- [ ] A scene camera exists (otherwise SDK falls back to `Camera.main` with a warning).
- [ ] Editor is not in Play mode when adding the package.

## Mandatory Compliance Gates (MUST PASS)

1. **MUST** install via **Package Manager → Add package from tarball…**. Manually copying the SDK into `Assets/` breaks assembly resolution and shader inclusion.
2. **MUST** have exactly **one** `StreamController` per scene. Multiple controllers are unsupported and will fight for the shared occlusion state.
3. **MUST** create a layer named exactly **`PolygonStreaming`** (case-sensitive) in **Edit → Project Settings → Tags and Layers** when occlusion culling is enabled (the default). The Editor auto-adds this via `AutoAddLayerOnLoad`, but verify it in fresh projects.
4. **MUST** assign the `StreamController` reference on every `StreamingModel`. It is **not** discovered automatically.
5. **MUST NOT** hand-create `Assets/Resources/SosCacheSettings.asset`. The SDK creates it on first use.
6. **MUST NOT** rename or move the `PolygonStreaming` layer — the runtime hard-codes the name.

## Installation

1. **Window → Package Manager**.
2. Click **+** → **Add package from tarball…**.
3. Select the downloaded `.tgz`.
4. Wait for dependency resolution. The package appears under **Packages → Polygon Streaming**.

Dependencies pulled in automatically:

| Package | Version | Purpose |
| --- | --- | --- |
| `com.unity.cloud.ktx` | 3.3.0 | KTX texture decode |
| `com.unity.burst` | 1.8.4 | Occlusion raycast perf |
| `com.unity.mathematics` | 1.2.6 | Math |
| `com.unity.collections` | 1.5.2 | Native containers |

### Optional: UniVRM (only for VRM avatars)

Skip for GLB/FBX. Required only for VRMA animation playback and MToon shading. Install both via **Package Manager → Add package from git URL**:

```
https://github.com/vrm-c/UniVRM.git?path=/Assets/UniGLTF#v0.130.1
https://github.com/vrm-c/UniVRM.git?path=/Assets/VRM10#v0.130.1
```

## Scene Setup — Option A (Prefabs, Recommended)

1. In **Package Manager → Polygon Streaming**, expand **Samples → Prefabs** and click **Import**.
2. Drag the **Stream Controller** prefab into the scene. In the Inspector, assign your scene camera to **Viewer Camera**.
3. Drag the **Streaming Model** prefab into the scene. Position/rotate/scale its Transform where you want the model. Assign:
   - **Stream Controller** → the prefab from step 2.
   - **Source Url** → your XRG URL.
4. Press **Play**.

## Scene Setup — Option B (Manual)

1. Create an empty GameObject named `StreamController`, add the `StreamController` component, assign **Viewer Camera**.
2. Create an empty GameObject where the model should appear, add the `StreamingModel` component, assign the `StreamController` reference and paste the **Source Url**.
3. Press **Play**. Loading starts automatically on the first `Update`; no `Load()` call is needed.

## StreamController — Key Inspector Fields

Under **Stream Controller Monobehaviour Configuration**:

| Field | Default | Description |
| --- | --- | --- |
| **Viewer Camera** | — | Camera used for LOD + occlusion. Falls back to `Camera.main` with a warning if unset. |
| **Triangle Budget** | 3,000,000 | Global triangle cap across all streamed models. Set to ≥ 30% of the total polys you plan to load. |
| **VRAM Budget (MB)** | 0 | GPU memory cap. `0` = unlimited. |
| **Distance Factor** | 1.1 | `>1` favors nearby geometry; `<1` favors distant. |
| **Close Up Distance** | 3 | Distance (m) at which Close-Up Distance Factor activates. `0` disables. |
| **Close Up Distance Factor** | 5 | Distance factor when camera is closer than Close-Up Distance. |
| **Distance Type** | BoundingBox | `BoundingBox` (nearest edge) or `BoundingBoxCentre`. Use `BoundingBoxCentre` for walkable environments. |
| **Occlusion Culling** | `true` | Raycast-based dynamic culling. Requires the `PolygonStreaming` layer. |
| **Time Slicing** | 120 | Frames occluded before hiding. |
| **Rays Per Frame** | 256 | Occlusion rays per frame. |
| **Use Bbox Only Partitions** | `true` | Keeps objects visible when the camera enters their bounds. |
| **Bbox Only Partition Padding** | 0.004 | Bounding box padding (m). |
| **Leaf Distance Threshold** | 10 | Distance (m) at which leaf-based distance calc kicks in. |

## StreamingModel — Key Inspector Fields

| Field | Default | Description |
| --- | --- | --- |
| **Stream Controller** | — | Required. |
| **Source Url** | — | XRG URL. |
| **Hash Code** | — | Optional cache validation hash. |
| **Light Probe Usage** | `false` | Unity light probe sampling. |
| **Custom Materials** | `false` | Override default materials (see the static-models skill). |
| **Quality Priority** | 1 | Relative quality weight vs other models sharing the budget. |
| **Animation Mode** | `Auto` | `Auto` / `EmbeddedClips` / `VrmMecanim` / `Vrma` / `None`. See the animated-glb / vrm skills. |
| **Custom Animator Controller** | — | `RuntimeAnimatorController` for Mecanim. |
| **Auto Play Loop** | `true` | Auto-loops embedded clips. Disabled automatically when Custom Animator Controller is set. |

## Minimal Verification Script

```csharp
using UnityEngine;

public class StreamingSanityCheck : MonoBehaviour
{
    public StreamingModel streamingModel;

    void Start()
    {
        streamingModel.SetLoadedCallback(() => Debug.Log("[PS] Model loaded"));
        streamingModel.SetLoadFailedCallback(ex => Debug.LogError($"[PS] {ex.Message}"));
    }
}
```

If nothing logs, work down the Gotchas table below before moving to a domain skill.

## Gotchas & Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Model never appears, no errors | `Stream Controller` reference is null on the `StreamingModel` | Assign it in the Inspector — auto-discovery does not happen. |
| Warning: "viewerCamera must be set" | **Viewer Camera** empty on the controller | Assign the scene camera. |
| Error: "PolygonStreaming layer not found" | Layer not created | Add the `PolygonStreaming` layer in **Project Settings → Tags and Layers**. |
| Model disappears unexpectedly | Occlusion culling too aggressive | Raise **Time Slicing** or disable occlusion culling. |
| Model appears at wrong position | Non-zero Transform on the `StreamingModel` GameObject | The `StreamingModel`'s Transform **defines** placement — reset it if needed. |
| Model looks low-poly and never improves | Triangle budget too small or too many competing models | Raise **Triangle Budget** on the controller, or increase **Quality Priority** on this model. |
| WebGL build has invisible textures | KTX or MToon shaders stripped | Check **Project Settings → Graphics → Always Included Shaders**. |

## Next Steps

- Static (non-animated) models → `unity-polygon-streaming-static-models`.
- GLB/FBX with animation clips → `unity-polygon-streaming-animated-glb`.
- VRM 1.0 avatars → `unity-polygon-streaming-vrm`.

## References

- Package: `com.viverse.polygon-streaming` (`package.json` in the SDK)
- In-package Gitbook: `Gitbook/PolygonStreamingUnitySDK.md`, `Gitbook/SubPage1.md`, `Gitbook/SubPage2.md`, `Gitbook/SubPage3.md`
- Model console: <https://stream.viverse.com/console>
