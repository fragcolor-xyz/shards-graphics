# User API

The user-facing side of the renderer deals with the following concepts

## Drawables

Wraps the state and parameters for a single instance of something to be passed to the renderer

Examples of `drawables` are: static meshes, skeletal meshes, height map terrains, particle systems, CSG models.

## Draw queues

A `DrawQueue` is a list of `Drawables`.

They are used to pass around lists of `Drawables` to other components, for example the `DrawablePass`

## Features

Features change how drawables are rendered.

They can contain the follow:

* Pipeline State
  * Z Write Enabled?
  * Color Write Mask?
  * Blend State (alpha/additive/etc.)
  * Depth Test
* `Parameter` definitions
* Shader `entry points`

`Features` can be applied to an entire frame, a single render pass or even individual `drawables`.
This way they represent a common interface for modifying the render pipeline at multiple levels.

## Parameter definitions

`Features` may contain `parameter` definitions.
Every drawable can have their own set of `parameters`. `Parameters` are passed to the GPU as buffers and texture bindings per drawable.

`Parameters` need to be defined in a feature before they are available within shader `entry points`.

They can be either number/vector values, e.g.:\
`:Params [{:Name "lightDirection" :Default (Float3 0.0 0.0 1.0)}]`\
`:Params [{:Name "lightDirection" :Type Float32 :Dimension 3}]`

Or they can be texture bindings:\
`:TextureParams [{:Name "normalMap"}]`

Any `parameters` that is set on a `drawable` that is not defined in a `feature` is ignored and not accessible within shader `entry points`

## Shader entry points

All shader code is wrapped in shader `entry points`

When a `drawable` is being rendered, all `features` and their shader entry points are collected.

The `entry points` are topologically sorted based on their dependencies and translated to a shader that calls them based on this sorting

### Example

Assume a pixel shader `entry point` named `ComputeLighting` exist to compute lighting color

```clojure
{
  :Stage Fragment
  :Name "ComputeLighting"
  :Code (->
    ...
    .lightColor (Shader.WriteGlobal "lightColor")
  )
}
```

Another entry point could reference the global output `lightColor` that has been written here by adding a dependency on the `ComputeLighting` entry point:

```clojure
{
  :Stage Fragment
  :Dependencies [{:Name "ComputeLighting"}]
  :Code (->
    (Shader.ReadGlobal "lightColor") >= .lightColor
    ... ; Do something with the light color
  )
}
```

## View/Camera

A `view` describes the primary viewer for a rendered frame. Typically this is described as a camera that has a world transform and some projection parameters

## Render Steps

A `render step` can be though of as a render pass or a sub-section of a render pass.

In an application frames are rendered with a call to `GFX.Render` and passed a list of `render steps` that make up the frame, for example:

```clojure
{... :Queue .queue-1} (GFX.DrawablePass) >> .render-steps
{... :Queue .queue-2} (GFX.DrawablePass) >> .render-steps
{...} (GFX.EffectPass) >> .render-steps
{...} (GFX.EffectPass) >> .render-steps
{...} (GFX.EffectPass) >> .render-steps

(GFX.Render :View .view :Steps .render-steps)
```

Each step is queued to run on the GPU in order or might run in parallel if they don't depend on each other

Different kinds of render steps exist:

### Drawable Pass

A `DrawablePass` takes some `Drawables` from a `DrawQueue` and renders them.

```clojure
(GFX.DrawQueue) >= .queue
(GFX.BuiltinFeature BuiltinFeatureId.Transform) >> .features
(GFX.BuiltinFeature BuiltinFeatureId.BaseColor) >> .features
{:Features .features :Queue .queue} (GFX.DrawablePass) >= .render-step
```

### Effect Pass

An `EffectPass` takes outputs from previous render steps and applies fullscreen shader `entry points` to them

```clojure
{
  :Inputs ["color"]
  :Outputs [{:Name "color" :Format RGBA8}]
  :EntryPoint (->
    (Shader.SampleTexture "color") >= .color
    ... ; Some effect shader code
    .color (Shader.WriteOutput "color")
  )
} (GFX.EffectPass)
```

### Implicit input/outputs

When not specified, render steps have some implicit inputs/outputs defined.

`DrawablesPasses` have a `"color"` and `"depth"` (depth buffer) output

`EffectPasses` have a `"color"` input and output