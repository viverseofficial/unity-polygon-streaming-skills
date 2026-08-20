---
name: unity-polygon-streaming-vrm
description: Stream VRM 1.0 avatars with the Polygon Streaming Unity SDK — expressions, look-at, spring bones, VRMA animation, Mecanim humanoid retargeting, MToon shading, first-person filtering.
prerequisites: [com.viverse.polygon-streaming installed, StreamController in scene, an XRG derived from a VRM 1.0 source, UniVRM optional but required for VRMA and MToon]
tags: [polygon-streaming, viverse, unity, xrg, vrm, avatar, univrm, mtoon, vrma, spring-bone, mecanim]
---

# Polygon Streaming Unity SDK — VRM Avatar Streaming

Use this skill when the streamed XRG was converted from a **VRM 1.0** source. VRM avatars retain full expressions, look-at eye tracking, spring bone physics, humanoid bone access, VRMA playback, Mecanim humanoid retargeting, first-person mesh filtering, and MToon toon shading.

## When To Use This Skill

- Streaming a VRM avatar into a Unity scene.
- Driving expressions (blink, happy, visemes) from gameplay or lip-sync.
- Enabling spring bone physics for hair / clothing / accessories.
- Making the avatar look at a camera / target.
- Playing VRMA (`.vrma`) or Mecanim humanoid animations.
- Rendering MToon materials correctly (BRP or URP).
- Setting up first-person camera mesh filtering.

## Prerequisites

- Package `com.viverse.polygon-streaming` installed.
- One `StreamController` in the scene with **Viewer Camera** assigned (see the quickstart skill).
- XRG URL from a VRM 1.0 source.
- **Optional but recommended: UniVRM** — required for VRMA playback and correct MToon rendering:
  ```
  https://github.com/vrm-c/UniVRM.git?path=/Assets/UniGLTF#v0.130.1
  https://github.com/vrm-c/UniVRM.git?path=/Assets/VRM10#v0.130.1
  ```

### Feature matrix (with vs without UniVRM)

| Feature | Without UniVRM | With UniVRM |
| --- | --- | --- |
| Expressions / blend shapes | ✅ | ✅ |
| Look-at eye tracking | ✅ | ✅ |
| Spring bone physics | ✅ standalone impl | ✅ native |
| Humanoid bone access | ✅ | ✅ |
| Mecanim humanoid animation | ✅ | ✅ |
| Node constraints | ✅ | ✅ |
| First-person mesh filtering | ✅ | ✅ |
| VRMA animation loading | ❌ | ✅ |
| VRMA expression / look-at curves | ❌ | ✅ |
| MToon toon shading | ⚠️ falls back to unlit | ✅ MToon10 |

## Mandatory Compliance Gates (MUST PASS)

1. **MUST** guard every VRM API call behind `SetLoadedCallback` **and** `if (streamingModel.IsVrm)`. Calling before load or on a non-VRM model is a no-op or throws.
2. **MUST NOT** manually add `Vrm*Controller` components — the SDK adds them automatically on load.
3. **MUST NOT** modify `Vrm10Instance.UpdateType` from user code — the SDK owns it via Animation Mode (`DisableControlRig` / `EnableControlRig` in the internal `UniVrmSetup`).
4. **MUST** use `AnimationMode.Auto` unless there is a specific reason to force a mode. `Auto` selects `VrmMecanim` when a controller is assigned and `None` (VRMA-ready) otherwise.
5. **MUST** set the controller on the `StreamingModel` **Custom Animator Controller** field (or via the property/method). Do **not** wire the controller directly on the auto-created `Animator`.
6. **MUST** install UniVRM for any VRMA (`.vrma`) usage — `LoadVrmaFromUrlAsync` returns `false` with a warning otherwise.
7. **MUST** ensure MToon shaders are not stripped from the build (see the Troubleshooting table).

## Scene Setup

Identical to `unity-polygon-streaming-quickstart` — one `StreamController` and a `StreamingModel` pointing at the VRM-derived XRG URL. Do not add UniVRM components manually; the SDK initializes them on load.

## Animation Modes

Set via Inspector or `SetAnimationMode` at runtime.

