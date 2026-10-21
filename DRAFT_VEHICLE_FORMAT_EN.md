# Dynamic models — glTF profile

Status: draft specification 0.4.0. Base format: glTF 2.0.

## 1. Scope and common part

This profile describes a vehicle or its equipment as an instance bound to the simulation. Geometry and standard effects are created in the authoring tool. Metadata describes how the simulation finds parts and drives their presentation.

The [common model profile](MODEL_FORMAT_EN.md) applies, including replaceable textures and independent light layers. This document adds the vehicle contract.

| Common rules | Specific to a vehicle instance |
|---|---|
| Geometry, PBR, alpha and replaceable textures | Binding to the simulated vehicle |
| `point`, `spot`, `directional` lights | Head/tail signals, headlights and dimming |
| Clips, skinning and morph targets | Sampling clips by door, controller and wiper state |
| Visibility and emission | Variants of couplers, hoses and equipment |
| Smoke, steam and other particles | Emission depending on engine operation |
| Data provenance | Source data of the model and measurements |
| Stable IDs and input channels | Axles, bogies, cabs, contact points and attachment points |

Instance placement is updated by the simulation. `asset.extras.osm` may describe the model with OSM tags if a matching tag set exists. The optional `asset.extras.geo` may document the location of the measured source object.

## 2. Header and vehicle axes

Example `asset` object:

```json
{
  "version": "2.0",
  "extras": {
    "model_format_version": "0.4.0"
  }
}
```

The vehicle front is at +Z, up at +Y, the left side at +X. Left/right side and front/rear refer to the model's construction and keep their meaning when the direction of travel changes. The recommended origin lies on the longitudinal axis, at top-of-rail level and at the middle of the constructional length.

Every independently moving element has a node whose pivot is at its point of rotation or translation. Rigid parts may be plain nodes; deformable hoses, bellows or characters may use skins.

## 3. Identifiers and model structure

`semantic_id` is both a stable address and the name of the element's function. The conventions below form a catalog recognized by the vehicle integration. The node name in Blender may be any description meaningful to the author.

| `semantic_id` pattern | Meaning |
|---|---|
| `vehicle.body` | Body / main structure |
| `vehicle.bogie.<n>` | Bogie |
| `vehicle.axle.<n>` | Wheelset |
| `vehicle.wheel.<n>.left` / `.right` | Separate visual wheels of a wheelset |
| `vehicle.door.left.<n>` / `.right.<n>` | Door assembly |
| `vehicle.doorstep.left.<n>` / `.right.<n>` | Step |
| `vehicle.headlight.front.left` / `.right` / `.upper` | Front headlight |
| `vehicle.headlight.rear.left` / `.right` / `.upper` | Rear headlight |
| `vehicle.high_beam.<end>.<side>` | High beam |
| `vehicle.marker.<end>.<side>` | End marker light |
| `vehicle.auxiliary_light.<end>.<side>` | Auxiliary light |
| `vehicle.coupler.front` / `.rear` | Coupler |
| `vehicle.hose.<end>.<system>.<side>` | Pneumatic hose |
| `vehicle.control_cable.<end>` | Control cable |
| `vehicle.gangway.<end>` | Gangway |
| `vehicle.buffer.<end>.<side>` | Buffer |
| `vehicle.pantograph.<n>` | Pantograph assembly |
| `vehicle.cab.<n>` | Cab assembly |
| `vehicle.compartment.<n>` | Compartment |
| `vehicle.corridor.<n>` | Corridor |
| `vehicle.load.<n>` | Load |
| `vehicle.exhaust.<n>` | Exhaust emitter |
| `vehicle.driver.<n>` | Driver character |
| `vehicle.mirror.<end>.<side>` | Mirror |
| `vehicle.wiper.<n>` | Wiper mechanism |
| `vehicle.cooling_shutter.<n>` | Cooling shutter |

`end` takes `front` or `rear`, `side` takes `left`, `right` or `center`; the hose system identifies e.g. `brake` or `supply`. `<n>` indices start at zero and are counted from front to rear. The patterns are document notation; the exporter writes concrete identifiers.

An assembly may have identifiers of sub-functions, e.g. `vehicle.headlight.front.left.source`, `.lens`, `.dimmed` or `vehicle.pantograph.0.lower_arm`. A reference points to the exact ID value. Attaching an `attachment` model creates a separate instance ID space; the attachment configuration specifies the owner instance and its target node.

Example hierarchy:

