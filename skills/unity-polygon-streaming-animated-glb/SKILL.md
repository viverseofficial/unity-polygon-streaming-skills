---
name: unity-polygon-streaming-animated-glb
description: Play embedded animation clips from an XRG (source GLB/FBX) using StreamingModelAnimator with a custom RuntimeAnimatorController. Covers Animation Mode selection, dummy-clip name matching, PlayAnimation / triggers / floats / bools / ints, morph target weights, and runtime controller swap.
prerequisites: [com.viverse.polygon-streaming installed, StreamController in scene, an animated XRG with embedded clips]
tags: [polygon-streaming, viverse, unity, xrg, animation, animator, mecanim, glb]
---

# Polygon Streaming Unity SDK — Animated GLB Models

Use this skill for **non-VRM** animated models where clips are embedded in the source GLB/FBX. For humanoid retargeting of VRM avatars, use `unity-polygon-streaming-vrm` instead.

## When To Use This Skill

- Playing embedded animation clips from a streamed XRG.
- Driving a state machine with a custom `RuntimeAnimatorController`.
- Firing triggers / setting floats-bools-ints from gameplay code.
- Blending between states with cross-fade control.
- Debugging "wrong clip plays" or "animator does nothing" symptoms.

## How the Animation System Works (Critical Mental Model)

The SDK uses a **hybrid Legacy + Animator** pipeline:

- **Legacy `Animation` component** drives the skinned mesh visually.
- **`Animator` component** runs your state machine (transitions, parameters, blend trees).
- **`StreamingModelAnimator`** — added automatically when a custom controller is set — reads Animator state each `LateUpdate` and forwards state + clip weights to the Legacy component with smooth blending.

Reason: clips are loaded **at runtime** from the stream, so they cannot be baked into an Animator Controller asset at edit time.

## Animation Mode

The `StreamingModel.animationMode` enum:

| Mode | Behavior for animated GLB |
| --- | --- |
| `Auto` (default) | Uses `EmbeddedClips`. If a custom controller is assigned **and** the model has humanoid bones (Mixamo / Unity / Blender naming), switches to `VrmMecanim`. |
| `EmbeddedClips` | Force embedded-clip playback via `StreamingModelAnimator`. Standard path. |
| `VrmMecanim` | Force Unity Mecanim humanoid retargeting. Requires a humanoid skeleton and a humanoid `RuntimeAnimatorController`. |
| `None` | No bone animation. Rest pose only. |

Read / set at runtime:

```csharp
AnimationMode m = streamingModel.ActiveAnimationMode;
streamingModel.SetAnimationMode(AnimationMode.EmbeddedClips);
```

## Mandatory Compliance Gates (MUST PASS)

1. **MUST** call any `PlayAnimation` / `SetAnimation*` API **after** `SetLoadedCallback` fires. The Animator is only wired up after load.
2. **MUST** name each dummy clip **exactly** as it appears in the loaded model — otherwise the state has no visual output. Enable **Log Animations** to verify the real names.
3. **MUST NOT** rely on the content of dummy clips. They are placeholders; only their **name** matters.
4. **MUST NOT** leave **Auto Play Loop** on when using a custom controller — the SDK disables it automatically, but relying on it produces confusing behavior during migration.
5. **MUST** re-assign the controller via the property or method rather than editing `AnimatorController.runtimeAnimatorController` directly on the `Animator` — the SDK's hooks are the property setter and `SetRuntimeAnimatorController`.
6. **MUST** treat clip-name matching as case-sensitive.

## Discovering Clip Names

Enable **Log Animations** on the `StreamingModel`. On load, the Console prints every clip name and index:

```
[Polygon Streaming] Streaming Model : https://… Loaded 4 Clips:
  Index [0]: "Idle"
  Index [1]: "Walk"
  Index [2]: "Run"
  Index [3]: "Jump"
```

Use these names **verbatim** when creating dummy clips.

## Authoring the Animator Controller

1. **Create → Animator Controller** in Project window.
2. Open in Animator window, add states.
3. For each state, create a placeholder `AnimationClip` (**Create → Animation**) named **exactly** matching the loaded clip name (e.g. `"Walk"`). Assign it to the state's **Motion** field. Clip content is irrelevant.
4. Add parameters (Trigger / Float / Bool / Int) driving your transitions.

### Clip-name matching priority

