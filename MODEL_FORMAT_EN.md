# Models — common glTF profile

Status: draft specification 0.4.0. Base format: glTF 2.0.

## 1. Scope and common rules

This profile describes the rules shared by static models and [dynamic models](DRAFT_VEHICLE_FORMAT_EN.md).

Custom fields defined here are placed directly in the `extras` of the relevant object. Keys describe specific data or behaviors. Standard glTF data is the source of values for geometry, materials, lights and animations.

This document defines the target contract. Implementation conformance is checked against the version given in `model_format_version`. Support for an extension in the format, the exporter and the rendering engine is verified separately during integration.

## 2. Container, coordinate system and resources

A model is stored as a valid glTF 2.0 file: `.gltf` with its resources, or `.glb`. External resource URIs are relative to the glTF file. The package contains all dependencies. A GLB file may also have external dependencies, which the packer resolves in the same way.

The profile uses meters, seconds and radians. A model has a single default scene with a single root representing the whole object. Model coordinate system: +Y up, +Z object front, +X to the left from the perspective of a viewer looking towards +Z. The exporter converts the authoring tool's coordinate system to this one.

The root transform in the file is identity. The instance placement transform holds the world transform. Directional lights and cameras keep their own glTF axis conventions. Moving nodes use TRS with a correctly placed pivot. Mechanism scale is baked before export.

Geometry, hierarchy, skeletons, morph targets and clips use the corresponding [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html) objects. `name` fields are for presentation in tools.

## 3. Standard representations

| Author data | Representation |
| --- | --- |
| Mesh and PBR materials | `meshes`, `materials`, `textures` |
| Transform and hierarchy | `nodes` |
| Translation / rotation / scale | `animations` with TRS channels |
| Skeleton and shape keys | `skins`, skeleton nodes, morph targets and animated `weights` |
| Optional viewpoint | glTF camera attached to a node |
| Spot, point or directional light | `KHR_lights_punctual`: `spot` / `point` / `directional` |
| Strong surface emission | `emissiveFactor`, `emissiveTexture`, `KHR_materials_emissive_strength` |
| Time-varying parameters | `KHR_animation_pointer` |
| Branch visibility | `KHR_node_visibility` |
| Levels of detail (LOD) | `MSFT_lod` on a node or material |

Every extension used in the model is listed in `extensionsUsed`. Extensions required for correct interpretation of the model are also listed in `extensionsRequired`; the importer checks them before creating an instance. When geometry variants rely on hiding parts, `KHR_node_visibility` is required. Control through pointers requires `KHR_animation_pointer`.

Levels of detail are stored with [MSFT_lod](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/MSFT_lod). The node or material that contains the extension is the highest LOD level. The `ids` array points to subsequent nodes or materials in order of decreasing detail.

The following extensions may be chosen for materials and asset distribution:

| Need | Extension |
| --- | --- |
| Glass, light transmission | `KHR_materials_transmission`, `KHR_materials_volume`, `KHR_materials_ior` |
| Clear coat, reflections | `KHR_materials_clearcoat`, `KHR_materials_specular` |
| Directional highlights, fabrics | `KHR_materials_anisotropy`, `KHR_materials_sheen` |
| Iridescence, dispersion | `KHR_materials_iridescence`, `KHR_materials_dispersion` |
| Lighting-independent material | `KHR_materials_unlit` |
| UV transform | `KHR_texture_transform` |
| KTX2/Basis Universal textures | `KHR_texture_basisu` |
| Geometry compression / quantization | `KHR_draco_mesh_compression`, `KHR_mesh_quantization` |
| Authorship and rights metadata | `KHR_xmp_json_ld` |

Extension selection is based on the [Khronos registry](https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md). The list above is the feature selection for this profile. Each use follows the semantics and constraints of the corresponding extension specification.

## 4. Model structure

### 4.1. Header

Example `asset` header:

```json
{
  "version": "2.0",
  "extras": {
    "model_format_version": "0.4.0"
  }
}
```


| Key in `asset.extras` | Described value | Example | Requirement |
| --- | --- | --- | --- |
| `model_format_version` | Model profile version. | `"0.4.0"` | Required |
| `osm` | Tags describing the model. | `{"natural":"tree"}` | Optional |
| `geo` | Geographic position and orientation of the model. | `{"lat":54.3520,"lng":18.6466,"elevation":12.5,"heading":90.0}` | Optional |
| `t3d` | T3D/E3D compatibility data. | `{"skins":["body","cab"]}` | Optional |