```text
vehicle.root
├── vehicle.body
│   ├── vehicle.cab.0
│   ├── vehicle.door.left.0
│   ├── vehicle.headlight.front.left
│   │   ├── vehicle.headlight.front.left.source
│   │   └── vehicle.headlight.front.left.lens
│   └── vehicle.pantograph.0
├── vehicle.bogie.0
│   ├── vehicle.axle.0
│   └── vehicle.axle.1
└── vehicle.bogie.1
    ├── vehicle.axle.2
    └── vehicle.axle.3
```

The actual parenting follows from transform needs. Mechanical dependencies are additionally expressed by explicit references in drivers.

## 4. Simulation channel contract

Channels are input values of a specific instance. The names in this document form an integration contract that the physics and scenario adapter MUST supply for the features used.

| Channel | Type / unit | Meaning |
|---|---|---|
| `door.left.<n>.open_fraction` | number [0,1] | Actual door opening |
| `doorstep.left.<n>.extension` | number [0,1] | Actual step extension |
| `axle.<n>.angle` | number, rad | Wheelset rotation angle |
| `bogie.<n>.yaw` | number, rad | Bogie yaw relative to its parent |
| `body.heave` | number, m | Vertical body displacement |
| `headlight.<end>.<side>.level` | number [0,1] | Light level |
| `headlight.<end>.<side>.mode` | string | `off`, `dimmed`, `full` |
| `coupler.<end>.visual` | string | `disconnected`, `connected_owner`, `connected_peer`, `adapter` |
| `hose.<end>.<system>.<side>.visual` | string | `disconnected`, `straight`, `slanted` |
| `buffer.<end>.<side>.compression` | number, m | Buffer compression |
| `pantograph.<n>.contact_height` | number, m | Target height of the contact point in the asset coordinate system |
| `pantograph.<n>.raised` | bool | Raised state |
| `engine.exhaust.<n>.output` | number [0,1] | Exhaust emission intensity |
| `wiper.<n>.phase` | number [0,1] | Wiper cycle phase |
| `cab.<n>.occupied` | bool | Cab occupancy |
| `display.destination.text` | string | Destination text |

The corresponding right-side door and step channels use `right`. Channels for new devices MAY be added in the instance configuration with a documented type and unit. All channels used MUST be resolved. A channel describing the actual opening comes from the simulated mechanism and its constraints.

Driver default values are used for preview and initialization. Vehicle state and visual state are separate for each instance, even with shared geometry and materials.

## 5. Clips: doors, steps, wipers, indicator lights

A glTF animation stores motion prepared by the author. `animation.extras.playback` from the common profile binds it to a channel. Example extras of a door clip:

```json
{
  "semantic_id": "vehicle.door.left.0.opening",
  "playback": {
    "mode": "position",
    "source": "door.left.0.open_fraction",
    "input": [0.0, 1.0],
    "time": [0.0, 3.0],
    "default": 0.0
  }
}
```

The sample at 0 shows the door closed, and at 1 open. A single clip may move several leaves and linkages. Door speed, interlocks and stopping are determined by the simulation through the channel's course over time.

Steps, mirrors, shutters and controllers use the same rule. A wiper may have a full-cycle clip driven by `wiper.<n>.phase`; the start and end have the same pose. Cams and gears prepared in Blender are baked into the clip.

Clips animating a light, material or visibility use [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer). A shared clip may change lens emission and light intensity at the same time. In the configuration each property has a single active writer: a clip or the corresponding driver.

## 6. Procedural motion: axles, bogies and suspension

Motion computed directly by the simulation is described by `node.extras.rotation_control` or `translation_control`. Example wheelset:

```json
{
  "semantic_id": "vehicle.axle.0",
  "rotation_control": {
    "source": "axle.0.angle",
    "axis": [1.0, 0.0, 0.0],
    "scale": 1.0,
    "offset": 0.0,
    "default": 0.0
  }
}
```

`source` and a unit local `axis` are required. `scale` defaults to 1, `offset` and `default` to 0. Angle `θ = source*scale + offset`. The resulting local rotation is `q_rest * angleAxis(θ, axis)`, with the angle in radians and the right-hand rule. The optional `limits: [min,max]` clamps θ after conversion. The wheel angle may grow continuously; the renderer reduces it modulo 2π when building the quaternion.

The simulation computes the wheel angle from rotational speed, or from distance and rolling radius, accounting for slip. Units and sign of the channel are part of the axle configuration.

