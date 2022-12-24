# wgpu-mc shaderpack spec v0.0.1-SNAPSHOT

This document details the format for shaderpacks written for the wgpu-mc rendering engine.


## Config file

The config file is the entry point for wgpu-mc into the shaderpack. It specifies to wgpu-mc what resources and shaders are available, along with information about them, and how to composite them to render the final image. The config file is written in yaml.

The config file must be located at the root of the shaderpack, and must have the exact file name "shaderpack.yaml"

A config file has the following fields:
```yaml
version: "v0.0.1-SNAPSHOT"
support: wgsl # specifies the shading language. must be *either* glsl or wgsl
resources:
  ...
pipelines:
  ...
```

### Resources

The core way that shaderpacks define uniforms which are passed into shaders, and how that data is provided, is through **resources**. 
wgpu-mc provides many different built-in resources, which are used as the base for creating most other resources. All built-in resources are prefixed with `wm_`.

There are multiple different kinds of resources:
- geometry
- textures
- blobs
- matrices
- numbers

Specify your resources like this:

```yaml
resources:
  sky_box:
    type: texture_3d
    src: skybox.png
```

Example for a floating-point variable which can be changed by the player:

```yaml
# this makes it possible for the user to change it in a settings menu
blur:
  type: f32
  range: [0.0, 0.30]
  value: 0.0
  desc: This is an optional description for users
```

If you don't want to specify anything other than the default value, then for numbers and matrices this shorthand exists:

```yaml
int: 30 # type gets inferred to i32
float: 3.0 # type gets inferred to f32
matrix: # type gets inferred to mat4
  - [0.0, 0.3, 0.4, 0.5]
  - [3.0, 3.0, 3.0, 3.0]
  - [3.0, 3.0, 3.0, 3.0]
  - [3.0, 3.0, 3.0, 3.0]
```

It's possible to show values as settings to the user by specifying the `show` attribute. The user can then change these in a menu. Specifying a `range` attribute implies `show: true`

```yaml
lense_flare:
  type: f64
  show: true
  value: 64.0
```

Matrices can be specified either like this:

```yaml
matrix:
  type: mat4
  value:
    - [0.0, 0.3, 0.4, 0.5]
    - [3.0, 3.0, 3.0, 3.0]
    - [3.0, 3.0, 3.0, 3.0]
    - [3.0, 3.0, 3.0, 3.0]
```

Or like this:

```yaml
matrix:
  type: mat3
  value: [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], [0.7, 0.8, 0.9]]
```

### Geometry

Geometry implies both the vertex layout that a vertex shader must use, and the vertex buffer objects (VBOs) that are bound during a draw call.

Geometry is, at this time, only provided internally by wgpu-mc, as built-ins. In the future, you may be able to specify geometry through the use of the file-system, or through a compute-shader stage.

Some geometry may imply that the pipeline is (opaquely) rendered multiple times per frame, like with entities `wm_geo_entities`.

The currently existing geometry resources are defined as such:

`wm_geo_terrain`

Vertex layout:
- 0, position: `vec3f`
- 1, color: `vec4f`
- 2, albedo_uv: `vec2f`
- 3, overlay_uv: `vec2f`
- 4, lightmap_uv: `vec2f`
- 5, normal: `vec3f`
- 6, (TODO), `vec3f`
- 7, sprite_center_uv: `vec2f`
- 8, tangent: `vec4f`
- 9, (TODO), `vec3f`
- 10, block_middle_offset: `vec3f`

---
`wm_geo_entities`

TBD

---
`wm_geo_transparent`

Same as `wm_geo_terrain`

---
`wm_geo_fluid` 

Same as `wm_geo_terrain`

---
`wm_geo_skybox` 
A simple unit cube, with side lengths of 1. Centered around the origin (0,0,0)

Vertex layout:
- 0: vec3\<f32\> (position)
- 1: vec2\<f32\> (UV)

---
`wm_geo_quad` 

Vertex layout:
- 0: vec2\<f32\> (position)
- 1: vec2\<f32\> (UV)

### Textures

Textures can be created through three methods:
- being provided by wgpu-mc as a built-in
- as the result of a pipeline
- being defined in the resources

Resource fields:
- `src`: string
- `clear_each_frame`: bool

Shaders which are defined in the resources may provide a resource location through the `src` field. This field must be a valid path within the shaderpack to a valid PNG file. No other file types are supported at this time.

If the `src` field is not present, that texture will be created when the shaderpack is initialized, but left blank.


There are multiple types of textures:
- texture_2d
- texture_3d
- texture_depth

Built-ins:
- framebuffer_texture_2d
- framebuffer_depth

Example:

```yaml
resources:
    depth_texture:
        type: texture_depth
        clear_after_frame: true
```

### Blobs

Blobs are binary data which are passed to shaders in the form of an SSBO (shader storage buffer object). 
In the future, you may be able to create blobs through the use of the OS file-system, or through a compute-shader stage, however at the moment thsese are only provided as built-ins.

Blobs:
- `wm_ssbo_entity_part_transforms`: `array<mat4<f32>>` (note: only accessible in pipelines where the geometry field is set to `wm_geo_entities`) (still todo)