| Mode | Meaning |
| --- | --- |
| `Auto` (default) | SDK picks based on model type + controller. Recommended. |
| `EmbeddedClips` | Play embedded clips via `StreamingModelAnimator`. |
| `VrmMecanim` | Unity Mecanim humanoid animation with the assigned `RuntimeAnimatorController`. |
| `Vrma` | VRMA playback via ControlRig. Requires UniVRM. |
| `None` | No bone animation. |

### Auto-detection matrix

| Model | Controller assigned? | Humanoid bones? | `Auto` resolves to |
| --- | --- | --- | --- |
| VRM | Yes | (always) | `VrmMecanim` |
| VRM | No | (always) | `None` (ready for VRMA) |
| Non-VRM | Yes | Yes | `VrmMecanim` |
| Non-VRM | Yes | No | `EmbeddedClips` |
| Non-VRM | No | — | `EmbeddedClips` |

Expressions (blink, happy, visemes) work in **every** mode — they touch blend shapes only.

## Mecanim (RuntimeAnimatorController)

### Simplest path (no scripting)

1. Assign a humanoid `RuntimeAnimatorController` (e.g. Mixamo-retargeted) to **Custom Animator Controller**.
2. Leave **Animation Mode** at `Auto`.
3. Press Play. The SDK creates the humanoid avatar, manages ControlRig, and applies the controller.

### From script

```csharp
streamingModel.RuntimeAnimatorController = myController;
// or:
streamingModel.SetRuntimeAnimatorController(myController);

// Force a specific mode:
streamingModel.SetAnimationMode(AnimationMode.VrmMecanim);

// Read the currently active mode (after Auto resolution):
AnimationMode active = streamingModel.ActiveAnimationMode;
```

### Non-VRM humanoid GLBs

If the source GLB uses **Mixamo / Unity standard / Blender** bone naming, the SDK auto-detects it and creates a humanoid avatar. Same code path — just assign a controller and use `Auto`.

## Look-At API

```csharp
streamingModel.SetVrmLookAtTarget(Camera.main.transform);   // per-frame tracking
streamingModel.SetVrmLookAt(new Vector3(0, 1.6f, 3f));      // one-shot world position
streamingModel.SetVrmLookAtYawPitch(15f, -5f);              // manual yaw/pitch (deg)
streamingModel.ResetVrmLookAt();
```

Setting a target enables automatic per-frame tracking. One-shot calls clear the target for that frame.

## Expression API

Weights are `0.0`–`1.0`.

```csharp
streamingModel.SetVrmHappy(0.8f);
streamingModel.SetVrmAngry(0.5f);
streamingModel.SetVrmSad(0.3f);
streamingModel.SetVrmSurprised(1.0f);

streamingModel.SetVrmBlink(1.0f);

// Visemes for lip-sync
streamingModel.SetVrmAa(0.5f);
streamingModel.SetVrmIh(0.5f);
streamingModel.SetVrmOu(0.5f);
streamingModel.SetVrmEe(0.5f);
streamingModel.SetVrmOh(0.5f);

// Generic
streamingModel.SetVrmExpression("relaxed", 0.7f);
float w = streamingModel.GetVrmExpression("happy");
foreach (string n in streamingModel.GetVrmAvailableExpressions()) Debug.Log(n);
streamingModel.ResetVrmExpressions();
```

## Spring Bone API

```csharp
streamingModel.SetVrmSpringBonesEnabled(true);
streamingModel.ResetVrmSpringBones();   // after teleporting
```

### Physics override (multipliers vs original VRM values)

By default, spring bones use the **original VRM parameters** for accurate simulation. To customize:

```csharp
streamingModel.vrmSettings.overrideSpringBonePhysics = true;
streamingModel.vrmSettings.springBoneStiffnessMultiplier = 2.0f;
streamingModel.vrmSettings.springBoneGravityMultiplier  = 1.5f;
streamingModel.vrmSettings.springBoneDragMultiplier     = 0.5f;
streamingModel.vrmSettings.springBoneExternalForce      = new Vector3(0.3f, 0, 0);
streamingModel.ApplySpringBonePhysicsOverride();

// Revert:
streamingModel.vrmSettings.overrideSpringBonePhysics = false;
streamingModel.ApplySpringBonePhysicsOverride();
```