For a bogie, `rotation_control` is used with axis [0,1,0] and the `bogie.<n>.yaw` channel. Wheelsets are children of the bogie, so they inherit its motion.

`translation_control` has the same fields. The result is `t_rest + axis*(source*scale+offset)` in meters. It is used e.g. for vertical body motion, suspension travel and buffer compression. To compose independent rotations around several axes, the author creates successive pivot nodes with separate channels.

The scale of driven nodes and their ancestors MUST be identity after baking on export. Pivots, offsets and geometry define the physical position of the mechanism in the model.

## 7. Vehicle lights and dimming

A headlight is a node with a standard `KHR_lights_punctual` light; its lens or reflector is geometry with a material. Both may be children of an assembly with a common ID. The author can prepare this set directly in Blender.

Example extras of the source node:

```json
{
  "semantic_id": "vehicle.headlight.front.left.source",
  "light_control": {
    "source": "headlight.front.left.level",
    "default": 0.0
  }
}
```

`light_control` multiplies the intensity written by the exporter in the light definition. Dimming to 0.2 yields 20% of that value. A controlled lens surface uses `light_id`; its state may be bound to the same channel. The material's `emissiveTexture` remains constant. Photometric distribution, color and cone belong to the standard light description.

With separate geometry for the dimmed lamp, its node has the standard `KHR_node_visibility` and:

```json
{
  "semantic_id": "vehicle.headlight.front.left.dimmed",
  "visibility_control": {
    "source": "headlight.front.left.mode",
    "equals": "dimmed"
  }
}
```

Likewise, the full variant compares against `full`, and the off variant against `off`. The same rule covers cab indicator lights. The default state is stored as `visible` in the node's extension.

Separate halo geometry may be a billboard with an emissive/unlit material as appropriate for the look. `node.extras.billboard` takes `"camera"` or `"vertical"`: respectively, it orients local +Z towards the camera, with the camera's up, or rotates only around the world +Y axis. Position comes from the hierarchy. The billboard is the writer of this node's rotation. The material determines alpha; visibility may use the same lamp channel.

Renderer properties, such as shadow map resolution and bias, are chosen by the renderer to suit the quality and scale of the scene.

## 8. Couplers, hoses and attachment points

The coupler assembly node has `semantic_id: vehicle.coupler.front` or `.rear`. Variant geometries use child IDs and `visibility_control` with the `coupler.<end>.visual` channel. Example variant:

```json
{
  "semantic_id": "vehicle.coupler.front.connected_peer",
  "visibility_control": {
    "source": "coupler.front.visual",
    "equals": "connected_peer"
  }
}
```

The adapter is a separate channel value, `adapter`; the author may assign it the same mesh as another variant while keeping separate nodes, or prepare a visibility clip.

Hoses have `hose.<end>.<system>.<side>.visual` channels: `straight`, `slanted`, `disconnected`. The position of pivots and end fittings follows from the geometry. Geometry variants may be replaced by skin deformation driven by a mechanism when the model contains such a rig.

Attachment points are transform nodes with an ID, e.g. `vehicle.coupler.front.anchor`, `vehicle.hose.front.brake.left.anchor`, `vehicle.pantograph.0.contact`. The anchor's local +Z points outwards, +Y is its up. An instance attached to an anchor multiplies the anchor's world transform by its own local transform.

## 9. Pantograph: geometry, motion and contact

A pantograph has an assembly node, moving parts and a contact point node. The author chooses one of two mechanism contracts: baked kinematics driven by height, or runtime IK. The choice is recorded by the presence of the corresponding control description.

### 9.1. Author-prepared kinematics

This is the recommended representation for pantographs with complex linkages, a parallelogram or several arms. The author solves the constraints in the tool, exports a clip of all members, and then binds the contact point height to the clip time.

`node.extras.pantograph` on the assembly node:

```json
{
  "semantic_id": "vehicle.pantograph.0",
  "pantograph": {
    "animation": "vehicle.pantograph.0.motion",
    "contact_node": "vehicle.pantograph.0.contact",
    "height_source": "pantograph.0.contact_height",
    "raised_source": "pantograph.0.raised",
    "rest_time": 0.0,
    "height_to_time": [
      [4.2, 0.2],
      [5.0, 0.8],
      [6.0, 1.7],
      [6.5, 2.0]
    ],
    "tolerance": 0.005
  }
}
```

