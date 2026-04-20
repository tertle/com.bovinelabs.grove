# BovineLabs Grove

BovineLabs Grove is a graph authoring and runtime package for Unity Entities. Grove graphs are authored with Unity Graph Toolkit, imported into Grove authoring assets, baked into blob assets, and executed from ECS systems.

Access and updates are provided on [Buy Me a Coffee](https://buymeacoffee.com/bovinelabs) and support on [Discord](https://discord.gg/RTsw6Cxvw3).

> Grove `2.0.0-exp` is the Graph Toolkit release. GraphView-era graphs and editor workflows are gone, and there is no backwards compatibility layer for old assets. If you need the existing GraphView workflow, use the `0.9.0` release.

## Installation

Grove targets Unity `6000.5` and depends on `com.bovinelabs.core`.

If you are using a Grove release bundle:

1. Extract `com.bovinelabs.grove` into your project's `Packages/` folder.
2. Make sure `com.bovinelabs.core` is also present.
3. Open Unity and let the package import.

## Sample

The package includes a working sample that shows the current Graph Toolkit workflow end to end.

To import it:

1. Open `Window > Package Manager`.
2. Select `BovineLabs Grove`.
3. Open the `Samples` section.
4. Import `Sample`.

After import:

1. Open `Samples/BovineLabs Grove/[version]/Sample/Scenes/SampleScene`.
2. Enter Play Mode.
3. Double-click `MyGraph.mygraph` in the sample data folder to open the graph source asset.

The sample demonstrates:

- A custom graph type and importer.
- Custom execution and data nodes.
- An imported `GroveAuthGraph` main asset.
- A `GraphAuthoring` component and `GraphBakingSystem`.
- A runtime `ISystem` executing baked graph blobs.
- Variants and instance overrides.

## The Pipeline

Grove's authoring pipeline is:

1. Create a Graph Toolkit graph asset such as `MyGraph.mygraph`.
2. Import it with `GroveImporter<TAuth>`.
3. The importer creates a `GroveAuthGraph` main object plus auth-node subassets.
4. `GraphAuthoring` or `GraphBufferAuthoring` references that imported auth asset.
5. `GraphBakingSystem` converts it into `BlobAssetReference<GraphData>`.
6. `GraphImpl<TContext>` and `GraphExecution<TContext>` run it in ECS.

That split is deliberate:

- `BovineLabs.Grove.Editor`: Graph Toolkit nodes and import.
- `BovineLabs.Grove.Authoring`: imported auth assets and baking.
- `BovineLabs.Grove`: runtime graph execution.

## Core Concepts

### Graph Assets

A Grove graph starts as a Graph Toolkit asset that derives from `GroveGraph`.

```csharp
using System;
using BovineLabs.Grove.Editor;
using Unity.GraphToolkit.Editor;
using UnityEditor;

[Serializable]
[Graph(AssetExtension)]
public class MyGraphGraph : GroveGraph
{
    public const string AssetExtension = "mygraph";

    [MenuItem("Assets/Create/BovineLabs/My Graph")]
    private static void CreateAssetFile()
    {
        GraphDatabase.PromptInProjectBrowserToCreateNewAsset<MyGraphGraph>("MyGraph");
    }
}
```

Import that graph into an authoring asset:

```csharp
using BovineLabs.Grove.Authoring;
using BovineLabs.Grove.Editor;
using UnityEditor.AssetImporters;

[ScriptedImporter(1, MyGraphGraph.AssetExtension)]
public class MyGraphImporter : GroveImporter<MyGraphAuth>
{
}

public class MyGraphAuth : GroveAuthGraph
{
}
```

The imported `MyGraphAuth` is the asset you reference from authoring components and bakers.

### Nodes

Custom Grove nodes are split across assemblies:

- Editor assembly: Graph Toolkit node definition and ports.
- Authoring assembly: auth asset that serializes into blob data.
- Data assembly: unmanaged blob payload types and enums.
- Runtime assembly: `[ExecuteNode]` or `[DataNode]` methods.

#### Execution Node Example

```csharp
// Data assembly
public enum ExecutionType
{
    Default = 0,
    Move = 1,
}

public struct ExecuteMoveData
{
    public float Speed;
}

// Authoring assembly
public sealed class ExecuteMoveAuth : GroveExecutionAuth<ExecuteMoveData>
{
    public float Speed = 1;

    public override int NodeType => (int)ExecutionType.Move;

    protected override void Init(ref BlobBuilder builder, ref ExecuteMoveData execution, IGroveAuthState state)
    {
        execution.Speed = this.Speed;
    }
}

// Editor assembly
[Serializable]
[UseWithGraph(typeof(MyGraphGraph))]
public sealed class ExecuteMoveNode : GroveExecutionNode<ExecuteMoveAuth, ExecuteMoveData>
{
    public float Speed = 1;

    protected override void Init(ExecuteMoveAuth auth, IGroveNodeState state)
    {
        auth.Speed = this.Speed;
    }
}

// Runtime assembly
public static class ExecuteMove
{
    [ExecuteNode((int)ExecutionType.Move)]
    public static void Execute(in ExecuteMoveData data, in EntityContext entityContext, ref MyContext context)
    {
        ref var transform = ref context.LocalTransform.GetRW(entityContext.EntityIndexInChunk).ValueRW;
        transform.Position.x += data.Speed * entityContext.DeltaTime;
    }
}
```

`[UseWithGraph]` is how custom Graph Toolkit nodes are scoped to a graph type.

#### Data Node Example

Data nodes follow the same pattern, but return values instead of driving execution:

```csharp
[DataNode((int)DataType.IsDirectionOld)]
public static float Calculate(in ScoreIsDirectionOldData data, in EntityContext entityContext, ref MyContext context)
{
    var state = context.GetState(entityContext.EntityIndexInChunk);
    ref var lastDirectionChange = ref state.GetOrAddRef((short)StateKeys.LastDirectionChange, double.MinValue);
    var isOld = lastDirectionChange + data.Duration <= entityContext.ElapsedTime;
    return data.Score.Score(isOld);
}
```

### Contexts And Source Generation

Your runtime context implements `IContext<T>`. The source generator fills in the boilerplate required to wire component containers, chunk setup, and dispatch for your custom node methods.

```csharp
using BovineLabs.Grove.Core;
using BovineLabs.Grove.Utility;
using Unity.Transforms;

public partial struct MyContext : IContext<MyContext>
{
    public ComponentContainer<LocalTransform> LocalTransform;
    public BufferContainer<GraphState> GraphStates;

    public DynamicUntypedHashMap<short> GetState(int index)
    {
        return this.GraphStates.GetRW(index).AsMap();
    }
}
```

You only write the context members you actually need. The source generator handles:

- `CreateGenerated`
- `UpdateGenerated`
- `SetChunkGenerated`
- dispatch to `[ExecuteNode]`, `[DataNode]`, and debug methods

If you use graph boundary inputs or outputs, also implement `GetVariables()` on the context and back it with a `NativeUntypedHashMap<Hash128>`.

### Boundary Variables

Graph Toolkit variables are how Grove defines graph boundaries.

- Root graph `Input` variables become graph inputs.
- Root graph `Output` variables become graph outputs.
- Local variables stay local to the graph scope.
- Subgraph boundary variables are resolved through the importer, including nested subgraphs.

For execution-style graphs, an `Input` boundary variable connected to an execution node defines the explicit entry point.

For calculation-style graphs, `Output` boundary variables expose typed outputs you can query at runtime.

At runtime this enables APIs like:

```csharp
execution.Execute(entity, entityIndexInChunk, graphBlob, inputValue);
var result = execution.Calculate<int, float>(entity, entityIndexInChunk, graphBlob, inputValue);
```

If a graph has a single matching input or output type, Grove can resolve it by type. If you need multiple inputs or outputs of the same type, use the variable key overloads.

### Subgraphs

Native Graph Toolkit subgraphs are supported. The importer walks subgraphs, scopes node identities correctly, and resolves boundary-variable connections across parent and child graph scopes.

You do not need a separate Grove-specific subgraph asset type. Use Graph Toolkit subgraphs directly.

### Variants

`GroveVariantNode` selects one execution branch during baking.

- Add a variant node to the graph.
- Connect each output to a different execution branch.
- Set the default branch on the node.
- Override the selected branch per graph instance in the `Variants` foldout on the authoring component.

Branches that are not selected are stripped out of the baked graph.

### Instance Overrides

Graph instances can override serialized auth-node fields per entity.

This is useful when you want multiple entities to share one graph asset but vary values such as speeds, durations, radii, thresholds, or other serialized configuration.

The `GraphInstance` inspector:

- builds the available override list from imported auth nodes,
- hides Grove node and graph references,
- respects `[HideInstance]`,
- auto-adds fields marked with `[AlwaysAddInstance]`, and
- keeps overrides stable across prefab revert/import refreshes.

## ECS Integration

### Runtime Graph Component

Your ECS component or buffer element implements `IGraphReference`:

```csharp
using Unity.Entities;

public struct MyGraph : IComponentData, IGraphReference
{
    public BlobAssetReference<GraphData> Graph;

    unsafe ref BlobAssetReference<GraphData> IGraphReference.GraphRef
    {
        get
        {
            fixed (MyGraph* ptr = &this)
            {
                return ref ptr->Graph;
            }
        }
    }
}
```

### Authoring Component

Use `GraphAuthoring<TGraph, TAuth>` for a single graph or `GraphBufferAuthoring<TGraph, TAuth>` for a buffer of graphs.

```csharp
using BovineLabs.Grove.Authoring;
using Unity.Entities;
using UnityEngine;

[DisallowMultipleComponent]
public class MyGraphAuthoring : GraphAuthoring<MyGraph, MyGraphAuth>
{
    private class Baker : Baker<MyGraphAuthoring>
    {
        public override void Bake(MyGraphAuthoring authoring)
        {
            base.Bake(authoring);
            this.AddBuffer<GraphState>(this.GetEntity(TransformUsageFlags.Dynamic)).Initialize();
        }
    }
}
```

Reference the imported `MyGraphAuth` main object created by the importer, not the source `.mygraph` asset itself.

### Baking System

Each graph type needs a baking system so imported auth assets become runtime blob assets:

```csharp
using BovineLabs.Grove.Authoring;
using Unity.Entities;

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
public partial class MyGraphBakingSystem : GraphBakingSystem<MyGraph, MyGraphAuth>
{
    protected override string VersionKey => "my_graph_version";
}
```

Register the generic baking component in `AssemblyInfo.cs`:

```csharp
[assembly: RegisterGenericComponentType(typeof(GraphBakingData<MyGraphAuth>))]
```

### Runtime Execution

At runtime, create `GraphImpl<TContext>` once, get a `GraphExecution<TContext>` each update, and schedule a job that sets the current chunk before executing each entity's graph blob.

```csharp
using Unity.Burst;
using Unity.Burst.Intrinsics;
using Unity.Collections;
using Unity.Entities;

public partial struct MyGraphSystem : ISystem
{
    private GraphImpl<MyContext> impl;

    public void OnCreate(ref SystemState state)
    {
        this.impl = new GraphImpl<MyContext>(ref state);
    }

    public void OnUpdate(ref SystemState state)
    {
        var execution = this.impl.GetExecution(ref state);
        var query = SystemAPI.QueryBuilder().WithAll<MyGraph>().Build();

        state.Dependency = new ExecuteGraphJob
        {
            GraphExecution = execution,
            EntityHandle = SystemAPI.GetEntityTypeHandle(),
            MyGraphHandle = SystemAPI.GetComponentTypeHandle<MyGraph>(true),
        }.ScheduleParallel(query, state.Dependency);
    }

    [BurstCompile]
    private struct ExecuteGraphJob : IJobChunk
    {
        public GraphExecution<MyContext> GraphExecution;

        [ReadOnly]
        public EntityTypeHandle EntityHandle;

        [ReadOnly]
        public ComponentTypeHandle<MyGraph> MyGraphHandle;

        public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
        {
            this.GraphExecution.SetChunk(chunk, unfilteredChunkIndex);

            var entities = chunk.GetNativeArray(this.EntityHandle);
            var graphs = chunk.GetNativeArray(ref this.MyGraphHandle);

            for (var entityIndex = 0; entityIndex < chunk.Count; entityIndex++)
            {
                this.GraphExecution.Execute(entities[entityIndex], entityIndex, graphs[entityIndex].Graph);
            }
        }
    }
}
```

`GraphExecution<TContext>` also supports:

- `SetInput<TInput>()`
- `TrySetInput<TInput>()`
- `Execute<TInput>()`
- `Calculate<TOutput>()`
- `Calculate<TInput, TOutput>()`

## Built-In Graph Toolkit Nodes

The current Graph Toolkit authoring flow includes built-in Grove support for:

- Graph Toolkit constants and boundary variables
- Native Graph Toolkit subgraphs
- Custom Grove execution, data, context, and block nodes
- `CompositeNode`
- `GroveVariantNode`
- Selector nodes
- Qualifier nodes

## Debugging

Grove supports debug callbacks in editor and `BL_DEBUG` builds.

Add debug methods with:

- `[ExecuteNodeDebug(...)]`
- `[DataNodeDebug(...)]`

Runtime debug execution is gated by:

- `graph.debug-enabled`
- `graph.debug-selected`

If `graph.debug-selected` is enabled, Grove only runs debug methods for the selected entity.
