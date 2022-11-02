# Internals

The core of rendering operations is the `Renderer` class, it contains all the caches, state and functions to render frames.

## Renderer

The main goals of the renderer are (from highest to lowest level):

* Take a list of `RenderSteps` and allocate render targets for each intermediate render texture and connect them to the inputs/outputs of the correct passes
* Run each `RenderStep` in the correct order and render the `Drawables` that belong to them
* Group and batch `Drawables` into render pipelines and draw them using those pipelines.

The main entry point for rendering is the render() call which takes the following arguments:

* A view, describing the main viewer of the frame to render (camera, shadow caster location, light probe capture location, etc.)
* A list of `RenderSteps`, describing the render graph, which passes to render and what effects in what order.

## Render Steps

At the highest level the renderer receives a list of `RenderSteps` to process.

Steps are considered immutable by instance (shared pointer), so they are hashed inside `RendererImpl::getOrBuildRenderGraph` to cache the generated render graph. Alongside this the input/output to the render graph is hashed as well so that the graph is recomputed if these change.

When not in the cache a render graph is built out of the given render steps using `RendererImpl::buildRenderGraph`

It looks at all of the Inputs/Outputs of the `RenderSteps` and connects outputs from previous steps to inputs of later steps if they have the same name

```clojure
; This step outputs to 2 named render targets: 'color' and 'depth'
{:Outputs ["color" "depth"]} >> .steps
; This second step receives the 'color' and 'depth' texture written by the first step as input
; This second step writes to the same color output as the previous step
{:Inputs ["color" "depth"] :Outputs ["color"]} >> .steps
```

Additionally outputs may specify if they load the previous texture contents or clear them

```clojure
{:Outputs [{:Name "color"}]} >> .steps
; This step writes to the same 'color' texture without clearing
{:Outputs [{:Name "color"}]} >> .steps
; This step overwrites the previous contents of 'color'
{:Outputs [{:Name "color" :Clear true}]} >> .steps
```

### RenderGraphBuilder

The `RenderGraphBuilder` handles making connections and building the `RenderGraph` object which is then used and cached.

The function `RenderGraphBuilder::attachOutput` is used to define an output for the entire graph that can be connected to for example the applications main output or VR headset.

> The graph is still only built once, but the outputs can change every time the graph is evaluated (as long as the format & size is the same).

`RenderGraphBuilder::allocateInputs` defines named inputs for a node to reuse from previous node outputs

`RenderGraphBuilder::allocateOutputs`defines named outputs that can be used in later nodes or can be attached to a final output

The builder is also responsible for validating the connections and inserting additional operations where applicable, think of:

* Copying a texture to a different format/size
* Up/Down-sampling a texture
* Resolving multi-sampled textures

### RenderGraph

The result from the `RenderGraphBuilder` is a `RenderGraph` that contains:

* A list of `Frames`, each of which will be assigned a texture
* A list of `RenderGraphNodes`, which say which `Frames` they read and write to.
* A list of `Outputs`, which points to which frames will contain the final output of the graph

Each node also builds a `RenderTargetLayout` which describes the layout of the render pass attachments later passed to `wgpuCommandEncoderBeginRenderPass`. This layout is also used during the drawable grouping since the generated `WGPURenderPipeline` and `Shader` will be different based on the layout.

### RenderGraphEvaluator

The `RenderGraphEvaluator` is a persistent object that is used to evaluate `RenderGraphs`, it assigns temporary textures to `Frames`. It also takes a list of output textures, previously declared using `RenderGraphBuilder::attachOutput` and and connects them to the output frames of the render graph.

Each `RenderGraphNode` evaluates the contents of a single `RenderStep`.

> This can be optimized to run multiple `RenderSteps` in a single `RenderGraphNode` if their layout matches to prevent superfluous render passes, which are especially expensive on systems using tiled renderers

### Evaluated steps

Some render steps will only render a single geometry, for example:

* `ClearStep` will explicitly clear the specified outputs with given clear values
* `RenderFullscreenStep` will render a single full-screen geometry with given shader code