Direct gravity override:
```csharp
streamingModel.SetVrmSpringBoneGravityPower(0.2f);
```

Fine-grained via the auto-added controller:
```csharp
var sb = streamingModel.GetComponent<VrmSpringBoneController>();
sb?.ApplyPhysicsMultipliers(2.0f, 0.5f, 1.5f, new Vector3(0.3f, 0, 0));
sb?.RevertToOriginalPhysics();
```

## Humanoid Bone Access

```csharp
bool ok = streamingModel.CreateVrmHumanoidAvatar("MyAvatar");
Animator a = streamingModel.Animator;

Transform rightHand = streamingModel.GetVrmBone(HumanBodyBones.RightHand);
Transform head       = streamingModel.GetVrmBone(HumanBodyBones.Head);

// Lower-level:
Sos.VrmInstance vrm = streamingModel.VrmInstance;
Transform hips  = vrm?.HipsBone;
Transform spine = vrm?.GetHumanBone(HumanBodyBones.Spine);
```

## VRMA Animation (requires UniVRM)

```csharp
// Load
bool ok = await streamingModel.LoadVrmaFromUrlAsync("https://example.com/dance.vrma");
bool ok2 = await streamingModel.LoadVrmaFromFileAsync(Application.streamingAssetsPath + "/wave.vrma");
bool ok3 = await streamingModel.LoadVrmaFromBytesAsync(bytes);

// Playback
streamingModel.PlayVrma();
streamingModel.PauseVrma();
streamingModel.ResumeVrma();
streamingModel.StopVrma();
streamingModel.UnloadVrma();

// Settings
streamingModel.SetVrmaLoop(true);
streamingModel.SetVrmaSpeed(1.5f);
streamingModel.SetVrmaBlendWeight(0.8f);

// Time
streamingModel.SetVrmaTime(2.5f);
streamingModel.SetVrmaNormalizedTime(0.5f);

// State
bool loaded  = streamingModel.IsVrmaLoaded;
bool playing = streamingModel.IsVrmaPlaying;
float dur    = streamingModel.GetVrmaDuration();
float t      = streamingModel.GetVrmaTime();
float p      = streamingModel.GetVrmaNormalizedTime();

// Crossfade
streamingModel.CrossFadeVrma(0.3f);
```

`SetAnimationMode(AnimationMode.Vrma)` switches automatically when `LoadVrma*` succeeds; you can also call it explicitly.

## First-Person Mesh Filtering

```csharp
var fp = streamingModel.GetComponent<VrmFirstPersonController>();
if (fp != null && fp.IsInitialized)
{
    fp.Setup(firstPersonCamera, firstPersonLayer: 10, thirdPersonLayer: 11);
    // fp.Teardown(); // restores original layers
}
```

Configure your first-person camera's culling mask to **exclude** `thirdPersonLayer`, and your third-person camera to **exclude** `firstPersonLayer`.

## MToon Material Support

- The build preprocessor `MToonShaderIncluder` auto-adds MToon shaders. In most cases no manual work is required.
- BRP shader: `VRM10/MToon10`.
- URP shader: `VRM10/Universal Render Pipeline/MToon10`.
- Without UniVRM installed, models using MToon fall back to unlit rendering.

## Auto-Added Component Reference

Added to the `StreamingModel` GameObject on load — use `GetComponent<T>()` for advanced access:

| Component | Purpose |
| --- | --- |
| `VrmLookAtController` | Per-frame look-at target tracking. |
| `VrmExpressionController` | Blend-shape expression management. |
| `VrmSpringBoneController` | Hair / clothing / accessory physics. |
| `VrmHumanoidMapper` | Humanoid Avatar construction. |
| `VrmaController` | VRMA loading + playback (created on first `LoadVrma*`). |
| `VrmConstraintController` | Node constraints (aim, roll, twist). |
| `VrmFirstPersonController` | First-person mesh filtering. |

## Complete VRMA Example