All fields are required. The rows of `height_to_time` are `[contact height Y in meters in the asset coordinate system, clip time in seconds]`. Both columns MUST be strictly increasing. At least two rows. The runtime interpolates time linearly between adjacent rows and samples the clip. Outside the range it uses the nearest boundary. With `raised_source: false` it samples `rest_time`.

The height channel supplies the target point already converted to the asset coordinate system, taking into account the vehicle's pose and tilt. The exporter generates a table dense enough that the height of `contact_node` after interpolation differs from the target by at most `tolerance` over the whole range. The validator checks this property by sampling and adaptive refinement.

A clip driven by `pantograph` is owned by this mechanism. Contact state with the overhead line and current collection are computed by the simulation. When the mechanical limit is reached, it reports the reachable contact position to the simulation; the renderer shows the limit position. The author may add a separate pan head rotation driven by the contact tilt through a dedicated node.

### 9.2. Runtime IK — two-bone chain

The profile defines `node.extras.ik` for two rigid members. It is used for mechanisms with a suitable structure and for auxiliary rigs. Bones/nodes, skins and their rest poses are stored in standard glTF.

```json
{
  "ik": {
    "solver": "two_bone",
    "root": "vehicle.pantograph.0.lower_arm",
    "middle": "vehicle.pantograph.0.upper_arm",
    "tip": "vehicle.pantograph.0.contact",
    "target_source": "pantograph.0.target",
    "pole_source": "pantograph.0.pole",
    "enabled_source": "pantograph.0.raised",
    "tolerance": 0.001
  }
}
```

All fields are required. `middle` is a direct child of `root`, and `tip` a child of `middle`. Member lengths are positive distances between node origins in the rest pose. Their scales and the scales of their ancestors are identity.

`target_source` and `pole_source` supply vec3 points in the asset coordinate system. The integration defines these channels; `pole` indicates the bending side. `enabled_source` is a bool. When false, the rest pose applies. The solver writes the rotations of `root` and `middle`, preserving the lengths and the position of the chain's start.

The target distance is clamped to the reachable interval `[abs(L1-L2)+ε, L1+L2-ε]`, where `ε = min(tolerance, 0.001*min(L1,L2))`. For a target coinciding with the chain's start, the direction comes from the root→tip vector of the rest pose. The solution uses the law of cosines in the plane defined by the target direction and the pole. With a collinear pole, the rest pose bending plane is used; the author MUST ensure it is well defined. The selected solution branch keeps the pole side. Member twist follows from the minimal rotation of their rest axes to the solution axes.

The local rotation of `tip` remains at rest; a separate pan head node may receive orientation control. The solver reports the target-reaching error as the tip–target distance, so that the simulation can resolve contact. `tolerance` is positive and defines the allowed error for a reachable target.

The exporter MUST check that the mechanism geometry is compatible with the selected solver. Multi-member mechanisms with coupled constraints use contract 9.1, which stores the author's full kinematics.

## 10. Cabs, cameras, displays and loads

The cab and compartments are assembly nodes with corresponding IDs. Elements restricted to selected views may have `node.extras.view_filter`, an array of values `external`, `cab`, `passenger`, `reflection`, `shadow`, `editor`. Omission means all views. The filter applies to the subtree; the result is the intersection of the ancestors' filters. The view state is supplied by the renderer for a specific pass.

The optional viewpoint of a driver's position is a glTF camera attached to a node, e.g. `vehicle.cab.0.camera`. The node transform defines the position and orientation of the viewpoint. The simulator may use it to set the active camera.

A display has a node with geometry and a material. `node.extras.display` binds the content to a channel:

```json
{
  "semantic_id": "vehicle.display.destination",
  "display": {
    "source": "display.destination.text",
    "material": "vehicle.display.destination.surface",
    "resolution": [1024, 128],
    "font": "fonts/destination.ttf",
    "font_size": 96,
    "color": [1.0, 0.7, 0.0, 1.0],
    "background": [0.0, 0.0, 0.0, 1.0]
  }
}
```

All fields are required. Unicode text is rasterized into an sRGB image with a positive integer resolution; `font_size` is in pixels. RGBA colors have components in [0,1], RGB in sRGB. Text is centered, single-line, and clipped if too wide. The font is a package resource. The result replaces the `baseColorTexture` image of the specified material in this instance; the material MUST have this texture, and the geometry proper UVs. The author sets the emission and other material properties. The initial texture image serves as the export preview.

