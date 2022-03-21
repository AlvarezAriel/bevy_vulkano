# bevy_vulkano

[![Crates.io](https://img.shields.io/crates/v/bevy_vulkano.svg)](https://crates.io/crates/bevy_vulkano)
![Apache](https://img.shields.io/badge/license-Apache-blue.svg)
![CI](https://github.com/hakolao/bevy_vulkano/workflows/CI/badge.svg)

This plugin replaces core loop & rendering in [Bevy](https://github.com/bevyengine/bevy) with [Vulkano](https://github.com/vulkano-rs/vulkano) backend.
Basically this allows you to be fully in control of your render pipelines with Vulkano without having to bother yourself with engine
architecture much. Just roll your pipelines and have fun.

This makes it extremely easy to do following with Vulkano:
- Windowless Apps
- Multiple Windows
- Event handling

The plugin contains functionality for resizing, multiple windows & utility for beginning and ending the frame.
However, you'll need to do everything in between yourself. A good way to get started is to look at the examples.

1. Add `VulkanoWinitPlugin`. It also adds `WindowPlugin` and anything that's needed.
2. Then create your own rendering systems using vulkano's pipelines (See example.). You'll need to know how to use [Vulkano](https://github.com/vulkano-rs/vulkano).
3. If you want to use [egui](https://github.com/emilk/egui) library with this, add `egui` and `bevy_vulkano` with feature `gui`.

## Usage

```rust
pub struct PluginBundle;

impl PluginGroup for PluginBundle {
    fn build(&mut self, group: &mut PluginGroupBuilder) {
        group.add(bevy::input::InputPlugin::default());
        group.add(VulkanoWinitPlugin::default());
    }
}

fn main() {
    App::new()
        // Vulkano configs (Modify this if you want to add features to vulkano (vulkan backend).
        // You can also disable primary window opening here
        .insert_resource(VulkanoWinitConfig::default())
        // Window configs for primary window
        .insert_resource(WindowDescriptor {
            width: 1920.0,
            height: 1080.0,
            title: "Bevy Vulkano".to_string(),
            vsync: false,
            resizable: true,
            mode: WindowMode::Windowed,
            ..WindowDescriptor::default()
        })
        .add_plugins(PluginBundle)
        .run();
}
```

### Creating a pipeline

```rust
/// Creates a render pipeline. Add this system with app.add_startup_system(create_pipelines).
fn create_pipelines_system(mut commands: Commands, vulkano_windows: Res<VulkanoWindows>) {
    let primary_window = vulkano_windows.get_primary_window_renderer().unwrap();
    // Create your render pass & pipelines (MyRenderPass could contain your pipelines, e.g. draw_circle)
    let my_pipeline = YourPipeline::new(
        primary_window.graphics_queue(),
        primary_window.swapchain_format(),
    );
    // Insert as a resource
    commands.insert_resource(my_pipeline);
}
```

### Rendering system

```rust

/// This system should be added either at `CoreStage::PostUpdate` or `CoreStage::Last`. You could also create your own
/// render stage and place it after `CoreStage::Update`.
fn my_pipeline_render_system(
    mut vulkano_windows: ResMut<VulkanoWindows>,
    mut pipeline: ResMut<YourPipeline>,
) {
    let primary_window = vulkano_windows.get_primary_window_renderer_mut().unwrap();
    // Start frame
    let before = match primary_window.start_frame() {
        Err(e) => {
            bevy::log::error!("Failed to start frame: {}", e);
            return;
        }
        Ok(f) => f,
    };

    // Access the swapchain image directly
    let final_image = primary_window.final_image();
    // Draw your pipeline
    let after_your_pipeline = pipeline.draw(final_image);
    
    // Finish Frame by passing your last future
    primary_window.finish_frame(after_your_pipeline);
}
```

### Separating render systems

To allow parallel render systems for your pipelines: Splitting your rendering to smaller systems, you could create
`pre_render_setup_system` & `post_render_setup_system`. Which can update the futures pre and post render.
`PipelineSyncData` would be used to update `after` and `before` futures during the frame.

Your render system in between should update the `after` future.

```rust
/// Starts frame, updates before pipeline future & final image view
pub fn pre_render_setup_system(
    mut vulkano_windows: ResMut<VulkanoWindows>,
    mut pipeline_frame_data: ResMut<PipelineSyncData>,
) {
    let vulkano_renderer = vulkano_windows
        .get_primary_window_renderer_mut()
        .unwrap();
    let sync_data = pipeline_frame_data.get_primary_mut().unwrap();
    let before = match vulkano_renderer.start_frame() {
        Err(e) => {
            bevy::log::error!("Failed to start frame: {}", e);
            None
        }
        Ok(f) => Some(f),
    };
    sync_data.before = before;
}

/// If rendering was successful, draw gui & finish frame
pub fn post_render_system(
    mut vulkano_windows: ResMut<VulkanoWindows>,
    mut pipeline_frame_data: ResMut<PipelineSyncData>,
) {
    let vulkano_window = vulkano_windows
        .get_primary_window_renderer_mut()
        .unwrap();
    let sync_data = pipeline_frame_data.get_primary_mut().unwrap();
    if let Some(after) = sync_data.after.take() {
        let final_image_view = vulkano_window.final_image();
        let at_end_future = vulkano_window.gui().draw_on_image(after, final_image_view);
        vulkano_window.finish_frame(at_end_future);
    }
}
```

## Dependencies

This library re-exports `vulkano` and `egui_winit_vulkano`.

Add following to your `Cargo.toml`:
```toml
[dependencies.bevy]
version = "0.6"
default-features = false
# Add features you need, but don't add "render". This might disable a lot of features you wanted... e.g SpritePlugin
features = []

[dependencies.bevy_vulkano]
version = "0.1.0"
default-features = false
# gui or no gui...
features = ["gui"]
```

## Examples:
```bash
cargo run --example circle --features example_has_gui
cargo run --example circle
cargo run --example multi_window_gui --features example_has_gui
cargo run --example windowless_compute
cargo run --example game_of_life
```

### Contributing

Feel free to open a PR to improve or fix anything that you see would be useful.