The `RenderDrawablesStep` is where large lists of `drawables` are grouped and renderer, see [Pipeline grouping](Internals.md#pipeline-grouping)

## Pipeline grouping

Each `RenderStep` might render one or multiple `Drawables` from a `DrawQueue` specified inside the `RenderStep`

`DrawableGrouper::groupByPipeline` contains the hashing functionality to group `Drawables` by render pipline.

The main things that are used to build this hash:

* List of features attached to drawable & material
* Mesh vertex format
* The `defaultTexcoordBinding` on material texture parameters
* The render target layout for the `RenderGraphNode` that is rendering the given `Drawable`

> `defaultTexcoordBinding` indicates the default texture coordinates to use when sampling a texture

The result is a `PipelineDrawableCache` containing a `map<Hash128, PipelineDrawables>`. Each key corresponds to a unique `WGPURenderPipeline`, the value contains the list of `drawables` belonging to this pipeline.

Additionally, inside `sortAndBatchDrawables`, `Drawables` are sorted and ones using the same texture bindings & meshes are grouped together.

> This would be the place to perform CPU side frustum culling too

The result of sorting is stored in `PipelineDrawables::drawablesSorted`. The result of batching is stored in `PipelineDrawables::drawGroups` which contains ranges of sorted drawables to render

All float/vector shader parameters for the `drawables` in a `PipelineDrawables` are stored in a single buffer, the ordering is based on the `PipelineDrawables::drawablesSorted` and shader buffer accesses are offset by the index in this array.

## Drawable rendering

After grouping and sorting each drawable is rendered, iterating over pipelines rendering all `Drawables` for each pipeline first to prevent switching. `renderPipelineDrawables` is the function that does this.

This is also where the buffers that are bound are created by calling `setupBuffers`. It fills the per-view and per-object buffers.

## Pipeline building

After drawable hashing, if the pipeline they belong to is not build yet, it is done shortly after in `Renderer::buildPipeline`.

`PipelineBuilder` is used to store temporary data while building pipelines. It's main entry point is the `build()` function

### Collect buffer bindings

The first step is collecting the layout buffer bindings. Some built-in parameters (such as "view", "proj", "world", etc.) are set first. After that all parameters are taken from referenced features and added to their respective layouts.

> Currently all shader parameters go into the per-object buffer

### Collect shader entry points

Next all shader entry points are collected from referenced features.

### Optimize bindings

`shader::Generator::indexBindings` is ran on the shader entry points to do a first pass on the shader and collect information about which shader parameters, textures and outputs are used.

`PipelineBuilder::optimizeBufferLayouts` is ran to remove fields from buffers that are not used by the shader

### Build layout

`PipelineBuilder::buildPipelineLayout` takes the list of object/view bindings and converts it to bind group layouts. Texture bindings are also included into the bind group that is bound per-object.

Separate bind groups are created:

* One for per-object buffer & textures
* One for per-view buffer

After this the pipeline layout is built based on these bind group layouts.

### Finishing

Finally `PipelineBuilder::finalize` is called where:

* A call to `generateShader()` generates the WGSL shader code
* The pipeline state is computed by combining the state from all referenced features in `computePipelineState()`
* Mesh layout is collected from the pipeline's `MeshFormat`
* Render targets are collected from the pipeline's `RenderTargetLayout`
* Everything is combined into the input to `wgpuDeviceCreateRenderPipeline()` and the result is returned

## Shader generator

The shader generator operates on some kind of syntax tree of nodes called `Blocks` that embed WGSL shader code annotated with information relevant to the operation. The final result is a single string of WGSL source code.

an example of a `Block` is the `ReadBuffer` block:

```cpp
  std::string fieldName;
  FieldType type;
  std::string bufferName;
```

It contains the information required to access a field from a buffer binding by giving the buffer name, the field name inside the buffer and the expected type of the value.

Using this structure allows the generator to know better what the shader code does, what fields it accesses, what inputs are used, what outputs are written.

### Inputs

The shader generator (`gfx::shader::Generator`) takes the following inputs:

* A description of vertex attributes / mesh format (`gfx::MeshFormat`)
* The texture bind group index
* A list of output fields
* A list of buffer bindings (`gfx::shader::BufferBinding`)
* A list of texture bindings (`gfx::shader::TextureBinding`)
* A list of `gfx::shader::EntryPoint`

The entry points define a single function that is injected into the final shader. An entry point describes:

* The shader stage that it applies to (vertex/fragment)
* Any other entry points it depends on (by name)
* The `gfx::shader::BlockPtr` pointing to the root block of the shader syntax tree

### Building

`build()` is the starting point of the shader generator.

First the following is done:

* All buffer structures are converted into WGSL structure definitions.
* The vertex format is converted into a vertex input structure and defined in WGSL
* The output fields are converted into a fragment output structure and defined in WGSL
* Texture bindings are converted to WGSL

Next, the generator goes over all the shader entry points for each stage. The vertex stage is processed first. Then the fragment stage. Shader entry points are sorted topologically before they are processed (based on their dependencies).

For each stage `gfx::shader::Stage::process` defines the input/output variables, it defines a main function and inserts calls to the generated entry point functions in the correct order. Inputs/outputs from stages are accessed through in intermediate `var<private>` and assigned later returned by the actual main function.

Communication between entry points happens through global variables, each usage of the block `WriteGlobal` generates a field in the `var<private> globals`

While the `process` function is looping over entry points it defines a function body in WGSL and calls `apply` on the entry point's shader `Block` passing in the `gfx::shader::GeneratorContext`. The `GeneratorContext` has functions that when called will generate WGSL output in the current output stream.

Each block either writes raw WGSL code into the output stream using `context.write("some string")` or uses one of the helper function such as `context.readGlobal("global name")`, `context.writeOutput("name", <type>)`, etc.

The `GeneratorContext` also keeps a list of all the inputs/outputs/textures and buffers so that they may be inspected by blocks to generate different output based on these.

## Shader translator

When defining shader code in shards, the shader translator is involved. It takes a list of shards and converts compatible shards into their shader block counterparts.

The context, `gfx::shader::TranslationContext`, is passed around to translators or translatable shards to generate the resulting shader `Block`

`TranslationContext::processShard` takes a single shards and appends the generated shader `Blocks` to the context's root shader `Block`, recursively where necessary

Translation for native shader shards is done through the `translate(context)` function on the shard, they are registered using `REGISTER_SHADER_SHARD("Name", ShardClass)`

Translation for external shards - meaning shards that are not specifically for shaders - is done through translators. Translators have a single static function:

```cpp
static void translate(TShard *shard, TranslationContext &context) {}
```

where TShard is the shard type. They are registered using `REGISTER_EXTERNAL_SHADER_SHARD(TranslatorClass, "Name", ShardClass)`

### Shard matching

During translation shards are matched to their registered translator by name, to receive the actual shard instance to pass to the translator, each of the external shards are assumed to be registered using `ShardWrapper` or it's macros.