```csharp
using UnityEngine;

public class VrmAvatarController : MonoBehaviour
{
    public StreamingModel avatar;
    public Transform lookTarget;
    public string vrmaUrl = "https://example.com/idle.vrma";

    void Start() => avatar.SetLoadedCallback(OnLoaded);

    async void OnLoaded()
    {
        if (!avatar.IsVrm) return;

        avatar.SetVrmLookAtTarget(lookTarget);
        avatar.SetVrmHappy(0.3f);

        if (await avatar.LoadVrmaFromUrlAsync(vrmaUrl))
        {
            avatar.SetVrmaLoop(true);
            avatar.PlayVrma();
        }
    }

    void Update()
    {
        if (!avatar.IsVrm) return;
        float blink = Mathf.PingPong(Time.time * 3f, 1f) > 0.9f ? 1f : 0f;
        avatar.SetVrmBlink(blink);
    }

    // Hook into an audio system for lip-sync
    public void OnVisemeUpdate(float aa, float ih, float ou, float ee, float oh)
    {
        avatar.SetVrmAa(aa); avatar.SetVrmIh(ih);
        avatar.SetVrmOu(ou); avatar.SetVrmEe(ee); avatar.SetVrmOh(oh);
    }
}
```

## Verification Checklist

- [ ] `streamingModel.IsVrm` is `true` after load.
- [ ] All VRM API calls occur inside `SetLoadedCallback`.
- [ ] For VRMA: both `com.vrmc.gltf` and `com.vrmc.vrm` packages are installed.
- [ ] Custom Animator Controller (if used) is assigned to **StreamingModel**, not directly to the runtime `Animator`.
- [ ] `Animation Mode` stays at `Auto` unless explicitly forced.
- [ ] MToon models render with correct toon shading (BRP or URP shader present in Always Included Shaders).
- [ ] For first-person mode: layer masks configured on both cameras.

## Common Mistakes

### ❌ `VrmMecanim` without a controller
```csharp
streamingModel.SetAnimationMode(AnimationMode.VrmMecanim); // no controller → avatar stays in rest pose
```
Use `Auto` or assign a controller before forcing the mode.

### ❌ VRM API before load
```csharp
void Start() { streamingModel.SetVrmHappy(1.0f); }  // no-op — VrmInstance is null
```
Wrap in `SetLoadedCallback(() => { if (streamingModel.IsVrm) streamingModel.SetVrmHappy(1.0f); });`.

## Gotchas & Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| VRM API has no effect | Model not yet loaded | Use `SetLoadedCallback`. |
| `IsVrm` returns false | Source was not VRM | The XRG must be derived from a VRM source. |
| Expressions do nothing | Model has no expression data | Call `GetVrmAvailableExpressions()` to verify. |
| Spring bones frozen | Disabled | Call `SetVrmSpringBonesEnabled(true)` or enable in Inspector. |
| Spring bone physics feels wrong | Override enabled with non-default multipliers | Set `overrideSpringBonePhysics = false` and call `ApplySpringBonePhysicsOverride()`. |
| VRMA calls return `false` / warn | UniVRM not installed | Install both `com.vrmc.gltf` and `com.vrmc.vrm`. |
| Look-at not updating | Target not set | `SetVrmLookAtTarget(yourTransform)` after load. |
| Humanoid avatar creation fails | Hips bone missing | Verify complete humanoid skeleton on the source VRM. |
| `VrmMecanim` not activating | No controller or non-standard bone naming | Assign a `RuntimeAnimatorController`; for non-VRM humanoids use Mixamo/Unity/Blender naming. |
| Face frozen while body animates | Expressions not driven | Expressions require explicit `SetVrm*` calls even in `VrmMecanim` mode. |
| Model renders white / unlit despite UniVRM | MToon shader stripped from build | Add `VRM10/MToon10` (BRP) or `VRM10/Universal Render Pipeline/MToon10` (URP) to **Project Settings → Graphics → Always Included Shaders**. For WebGL, check shader stripping settings. |

## References

- In-package doc: `USAGE_VRM_STREAMING.md`
- In-package Gitbook: `Gitbook/VRM Avatar Streaming.md`, `Gitbook/SubPage6.md` (if present)
- Runtime source: `Runtime/Scripts/StreamingModel.cs`, `Runtime/Scripts/VrmSettings.cs`, `Runtime/PolygonStreaming/Source/Vrm/*`
- UniVRM: <https://github.com/vrm-c/UniVRM>