Mechanical digits and gauge needles use `position` clips. Visibility of a load or a character may be bound to a channel via `visibility_control`; deformation and motion use clips, morphs or a skin.

## 11. Exhaust and steam

The `vehicle.exhaust.<n>` emitter uses `smoke_source` from the common profile. The `engine.exhaust.<n>.output` channel modulates emission, and `inherit_velocity` defines the share of the emitter's velocity passed to particles at birth.

Example emitter configuration in `node.extras`:

```json
{
  "semantic_id": "vehicle.exhaust.0",
  "smoke_source": {
    "rate": 30.0,
    "max_particles": 600,
    "lifetime": [2.0, 5.0],
    "speed": [1.0, 2.0],
    "size": [0.2, 0.4],
    "cone_angle": 0.15,
    "growth": 0.3,
    "acceleration": [0.0, 0.2, 0.0],
    "drag": 0.15,
    "inherit_velocity": 1.0,
    "material": "effects.diesel_smoke",
    "source": "engine.exhaust.0.output",
    "default": 0.0
  }
}
```

The emission position and direction inherit the body's motion. The emitter velocity includes the vehicle's translational motion and the point's motion resulting from rotation. Color and alpha come from the material; changes depending on load may be prepared by the author in a material parameter clip driven by a separate channel.

## 12. Mapping authoring data to the format

| Element prepared in Blender | Target data | Additional simulation contract |
|---|---|---|
| Point / Spot / Sun | `KHR_lights_punctual` | Optional `light_control` |
| Controlled lens surface | Mesh with a base atlas and a separate light atlas | `light_id`, optional `light_color` |
| Door motion baked from constraints | TRS clip | `playback` of the opening channel |
| Armature and deformed hose | Skin and joints | Clip or mechanism driving the joints |
| Load shape keys | Morph targets | Clip sampled by the load channel |
| Pantograph with authored IK | Baked clip of all members | `pantograph.height_to_time` |
| Two-bone rig driven by a world target | Nodes / joints | `ik` with target and pole channels |
| Wheel with a correct pivot | Node | `rotation_control` |
| Optional cab viewpoint | glTF camera and node | Driver position `semantic_id` |
| Vehicle livery | `node.extras.replacable_texture` | Texture selection in the skin |
| Exhaust outlet Empty | Transform node | `smoke_source` |

Export MUST preserve nodes used as pivots, contact points and reference targets. The optimizer updates indices and JSON Pointers while preserving semantic IDs. The importer builds ID maps once while loading the asset.

## 13. T3D/E3D and MMD mapping

The mapping applies to the function recognized from the base name and the vehicle configuration. The converter determines the meaning of the suffix in this context.

### 13.1. Lights and couplers

| Legacy | Meaning in the checked code | Result |
|---|---|---|
| `headlamp*_off` | Headlight off | Geometry controlled by `mode == off` |
| `headlamp*_on` | Full light | Geometry controlled by `mode == full` |
| `headlamp*_xon` | Dimmed headlight | Geometry controlled by `mode == dimmed` |
| `headsignal*_xon` | Dimmed auxiliary light | Corresponding auxiliary lamp channel |
| `highbeam*_xon` | Variant selected when high beams are on | High beam activity channel |
| `coupler*_off` | Coupler disconnected | `visual == disconnected` |
| `coupler*_on` | Connected, owner of the representation | `visual == connected_owner` |
| `coupler*_xon` | Other side of the connection or adapter | Variants `connected_peer` and `adapter` |
| `pneumatic*`, `cpneumatic*` with `_off` | Hose stowed | `visual == disconnected` |
| These hoses with `_on` | Straight connection | `visual == straight` |
| These hoses with `_xon` | Slanted connection | `visual == slanted` |

Front/rear and side bindings are read from the legacy configuration. In the code `headlamp11` is front upper, `12` right, `13` left; `21…23` correspond to the rear. The converter maps the `cpneumatic` and `pneumatic` prefixes according to the hose's function in the simulation, to the brake and supply connection channels respectively.

The code selects `_xon` for headlights with `DimHeadlights`, and for `highbeam` when high beams are switched on. The shared class handling these variants is a legacy implementation detail; the mapping result follows from the device's function.

### 13.2. Parts and animations

