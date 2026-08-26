# unity-polygon-streaming-skills

AI Skills bundle for the **Viverse Polygon Streaming Unity SDK** (`com.viverse.polygon-streaming`).

Polygon Streaming streams high-fidelity `.xrg` 3D models (converted from GLB / FBX / VRM) into a Unity scene at runtime, with progressive LOD, KTX-compressed textures, and dynamic occlusion culling. This bundle equips AI coding agents to integrate the SDK correctly — matching the exact component names, inspector fields, animation modes, and VRM APIs shipped in the package.

> [!NOTE]
> This is the **Unity** counterpart to [`web-polygon-streaming-skills`](https://github.com/viverseofficial/web-polygon-streaming-skills) (PlayCanvas / Three.js / Babylon.js).

## SDK identity

- **Package**: `com.viverse.polygon-streaming`
- **Unity**: 2021+
- **Render pipelines**: Built-in (BRP), URP, HDRP
- **Core components**: `StreamController` (scene-level), `StreamingModel` (per-model), `StreamingModelAnimator` (Legacy↔Mecanim bridge)
- **Optional dep**: UniVRM (`com.vrmc.gltf` + `com.vrmc.vrm`) — required for VRMA animation and MToon shading

## Skills in this bundle

| Skill | Covers | When to use |
| --- | --- | --- |
| [`unity-polygon-streaming-quickstart`](skills/unity-polygon-streaming-quickstart/SKILL.md) | Install (.tgz), scene setup, `PolygonStreaming` layer, prefabs | First-time integration or a fresh scene |
| [`unity-polygon-streaming-static-models`](skills/unity-polygon-streaming-static-models/SKILL.md) | Static GLB / FBX streaming, custom materials, runtime URL swap, `FetchColliderMesh` | Non-animated models (props, environments, engineering models) |
| [`unity-polygon-streaming-animated-glb`](skills/unity-polygon-streaming-animated-glb/SKILL.md) | Embedded animation clips, `StreamingModelAnimator`, custom `RuntimeAnimatorController`, dummy-clip matching | Animated GLBs / FBX with a state machine (non-VRM) |
| [`unity-polygon-streaming-vrm`](skills/unity-polygon-streaming-vrm/SKILL.md) | VRM expressions, look-at, spring bones, VRMA, `VrmMecanim`, MToon | VRM 1.0 avatars |

## Choosing the right skill

```
Are you streaming a VRM avatar?
├── YES → unity-polygon-streaming-vrm
│         (+ unity-polygon-streaming-quickstart if scene is not set up yet)
│
└── NO
    ├── Model has animation clips (Mixamo, embedded, humanoid)
    │   → unity-polygon-streaming-animated-glb
    ├── Static geometry only
    │   → unity-polygon-streaming-static-models
    └── First time integrating the SDK / need scene wiring
        → unity-polygon-streaming-quickstart
```

## Related bundles

- [`web-polygon-streaming-skills`](../web-polygon-streaming-skills/) — same `.xrg` format, web engines.
- [`viverse-unity-sdk-skills`](../viverse-unity-sdk-skills/) — Viverse Unity SDK (auth, avatars, leaderboards, multiplayer).

## Repo layout

```
unity-polygon-streaming-skills/
├── skills/                     # SKILL.md + skill.json per skill
├── catalog/                    # routes.json, skills.json, bundles.json, versions.json
├── schemas/                    # JSON Schemas
├── scripts/                    # build-index.mjs, validate-skills.mjs
├── docs/                       # authoring / consumption / versioning / deprecation
└── tests/
```

## Regenerate / validate

```bash
npm run build:index
npm run validate
```

## Official references

- Polygon Streaming Unity SDK product page: <https://www.viverse.com/polygon-streaming>
- SDK docs (Gitbook): shipped in-package under `Gitbook/`
- In-package usage docs: `USAGE_STATIC_MODELS.md`, `USAGE_ANIMATED_GLB.md`, `USAGE_VRM_STREAMING.md`
- Model console: <https://stream.viverse.com/console>
- UniVRM (optional): <https://github.com/vrm-c/UniVRM>