`extras.semantic_id` on nodes, materials and animations identifies elements used by the simulation. Values are unique within the given model collection and stable across re-exports. The reference type follows from the field: `node` points to a node, `material` to a material, and `animation` to a clip. The value is always the `semantic_id` of the target object.

An identifier consists of dot-separated segments matching `[a-z0-9_]+`, e.g. `building.clock.minute`, `infrastructure.signal.0.red`, `vehicle.door.left.0`. Reserved meanings of specific paths are defined by profiles. Other paths are stable author addresses. Ordinal indices start at zero.

Standard glTF references and JSON Pointers use indices as defined by the standard. The exporter updates them when rebuilding arrays. Semantic identifiers are used to bind the model to configuration and world state.

### 4.2. Tags

A model is identified by the path of its file in the data package. Model semantics are described by tags in `asset.extras.osm`. It is an object whose keys and values correspond to OSM tags, e.g. `{"natural":"tree"}` or `{"building":"train_station"}`. Several tags may be given at once.

| Key in `asset.extras` | Described value | Example | Requirement |
|---|---|---|---|
| `osm` | Map of tags describing the model's purpose. | `{"natural":"tree"}` | Optional |

#### 4.2.1. Well-known tag sets

The table lists tag sets recognized by tools. It was derived from the categories and models in MaSzyna's `nodebank.txt` and the corresponding OSM tags.

| `asset.extras.osm` | Meaning | Group in `nodebank.txt` |
|---|---|---|
| `{"natural":"tree"}` | Tree | Małe drzewka, Duże drzewka |
| `{"natural":"shrub"}` | Shrub | Krzaki |
| `{"landuse":"grass"}` | Grass | Trawy, Trawy atlas |
| `{"building":"house"}` | Residential house | Domki |
| `{"building":"barn"}` | Barn | Stodoły |
| `{"building":"apartments"}` | Apartment block | Bloki, wieżowce |
| `{"building":"industrial"}` | Industrial building | Przemysł |
| `{"building":"train_station"}` | Station building | Dworce |
| `{"railway":"signal_box"}` | Signal box | Nastawnie |
| `{"railway":"signal"}` | Signal or indicator | Semafory i wskaźniki |
| `{"railway":"platform"}` | Platform | Perony |
| `{"railway":"buffer_stop"}` | Buffer stop | Kozły oporowe |
| `{"railway":"level_crossing"}` | Level crossing | Przejazdy |
| `{"railway":"turntable"}` | Turntable | Mosty, wiadukty, tunele, obrotnice |
| `{"barrier":"gate"}` | Gate or wicket | Bramy |
| `{"barrier":"fence"}` | Fence | Ogrodzenia |
| `{"power":"pole"}` | Power pole | Słupy nN, SN |
| `{"power":"tower"}` | High-voltage pylon | Słupy WN i NN |
| `{"highway":"street_lamp"}` | Lamp or street light | Lampy i latarnie |
| `{"man_made":"street_cabinet"}` | Technical cabinet | Obiekty elektryczne, obiekty przytorowe |
| `{"man_made":"bridge"}` | Bridge or viaduct | Mosty, wiadukty, tunele, obrotnice |

### 4.3. Geographic position

The optional `asset.extras.geo` describes the position of the model's origin and its orientation:

| Key in `geo` | Described value | Example | Requirement |
|---|---|---|---|
| `lat` | WGS84 latitude in degrees, from −90 to 90. | `54.3520` | Required when `geo` is present |
| `lng` | WGS84 longitude in degrees, from −180 to 180. | `18.6466` | Required when `geo` is present |
| `elevation` | Height in meters above mean sea level. | `12.5` | Optional |
| `heading` | Direction of the local `+Z` axis in degrees, clockwise from geographic north. | `90.0` | Optional; defaults to `0.0` |

For `heading: 0` local `+Z` points north, and for `heading: 90` east. The local `+Y` axis points up.

### 4.4. Materials

