---
title: "Culling of custom objects in Bevy"
date: 2023-05-13
---

# Culling of custom components in Bevy

I've been doing some experimentation with volume rendering in Bevy and I wanted to make my own representation of the actual rendered objects. I still wanted to have the regular visibility culling stage of Bevy to be performed on my volumes so here's a brief walkthrough of how I enabled culling for my non-mesh components.

First off, this is how a volume instance is represented in the ECS. It's fairly similar to how meshes are setup. We have a reference to the actual volume asset, transformations, and the visibility components. As with using meshes in Bevy, I typically don't have to care about anything else than `volume` and `transform` when actually using this bundle.
```rs
pub struct VolumeBundle {
    pub volume: Handle<Volume>,
    pub transform: Transform,
    pub global_transform: GlobalTransform,
    pub visibility: Visibility,
    pub computed_visibility: ComputedVisibility,
}
```

Now to the systems I needed to implement. First, I had to add a system that fetches the bounds for the culling itself. I made a `calculate_bounds` system and made sure to have it run when Bevy expects it to run (`VisibilitySystems::CalculateBounds`).
```rs
app
    .add_system(
        calculate_bounds.in_set(VisibilitySystems::CalculateBounds),
    );
```

The sytem implementation itself basically just filters out my volume instance entities, fetches the AABB, and inserts it as a new component of the entity:
```rs
/// Calculate bounds for all entities without an existing AABB or tagged as with NoFrustumCulling
/// Volume in this case is just a voxel volume
fn calculate_bounds(
    mut commands: Commands,
    volumes: Res<Assets<Volume>>,
    without_aabb: Query<
        (Entity, &Handle<Volume>),
        (Without<Aabb>, Without<NoFrustumCulling>),
    >,
) {
    for (entity, volume_handle) in &without_aabb {
        if let Some(volume) = volumes.get(volume_handle) {
            // Get the AABB of the voxel volume and insert it as a component
            if let Some(aabb) = volume.aabb() {
                commands.entity(entity).insert(aabb);
            }
        }
    }
}

impl Volume {
    /// Get the AABB of this volume
    pub fn aabb(&self) -> Option<Aabb> {
        Some(Aabb::from_min_max(
            Vec3::new(0.0, 0.0, 0.0),
            Vec3::new(
                self.size.x as f32,
                self.size.y as f32,
                self.size.z as f32,
            ),
        ))
    }
}
```
Now culling of all the volume entities should be automatically performed by Bevy and we can use the results from the culling in other systems. For instance, during the extraction stage of the RenderApp you might want to only extract instances that are visibile:

```rs
/// Extracts (Entity, Handle<Volume>) into RenderApp for all visible volume instances.
fn extract_volume_instances(
    mut commands: Commands,
    mut prev_entities_len: Local<usize>,
    volumes_query: Extract<
        Query<(Entity, &ComputedVisibility, &Handle<Volume>)>,
    >,
) {
    let visible_volumes =
        volumes_query.iter().filter(|(_, vis, ..)| vis.is_visible());

    let mut entities = Vec::with_capacity(*prev_entities_len);
    for (entity, _, volume_handle) in visible_volumes {
        entities.push((entity, volume_handle.clone()));
    }
    *prev_entities_len = entities.len();
    commands.insert_or_spawn_batch(entities);
}
```

Another use case might be to use the culling results per view to queue only relevant volumes before rendering each view:

```rs
/// Queues volume render assets to view-specific queues. This would be a good place to
/// queue objects into RenderPhases as well but for my purpose I've set up a custom
/// VolumeQueue component. This assumes that this component has already been added to the
/// view (happens during the Extract stage).
/// 
/// 1. Takes all extracted Handle<Volume>, as extracted in `extract_volume_instances`,
/// 2. Checks their visibility in each view
/// 3. If visible, add the RenderAsset representation of the volume into the queue.
fn queue_volume_instances(
    render_volumes: Res<RenderAssets<Volume>>,
    volumes: Query<(&Handle<Volume>,)>,
    mut views: Query<(&ExtractedView, &VisibleEntities, &mut VolumeQueue)>,
) {
    for (view, visible_entities, mut queue) in &mut views {
        for visible_entity in &visible_entities.entities {
            if let Ok((volume_handle,)) = volumes.get(*visible_entity) {
                if let Some(volume) = render_volumes.get(volume_handle) {
                    queue.push(visible_entity);
                }
            }
        }
    }
}
```


