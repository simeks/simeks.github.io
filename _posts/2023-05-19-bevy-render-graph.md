---
title: "Deferred shading with compute shaders in Bevy"
date: 2023-05-19
excerpt_separator: <!--more-->
---

# Deferred shading with compute shaders in Bevy

![Lights](/images/2023-05-19-screenshot.png){:width="75%" height="75%"}

I've been experimenting with my ray casting voxel renderer in Bevy and one of the things I've been looking into is a deferred approach for lighting the scene. This post just briefly goes through some background and the steps I took to implement this in Bevy v0.10.1, and it mostly focues on the Bevy render graph. Disclaimer: Since I've opted for running all my rendering in compute shaders, a lot of the decisions below are due to Bevy being designed with the traditional rendering flow in mind (and rightfully so).

<!--more-->

For the rendering, Bevy sets up the rendering flow through a typical render graph. Meaning you build the rendering flow by connecting various nodes. The graph then ensures they are performed in correct order. 

![Bevy pipeline](/images/2023-05-19-bevy-pipeline.svg)

The figure above illustrates the default render graph for 3D in Bevy. The `Prepass` is a node performing a prepass, this node can for instance run a depth prepass of the view. The `MainPass3d` node sets up render passes for the three core render phases: `Opaque3d`, `AlphaMask3d`, and `Transparent3d`, dispatching any queued up draw calls. This is where the main render logic ends up. For instance, Bevy provides clustered forward rendering through the `bevy_pbr` plugin. This plugin is responsible for setting up necessary shader pipelines and ensuring draw calls and necessary resources are queued up each frame.

I won't go into much detail about `bevy_pbr` simply because I've decided to opt-out from that plugin. The reason being that `bevy_pbr` is designed around the traditional render passes and I wasn't able to fit my ray caster into that framework. I decided to scratch the `MainPass3d` node in my render graph for the same reason, since it's built to run render passes, and not compute. 

As mentioned, Bevy implements clustered forward rendering, which basically means that you render your geometry and calculate your lighting in a single pass. Deferred rendering on the other hand splits the rendering into two phases; first you render the geometry, then you calculate the lighting.  There are a lot of pros and cons for both approaches that I won't go into here. The reason I opted for deferred shading is that I only need to do the scene rendering once, which is fairly expensive, but I'll still be quite flexible with what I can do in later passes.

![My pipeline](/images/2023-05-19-my-pipeline.svg)

After creating my own graph I ended up with something as illustrated in the image above. I decided to not scrap the whole default pipeline, since it would be nice to be able to use any post-processing nodes provided in Bevy. However, I removed `Prepass` and `MainPass3d` replacing them with two new nodes; `Ray casting` and `Lighting`. The first node is my compute shader that renders my scene using ray casting and stores the results into my gbuffer, which for now consists of albedo, normals, and depth. The second node, which is still quite simple, applies lighting on my scene. The result of the lighting then goes through the postprocessing steps just as usual.

Now for the actual implementation. I decided to setup a completely new graph with all the nodes I needed. In Bevy 0.10 this is quite a verbose procedure, but they are improving this in 0.11:

```rust
let render_app = app.sub_app_mut(RenderApp);

// Create all the render nodes needed
let raycast_node = RaycastNode::new(&mut render_app.world);
let light_node = LightNode::new(&mut render_app.world);
let blit_node = BlitNode::new(&mut render_app.world);
let tonemapping = TonemappingNode::new(&mut render_app.world);
let upscaling = UpscalingNode::new(&mut render_app.world);

let mut graph = render_app.world.resource_mut::<RenderGraph>();

// Create our new render graph as a sub-graph and add all nodes
let mut sub_graph = RenderGraph::default();
sub_graph.add_node(RAYCAST_NODE, raycast_node);
sub_graph.add_node(LIGHT_NODE, light_node);
sub_graph.add_node(BLIT_NODE, blit_node);
sub_graph.add_node(graph::node::TONEMAPPING, tonemapping);
sub_graph.add_node(graph::node::UPSCALING, upscaling);

// Setup a input slot for the graph, this lets us access the camera
// entity within the nodes. Things like render targets are saved as
// components to the camera entity so we need this to access them.
let input_node_id = sub_graph.set_input(vec![SlotInfo::new(
    graph::input::VIEW_ENTITY,
    SlotType::Entity,
)]);

// Connect our input node (the camera entity) to all new graph nodes
sub_graph.add_slot_edge(
    input_node_id,
    graph::input::VIEW_ENTITY,
    RAYCAST_NODE,
    RaycastNode::IN_VIEW,
);
sub_graph.add_slot_edge(
    input_node_id,
    graph::input::VIEW_ENTITY,
    LIGHT_NODE,
    LightNode::IN_VIEW,
);
sub_graph.add_slot_edge(
    input_node_id,
    graph::input::VIEW_ENTITY,
    BLIT_NODE,
    BlitNode::IN_VIEW,
);
sub_graph.add_slot_edge(
    input_node_id,
    graph::input::VIEW_ENTITY,
    graph::node::TONEMAPPING,
    TonemappingNode::IN_VIEW,
);
sub_graph.add_slot_edge(
    input_node_id,
    graph::input::VIEW_ENTITY,
    graph::node::UPSCALING,
    UpscalingNode::IN_VIEW,
);

// Connect the graph nodes, basically setting the order they shall execute.
sub_graph.add_node_edge(RAYCAST_NODE, LIGHT_NODE);
sub_graph.add_node_edge(LIGHT_NODE, BLIT_NODE);
sub_graph.add_node_edge(BLIT_NODE, graph::node::TONEMAPPING);
sub_graph
    .add_node_edge(graph::node::TONEMAPPING, graph::node::UPSCALING);

// Add our subgroup under the name `GRAPH_NAME`.
graph.add_sub_graph(GRAPH_NAME, sub_graph);
```

