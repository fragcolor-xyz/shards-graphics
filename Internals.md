# Internals

The core of rendering operations is the `Renderer` class, it contains all the caches, state and function to render frames.

## Renderer

The main goals of the renderer are (from highest to lowest level):

- Take a list of `RenderSteps` and allocate render targets for each intermediate render texture and connect them to the inputs/outputs of the correct passes
- Run each `RenderStep` in the correct order and render the `Drawables` that belong to them
- Group and batch `Drawables` into render pipelines and draw them using those pipelines.

The main entry point for rendering is the render() call which takes the following arguments:

- A view, describing the main viewer of the frame to render (camera, shadow caster location, light probe capture location, etc.)
- A list of `RenderSteps`, describing the render graph, which passes to render and what effects in what order.

## Render Steps

At the highest level the renderer receives a list of `RenderSteps` to process.

Steps are considered immutable by instance (shared pointer), so they are hashed inside `RendererImpl::getOrBuildRenderGraph` to cache the generated render graph.
Alongside this the input/output to the render graph is hashed as well so that the graph is recomputed if these change.

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

The builder

### RenderGraph

The result from the `RenderGraphBuilder` is a `RenderGraph` that contains:

- A list of `Frames`, each of which will be assigned a texture
- A list of `RenderGraphNodes`, which say which `Frames` they read and write to.
- A list of `Outputs`, which points to which frames will contain the final output of the graph

Each node also builds a `RenderTargetLayout` which describes the layout of the render pass attachments later passed to `wgpuCommandEncoderBeginRenderPass`. This layout is also used during the drawable grouping since the generated `WGPURenderPipeline` and `Shader` will be different based on the layout.

### RenderGraphEvaluator

The `RenderGraphEvaluator` is a persistent object that is used to evaluate `RenderGraphs`, it assigns temporary textures to `Frames`.
It also takes a list of output textures, previously declared using `RenderGraphBuilder::attachOutput`  and and connects them to the output frames of the render graph.

Each `RenderGraphNode` evaluates the contents of a single `RenderStep`.
> This can be optimized to run multiple `RenderSteps` in a single `RenderGraphNode` if their layout matches to prevent superfluous render passes, which are especially  expensive on systems using tiled renderers

### Evaluated steps

Some render steps will only do a single thing, for example:

- `ClearStep` will explicitly clear the specified outputs with given clear values
- `RenderFullscreenStep` will render a single full-screen geometry with given shader code

The `RenderDrawablesStep` is where large lists of `drawables` are grouped and renderer, see [Pipeline grouping](#pipeline-grouping)

## Pipeline grouping

Each `RenderStep` might render one or multiple `Drawables` from a `DrawQueue` specified inside the `RenderStep`

`DrawableGrouper::groupByPipeline` contains the hashing functionality to group `Drawables` by render pipline.

The result is a `PipelineDrawableCache` containing a `map<Hash128, PipelineDrawables>`. Each key corresponds to a unique `WGPURenderPipeline`, the value contains the list of `drawables` belonging to this pipeline.

Additionaly the drawables are sorted and batched per-pipeline inside `sortAndBatchDrawables`. For example, Drawables using the same texture bindings & meshes are batched.

> This would be the place to perform CPU side frustum culling too

The result of sorting is stored in `PipelineDrawables::drawablesSorted`. The result of batching is stored in `PipelineDrawables::drawGroups` which contains ranges of sorted drawables to render

All float/vector shader parameters for the `drawables` in a `PipelineDrawables` are stored in a single buffer, the ordering is based on the PipelineDrawables::drawablesSorted and shader buffer accesses are offset by the index in this array.

## Drawable rendering

## Pipeline building

## Shader generator