1. **Exact name match** — dummy `"Walk"` → loaded `"Walk"`.
2. **Clean name match** — loaded names with a state-machine path prefix (e.g. `"Layer|Walk"`) are stripped to `"Walk"` before matching.
3. **Numeric index** — dummy clip named `"0"`, `"1"`, … maps to the clip at that load-order index.

Unmatched dummy clips produce no visual output; other mapped states continue to work.

### Morph target (blend shape) channels

If clips carry morph-target weight channels (blink, mouth shapes, expressions baked into the GLB), they play automatically alongside the skeletal pose. No extra setup — with or without a custom controller.

## Inspector Setup

On the `StreamingModel` GameObject:

- **Custom Animator Controller** → assign your controller.
- **Log Animations** → enable while developing to verify clip names.
- **Auto Play Loop** → leave alone; auto-disabled when a controller is set.

## Controlling Playback From Code

Wait for load first:

```csharp
using UnityEngine;

public class AnimatedCharacter : MonoBehaviour
{
    public StreamingModel streamingModel;
    public RuntimeAnimatorController controller;

    void Start()
    {
        streamingModel.customAnimatorController = controller;   // safe to set pre-load
        streamingModel.SetLoadedCallback(OnLoaded);
    }

    void OnLoaded()
    {
        streamingModel.PlayAnimation("Idle");
    }

    void Update()
    {
        if (!streamingModel) return;

        streamingModel.SetAnimationFloat("Speed", Input.GetAxis("Vertical"));

        if (Input.GetKeyDown(KeyCode.Space))
            streamingModel.SetAnimationTrigger("Jump");
    }
}
```

### API surface

```csharp
streamingModel.PlayAnimation("Run");
streamingModel.SetAnimationTrigger("Jump");
streamingModel.SetAnimationFloat("Speed", 3.5f);
streamingModel.SetAnimationBool("IsGrounded", true);
streamingModel.SetAnimationInt("StateID", 2);
```

## Swapping the Controller at Runtime

Two equivalent forms:

```csharp
streamingModel.RuntimeAnimatorController = newController;
streamingModel.SetRuntimeAnimatorController(newController);
```

If the model has not finished loading, the controller is stored and applied when load completes.

## Tuning `StreamingModelAnimator`

Auto-added component with two inspector fields:

| Property | Default | Range | Purpose |
| --- | --- | --- | --- |
| **Speed Multiplier** | 1.0 | 0.1–2.0 | Global playback speed multiplier. |
| **Transition Duration** | 0.5 | 0.01–1.0 | Forced cross-fade duration between states. |

```csharp
var animator = streamingModel.GetComponent<StreamingModelAnimator>();
if (animator != null)
{
    animator.SpeedMultiplier = 1.5f;
    animator.TransitionDuration = 0.25f;
}
```

## Verification Checklist

- [ ] `SetLoadedCallback` fires before any animation API call.
- [ ] Console shows clip names from **Log Animations** matching your dummy clip names exactly.
- [ ] `StreamingModelAnimator` component is present on the `StreamingModel` GameObject at runtime.
- [ ] `Auto Play Loop` toggled off automatically when the custom controller is assigned.
- [ ] Cross-fades look smooth; if not, tune **Transition Duration**.

## Gotchas

| Symptom | Cause | Fix |
| --- | --- | --- |
| Animator set but model plays no animation | No clips loaded from the model | Enable **Log Animations** to verify clips are present in the XRG. |
| Wrong animation plays for a state | Clip name mismatch (case, layer prefix) | Compare logged name to dummy clip name, or use index-string `"0"`/`"1"` for order-based mapping. |
| `PlayAnimation` has no effect | Called before load | Move the call inside `SetLoadedCallback`. |
| Parameter has no effect | Typo in parameter name | Parameter names are case-sensitive; verify in the Animator Controller. |
| Bones snap / feel stiff between states | Cross-fade too short | Increase `TransitionDuration` (max 1.0). |
| Blend shapes / morph targets not animating | Source clip has no weight channels | Verify the source GLB actually contains morph-target animation. |
| `Auto` mode picked `VrmMecanim` unexpectedly | Model has humanoid bones + a controller assigned | Force `EmbeddedClips` via `SetAnimationMode` if you want clip-based playback. |

## References

- In-package doc: `USAGE_ANIMATED_GLB.md`
- In-package Gitbook: `Gitbook/SubPage5.md`
- Runtime source: `Runtime/Scripts/StreamingModel.cs`, `Runtime/Scripts/StreamingModelAnimator.cs`