Materials are stored as standard [`material`](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#materials) objects in the `materials` array. The `primitive.material` field contains the index of the material used by the given mesh primitive. If the field is absent, the default glTF material is used. The profile does not add its own material format.

| Field in `material` | Described value | Example | Default value | Requirement |
| --- | --- | --- | --- | --- |
| `name` | Material name. | `"paint_red"` | — | Optional |
| `pbrMetallicRoughness.baseColorFactor` | RGBA base color multiplier. | `[0.8, 0.1, 0.1, 1.0]` | `[1.0, 1.0, 1.0, 1.0]` | Optional |
| `pbrMetallicRoughness.baseColorTexture.index` | Base color texture index. | `0` | — | Optional |
| `pbrMetallicRoughness.metallicFactor` | Metalness from 0 to 1. | `0.0` | `1.0` | Optional |
| `pbrMetallicRoughness.roughnessFactor` | Roughness from 0 to 1. | `0.7` | `1.0` | Optional |
| `pbrMetallicRoughness.metallicRoughnessTexture.index` | Index of the combined roughness and metalness texture. | `1` | — | Optional |
| `normalTexture.index` | Normal map texture index. | `2` | — | Optional |
| `normalTexture.scale` | Normal deviation multiplier. | `0.8` | `1.0` | Optional |
| `occlusionTexture.index` | Ambient occlusion texture index. | `1` | — | Optional |
| `occlusionTexture.strength` | Occlusion strength from 0 to 1. | `1.0` | `1.0` | Optional |
| `emissiveTexture.index` | Emissive texture index. | `3` | — | Optional |
| `emissiveFactor` | RGB emissive color multiplier. | `[1.0, 0.8, 0.6]` | `[0.0, 0.0, 0.0]` | Optional |
| `alphaMode` | Alpha channel handling: `OPAQUE`, `MASK` or `BLEND`. | `"MASK"` | `"OPAQUE"` | Optional |
| `alphaCutoff` | Fragment discard threshold with `alphaMode: "MASK"`. | `0.5` | `0.5` | Optional |
| `doubleSided` | Render both sides of the surface. | `true` | `false` | Optional |

Every texture field contains an `index` pointing to an element of the `textures` array. A `texture` element points to an image via `source` and to an optional sampling mode via `sampler`. The `texCoord` field selects the texture coordinate set; its default value is `0`.

| Texture | Channel usage | Color space |
| --- | --- | --- |
| `baseColorTexture` | RGB: base color; A: transparency | sRGB |
| `metallicRoughnessTexture` | G: roughness; B: metalness | linear |
| `normalTexture` | RGB: tangent-space normal | linear |
| `occlusionTexture` | R: ambient occlusion | linear |
| `emissiveTexture` | RGB: emissive color | sRGB |

| Use case | Encoding |
|---|---|
| Solid surface | `alphaMode: OPAQUE` |
| Alpha cutout: mesh, leaf, fence | `alphaMode: MASK`, threshold `alphaCutoff` |
| Transparency blending | `alphaMode: BLEND` |
| Optical glass | PBR with transmission / volume / IOR extensions |
| Double-sided surface | `doubleSided` |

The exporter chooses the mode matching the intended effect. Coverage transparency and optical transmission are independent material properties. A dirty window pane may combine appropriately prepared layers.

Constant surface emission is stored in the material, optionally with [KHR_materials_emissive_strength](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_emissive_strength). Illumination of the surroundings is assigned to a light attached to the corresponding node.

`emissiveTexture` is always rendered according to the material and is not used to switch lights on or off.

### 4.5. Replaceable textures

The model contains default glTF materials and textures. A set of replaceable textures is a map from their names to resources, e.g. `{"body":"skins/red/body.png"}`. It replaces the specified parts of the model; the others use the default textures. The optional `asset.extras.t3d.skins` maps legacy format numbers to these names, without mixing them with glTF material numbers.

| Key in `asset.extras.t3d` | Described value | Example | Requirement |
|---|---|---|---|
| `skins` | Names of replaceable texture slots in T3D/E3D index order; index 0 corresponds to `-1`. | `["body", "cab"]` | Optional |

| Key in `node.extras` | Described value | Example | Requirement |
|---|---|---|---|
| `replacable_texture` | Name of the replaceable texture of the model part represented by the node. | `"body"` | Optional; present for parts with a replaceable texture |

### 4.6. Light layers

A model may have several independent light groups, e.g. signal light chambers or vehicle lamps. Each group has a node (submodel) with a mesh of the light surfaces. Two atlases are used: the base appearance and a separate light layer. Assignment of a surface to a group follows from the node. How the light atlas is referenced is still to be decided.

| Key in `node.extras` | Described value | Example | Default value | Requirement |
|---|---|---|---|---|
| `light_id` | Identifier of an independently controlled light group. | `0` | — | Required on a node with a light mesh |
| `light_color` | Light color as RGB. | `[1.0, 0.0, 0.0]` | `[1.0, 1.0, 1.0]` | Optional on a node with `light_id` |
| `light_state` | Light state represented by the node's mesh: `"on"` or `"off"`. | `"on"` | `"on"` | Optional on a node with `light_id` |
| `light_threshold` | Ambient light level below which the light is on. | `0.325` | `0.325` | Optional on a node with `light_id` |

### 4.7. Light definitions (DRAFT)

Lights are stored with [KHR_lights_punctual](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_lights_punctual). The Blender exporter maps `POINT` to `point`, `SPOT` to `spot`, `SUN` to `directional`. The exporter is responsible for converting the author's units.

A definition contains `type`, `color`, `intensity`, optional `range`, and the angles `spot.innerConeAngle` and `spot.outerConeAngle`. `point` and `spot` use candelas, `directional` uses lux. The node transform sets position and orientation; a `spot` light shines along local −Z. An omitted `range` means infinite range, as defined by the extension.

Example of a complete glTF with a lamp:

```json
{
  "asset": {"version": "2.0"},
  "extensionsUsed": ["KHR_lights_punctual"],
  "extensions": {
    "KHR_lights_punctual": {
      "lights": [{
        "type": "spot",
        "color": [1.0, 0.9, 0.75],
        "intensity": 1200.0,
        "range": 40.0,
        "spot": {"innerConeAngle": 0.25, "outerConeAngle": 0.5}
      }]
    }
  },
  "nodes": [{
    "extensions": {"KHR_lights_punctual": {"light": 0}},
    "extras": {"semantic_id": "infrastructure.lamp.0"}
  }],
  "scenes": [{"nodes": [0]}],
  "scene": 0
}
```

This is an example of the light mechanism; the profile header is added as described in section 4.1. A plain lamp works once this data is loaded. Scenario control is described by the optional `light_control` on the node:

```json
{
  "light_control": {
    "source": "lighting.platform.0.level",
    "default": 1.0
  }
}
```

`source` points to a numeric instance channel in the range [0,1]. A default value is required. The engine multiplies the light's base `intensity` by the channel value. The dusk rule or schedule is supplied by the world. A shared light definition is instantiated with separate state for each controlled node.

Photometric profiles are stored with the optional [EXT_lights_ies](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_ies); image-based lighting resources are described by [EXT_lights_image_based](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_image_based). Their use requires declared support in the processing pipeline.

### 4.8. Smoke and other emitters (DRAFT)

The emitter position is represented by a node. Local +Y defines the initial direction. `node.extras.smoke_source` describes a specific particle effect:

```json
{
  "semantic_id": "building.chimney.0",
  "smoke_source": {
    "rate": 12.0,
    "max_particles": 400,
    "lifetime": [3.0, 6.0],
    "speed": [0.8, 1.4],
    "cone_angle": 0.2,
    "size": [0.25, 0.5],
    "growth": 0.15,
    "acceleration": [0.0, 0.1, 0.0],
    "drag": 0.1,
    "inherit_velocity": 0.0,
    "material": "effects.chimney_smoke",
    "source": "chimney.0.output",
    "default": 0.0
  }
}
```

Required: `rate` in particles/s, a positive integer `max_particles`, a positive `lifetime` range in seconds, non-negative ranges `speed` in m/s and `size` (diameter) in meters, and a `material` reference. The remaining fields default to 0, except `default` = 1. `source` is an optional [0,1] channel that multiplies `rate`.

`cone_angle` is a half-angle in [0,π]; directions are distributed uniformly over solid angle. Range parameters are sampled uniformly and independently at birth. `growth` is the diameter growth rate in m/s; the diameter is clamped from below at zero. `acceleration` is a vector in m/s² in the model coordinate system, `drag` ≥ 0 is a drag coefficient in 1/s; the velocity equation is `dv/dt = acceleration - drag*v`. `inherit_velocity` ∈ [0,1] multiplies the emitter's world-space velocity at birth.

Particles live in world space and are camera-facing quads. At birth, the velocity direction is transformed by the emitter transform, and the acceleration by the model rotation, into world space; these values are then used by the integrator. The glTF material defines texture, color, emission and alpha. Alpha is additionally multiplied by `1-age/lifetime`. Once the limit is reached, new particles are skipped, and after the emission channel is switched off, existing particles live out their lifetime. Hiding the node via `KHR_node_visibility` stops births and hides the whole effect; particle age keeps advancing.

### 4.9. Animation (DRAFT)

A clip stores animation prepared by the author. The results of constraints and inverse kinematics (IK) computed in the tool are baked into channels. Animating material and light parameters uses [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer).

A clip selected by the world contains `animation.extras.semantic_id`. The optional `animation.extras.playback` defines how it is controlled:

```json
{
  "semantic_id": "building.gate.opening",
  "playback": {
    "mode": "position",
    "source": "gate.0.open_fraction",
    "input": [0.0, 1.0],
    "time": [0.0, 2.4],
    "default": 0.0
  }
}
```

In `position` mode the input x is clamped to `[a,b] = input`, and the sample time is `t0 + (t1-t0)*(x-a)/(b-a)`. All fields shown are required; `a < b`, and `time` lies within the clip. This mode allows reversing and stopping the motion at any opening.

`loop` mode uses `source` as a boolean run flag, `default` as a boolean value, `time` as the clip range, and an optional `speed` > 0 (default 1). A `false` → `true` change starts playback from t0, `true` plays cyclically, and `false` stops at t0. Time comes from the simulation clock. The author encodes a constant loop by omitting `source` and setting `default: true`.

Visibility and its animation use [KHR_node_visibility](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_node_visibility). The default `visible` is stored in the node's extension. A logical channel can control it via `node.extras.visibility_control`:

```json
{
  "visibility_control": {
    "source": "season",
    "equals": "winter"
  }
}
```

With `equals`, the result is a strict comparison of the channel value with the literal. With `source` alone, a boolean channel is required. Before the first channel update, `visible` from the extension applies. This encoding supports seasonal variants and signal-dependent appearance.

A clock may use `position` clips sampled by the channels `clock.second`, `clock.minute`, `clock.hour12`, `clock.hour24`. The world supplies numbers with a fractional part; the author sets the clip range and interpolation. Programmed blinking may use a `loop` clip, while the signal aspect is an input from the scenario.

#### 4.9.1. Update contract

A `source` channel is the name of a value supplied by the world or simulation to a specific instance. Instance configuration resolves the required channels before activation. Channels carry booleans, numbers or text depending on the consumer; IK target channels in the dynamic profile carry a three-component vector (`vec3`). Numbers are finite.

Order: read a consistent snapshot of channels → sample clips → procedural drivers → update world-space transforms → lights, visibility, emission and particles. Each written property has a single active writer in a given instance. A conflict between two clips, or between a clip and a driver, is a configuration error. Shared materials have per-instance state or copy-on-write. Two instances keep independent playback states.

## 5. Export and validation

1. The author prepares geometry, materials, lights, pivots and clips in the tool.
2. glTF export includes lights, animations and the custom properties used by the profile. Baking includes the results of constraints and inverse kinematics (IK).
3. The packaging stage normalizes metadata to the locations defined in this specification, assigns stable identifiers and updates references. Exporter settings and the placement of custom properties are checked for the tool version in use.
4. The glTF validator checks the standard structure; the profile validator checks the model contract.
5. An import test checks the scene, scale, emission, alpha, lights and representative clip samples. Support for each extension used is verified on the exported file.

Profile validation checks:

- the header, version, identity root and resolvability of resources;
- ID uniqueness and the type and target of each reference;
- value ranges in the optional `geo`;
- channel types, numeric ranges and the single writer of each property;
- assignment of `light_control` to a node with a light;
- the value of `light_state` and the numeric value of `light_threshold`;
- existence of the emitter material and positive lifetime limits;
- extension declarations, their constraints and importer capabilities.

The profile version is independent of `asset.version`. `PATCH` clarifies the contract, `MINOR` adds optional capabilities, and `MAJOR` changes data interpretation. During the 0.x phase the importer explicitly negotiates every version.

## 6. Sources

- [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html).
- [Extension registry](https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md).
- [Blender light exporter](https://github.com/blender/blender-addons/blob/main/io_scene_gltf2/blender/exp/gltf2_blender_gather_lights.py) — an example implementation of the conversion; the processing pipeline pins a specific exporter version.