In the above graph we now have 5 nodes: `Raycast`, `Light`, `Blit`, `Tonemapping`, and `Upscaling`. The last two are provided by Bevy. As for the others, `Raycast` is a compute shader node that performs raycasting of the scene to fill the GBuffer which `Light` (also a compute shader) then consumes to apply lighting. The `Blit` node is a bit of a hack at the moment just to copy the output from the `Light` node to a format that `Tonemapping` accepts, so this might change.

To not make this too lengthy, I won't go through all the nodes in detail, but a good start for setting up your own Nodes, and especially compute nodes, is to look at the [game of life example](https://github.com/bevyengine/bevy/blob/v0.10.1/examples/shader/compute_shader_game_of_life.rs) provided with Bevy.

Something that example didn't mention however, is if you want to setup resources per camera and not as global resources. This is also the reason we decided to define the view entity as an input when we built the graph. Now, using that input slot is a bit finicky since we just get an entity ID as input, so we need to set up a query ourself to get the components we need. Here's a rough outline of the things needed for my raycast node:

```rust

/// Define and setup routine for initializing the pipeline, just as the game of life 
/// example
#[derive(Resource)]
struct RaycastPipeline {
    ...
}

impl FromWorld for RaycastPipeline {
    fn from_world(world: &mut World) -> Self {
        ...
    }
}

/// A basic gbuffer component that we attach to the camera
#[derive(Component)]
pub(crate) struct GBuffer {
    pub(crate) albedo: TextureView,
    ..
}

/// System preparing the gbuffer, run in RenderApp during the prepare stage
fn prepare_gbuffer(
    mut commands: Commands,
    mut texture_cache: ResMut<TextureCache>,
    render_device: Res<RenderDevice>,
    cameras: Query<(Entity, &ExtractedCamera)>,
) {
    for (entity, camera) in cameras.iter() {
        // Use texture cache to setup our gbuffer targets and insert the component
        let albedo = texture_cache
            .get(
                &render_device,
                TextureDescriptor {
                    ...
                },
            )
            .default_view;
        ...
    
        // The GBuffer will now be availabe from our render nodes
        commands.entity(entity).insert(GBuffer {
            albedo,
            ...
        });
    }
}

/// Setup the compute node
pub(crate) struct RaycastNode {
    /// Here we also declare the query we want to perform against world with our input
    /// entity, so this is basically the components we want to use within the node.
    query: QueryState<(
        &'static ExtractedCamera,
        &'static GBuffer,
        &'static ViewUniformOffset,
    )>,
}

impl RaycastNode {
    /// Name of our input slow
    pub const IN_VIEW: &'static str = "view";

    pub fn new(world: &mut World) -> Self {
        Self {
            // https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.query_filtered
            query: world.query_filtered(),
        }
    }
}

/// Implement methods required by the Node trait
impl Node for RaycastNode {
    /// Specifies the required input slots
    fn input(&self) -> Vec<SlotInfo> {
        vec![SlotInfo::new(Self::IN_VIEW, SlotType::Entity)]
    }

    /// We need to manualy update the query each frame
    fn update(&mut self, world: &mut World) {
        self.query.update_archetypes(world);
    }

    fn run(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext,
        world: &World,
    ) -> Result<(), NodeRunError> {
        // First thing we do is fetch the view components w need:
        
        let view_entity = graph.get_input_entity(Self::IN_VIEW)?;

        let (camera, gbuffer, view_uniform_offset) =
            match self.query.get_manual(world, view_entity) {
                Ok(result) => result,
                Err(_) => return Ok(()),
            };

        // Now we can proceed by setting bind groups and dispatching the compute kernel
        ...
    }
}
```

When our graph and all nodes are setup you need to setup your camera to use the new render graph as well. By default the `Camera3dBundle` uses the default 3D graph (first figure). To make it use our new render graph instead you can set this when spawning the component:

```rust
    commands
        .spawn(Camera3dBundle {
            camera_render_graph: CameraRenderGraph::new(GRAPH_NAME),
            ..Default::default()
        });
```

Now, this is maybe not the ideal solution and definitely subject for change, but given that Bevy is currently updating a lot of backbone for this I'm postponing any changes until Bevy 0.11 is released.