### Matrices/numbers

Such a resource may have one of the following types:

- **f32**
- **f64**
- **u32**
- **i32**
- **mat3**
- **mat4**

`f32`s, `f64`s, `u32`s, and `i32`s have the following resource field(s)
- value: number

This value must be a valid instance of the respective type. As in, you cannot specify that the resource has type `i32` and have the `value` be set to `0.5`.

mat3 and mat4 are defined respectively as a 3x3 or 4x4 matrix of 32-bit floating point numbers.

`mat3`s and `mat4`s can specify the `mult` field. The `mult` field is an array of matrices (important: with the same type, as well as either built-in or specified before in the file - order matters) which are then multiplied together to create the resulting matrix.

You cannot specify both `mult` and `default` in one matrix.

Example:

```yaml
resources:
    mvp_mat4:
        type: mat4
        mult: [wm_model_mat4, wm_view_mat4, wm_projection_mat4]
```
This multiplies together the three specified matrices.

The following matrices are provided by wgpu-mc by default:

- `wm_model_mat4`
- `wm_view_mat4`
- `wm_projection_mat4`
- `wm_view_rotation_mat4` (same as `wm_view_mat4` but it is only composed of the rotation, and not any other properties such as translation)
- `wm_view_inverse_mat4` (inverse of `wm_view_mat4`)
- `wm_projection_inverse_mat4` (inverse of `wm_projection_mat4`)
- `wm_model_inverse_mat4` (inverse of `wm_model_mat4`)

Some of these matrices are (opaquely) context-specific, depending on the chosen `geometry` for a given pipeline.

## Pipelines

Pipelines are not fixed, and are entirely user-defined. Meaning you could write a shaderpack that doesn't render entities, at all. However, that's not something most players would expect.

Pipelines names are flexible. Their outputs do not have to match up their respective names, however the pipeline name does imply the name(s) of file(s) of the shader program(s).

Pipelines are executed **in the same order they are defined**.

Fields:

`geometry`: (required) Pipelines have geometry. Geometry is defined in the [Geometry](#Geometry) section. The value of this field must be a valid reference into a resource with the type `geometry`.

`uniforms`: Pipelines may have uniforms, where you must specify an explicit binding location for each uniform, and also to which shader programs the uniforms are visible, such as either the vertex shader (denoted as `vert`), or the fragment shader (denoted as `frag`).

`output`: Pipelines must have zero or more output textures, also known as color attachments. They may have up to 8 output textures.

`depth`: Pipelines may or may not write to a depth buffer. If they do not, then no `depth` field is provided. If there is a `depth` field provided, then the referenced resource must have the type `texture_depth`.

Example:
```yaml        
pipelines:
  terrain:
    geometry: wm_geo_terrain
    depth: terrain_depth # defined in the resources section of the config
    output: [terrain_output]
    uniforms:
      0: # binding location
        resource: mvp_mat4
        visibility: [vert]
```

## Assorted examples

Shadowmap

```yaml
version: "0.0.1-SNAPSHOT"
support: glsl # could also be wgsl
resources:
  shadowmap_texture_depth:
    type: texture_depth
    clear_after_frame: true
  shadow_ortho_mat4:
    type: mat4
    value: # this is just an identity matrix, it would be something different in practice
      - [1.0, 0.0, 0.0, 0.0]
      - [0.0, 1.0, 0.0, 0.0]
      - [0.0, 0.0, 1.0, 0.0]
      - [0.0, 0.0, 0.0, 1.0]
  model_view_mat4:
    type: mat4
    mult: [wm_model_mat4, wm_view_mat4]
  mvp_mat4:
    type: mat4
    mult: [wm_model_mat4, wm_view_mat4, wm_projection_mat4]
pipelines:
  terrain_shadows:
    geometry: wm_geo_terrain # one
    depth: shadowmap_texture_depth
    uniforms:
      0:
        resource: model_view_mat4
        visibility: [vert]
      1:
        resource: shadow_ortho_mat4
        visibility: [vert]

  entity_shadows:
    geometry: wm_geo_entities
    depth: shadowmap_texture_depth
    uniforms:
      0:
        resource: model_view_mat4
        visibility: [vert]
      1:
        resource: shadow_ortho_mat4
        visibility: [vert]
      2:
        resource: wm_ssbo_entity_part_transforms
        visibility: [vert]

  terrain:
    geometry: wm_geo_terrain
    depth: wm_framebuffer_depth
    output: [wm_framebuffer_texture]
  entities:
    geometry: wm_geo_entities
    depth: wm_framebuffer_depth
    output: [wm_framebuffer_texture]
    uniforms:
      0:
        resource: model_view_mat4
        visibility: [vert, frag]
      1:
        resource: shadow_ortho_mat4
        visibility: [vert, frag]
      2:
        resource: wm_ssbo_entity_part_transforms
        visibility: [vert, frag]
      3:
        resource: mvp_mat4
        visibility: [vert, frag]
      4:
        resource: shadowmap_texture_depth
        visibility: [frag]
```

---