| Legacy / MMD | Result |
|---|---|
| `bogie1/2`, `boogie01/02` | `vehicle.bogie.0/1`, rotation from the bogie channel |
| `animwheelprefix:` | Corresponding wheelset or wheel, `rotation_control` |
| `animdoorprefix:` | Door, `playback` clip of the opening fraction |
| `animstepprefix:` | Step, extension clip |
| `animmirrorprefix:` | Mirror, position clip |
| `animwiperprefix:` | Wiper, phase clip |
| `animpantrd1prefix:`, `animpantrd2prefix:` | Lower pantograph members |
| `animpantrg1prefix:`, `animpantrg2prefix:` | Upper pantograph members |
| `animpantslprefix:` | Pan head and contact node |
| `animpendulumprefix:` | Valve gear member, baked kinematics |
| `buffer_left01/02`, `buffer_right01/02` | Buffers, translation by the compression channel |
| `cab0/1/2` | Cabs with explicit mapping of author indices |
| `corridorN`, `korytarzN`, `compartmentN`, `przedzialN` | Corridor and compartment assemblies |
| `external_loadN` | Load node with visualization channels |
| `mechanik1/2` | Character controlled by cab occupancy |
| `shutters1/2` | Shutters controlled by a clip or visibility |
| `external_only` | `view_filter` containing the required external views |
| Materials `-1…-4` / `replacableskin` | Default glTF materials and textures; `node.extras.replacable_texture` and index mapping via `asset.extras.t3d.skins` |
| Destination board | `display` on the recognized surface |
| `cien` | Auxiliary geometry classified in the migration report; runtime model shadows follow from its geometry |

The converter MUST generate a table of the legacy names used, target IDs, channels and approximations. The pantograph rig requires reconstructing the kinematics from model parameters; member names alone serve identification. Materials and lights are calibrated to the target PBR and photometric parameters.

## 14. Update order and data ownership

1. The simulation computes the instance transform and the channel snapshot.
2. The runtime restores the base values of controlled properties, then samples clips.
3. Rotation/translation drivers and mechanisms update their assigned nodes in parent–child order.
4. The runtime recomputes world transforms and skin matrices.
5. Control of lights, materials, visibility and emitters uses the same snapshot.
6. The renderer applies view filters and computes billboards for the specific camera.

A mechanism may read an ancestor's result, but the dependency order MUST form an acyclic graph. Each written TRS, material parameter, light parameter or visibility has a single writer in a given pass. A collision of control scopes is a validation error.

A shared asset stores the base data. Materials, lights, clips, mechanisms and particles have state assigned to the instance. Despawn ends emission and releases its state according to the world's policy; recreating the instance restores the base values.

## 15. Validation and integration test examples

The profile validator MUST additionally check:

- consistency of part IDs and resolvability of the channels declared by the model;
- correct pivots, unit axes and baked mechanism scale;
- clip ranges and an unambiguous writer of each property;
- existence of the `pantograph.animation` and `contact_node` references;
- monotonicity of the pantograph table, its coverage and the contact tolerance;
- structure, lengths and rest plane of the IK chain;
- default visibility states of all variants;
- display materials, fonts and emitter resources.

Pipeline integration tests SHOULD cover:

| Test | Expected result |
|---|---|
| Two vehicles from the same asset with different light states | Independent lights and lens emission |
| Door 0 → 0.5 → 1 → 0.2 | Exact pose sampling, consistent motion reversal |
| Wheel through several revolutions and a sign change | Continuous rotation consistent with the axis |
| Bogie yaw with spinning wheels | Correct transform composition |
| Headlight off → dimmed → full | Correct intensity and selected geometry |
| Coupler peer / owner / adapter | Correct representation at each end |
| Pantograph at the extremes and between table rows | Contact within tolerance; correct limit position |
| IK target reachable, at the limit and collinear with the pole | Stable, deterministic pose |
| Switching camera cab/external/reflection | Correct subtrees and billboards |
| Emitter on a moving and turning vehicle | Correct velocity inheritance |
| Re-export with a different node order | IDs preserved and index references fixed |

## 16. Standard and mapping sources

- [Common rules and glTF sources](MODEL_FORMAT_EN.md).
- [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer).
- [KHR_node_visibility](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_node_visibility).
- [KHR_lights_punctual](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_lights_punctual).
- Legacy sources: `vehicle/AirCoupler.cpp` (`Init`, `Update`), `vehicle/AirCoupler.h`, `vehicle/DynObj.cpp` (coupler handling and `DimHeadlights`, part prefix initialization), `vehicle/Button.cpp`, `vehicle/Gauge.cpp`, `model/AnimModel.cpp`.
