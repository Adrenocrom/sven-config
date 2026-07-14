---
name: wgpu_rust_graphics
description: Guide for using wgpu - a cross-platform, safe, pure-Rust graphics API
  based on WebGPU standard. Covers setup, core concepts, rendering workflow, shaders
  (WGSL), and common patterns.
tags:
- rust
- graphics
- wgpu
- webgpu
- shaders
- wgsl
- rendering
- gpu
created_at: '2026-07-13T20:40:18.638989+00:00'
---

# WGPU Graphics Programming in Rust

## Overview

`wgpu` is a cross-platform, safe, pure-Rust graphics API based on the WebGPU standard. It runs natively on Vulkan, Metal, D3D12, and OpenGL; and on top of WebGL2 and WebGPU on wasm. MSRV: **1.87**.

**Key characteristics:**
- API is refcounted (all handles are cloneable)
- Uses D3D/Metal coordinate systems (depth ranges from [0, 1])
- Supports WGSL shaders by default, plus SPIR-V and GLSL via features
- Based on WebGPU standard but fully native Rust

## Installation

Add to `Cargo.toml`:
```toml
[dependencies]
wgpu = "21"
# Optional: winit for windowing (native only)
winit = "0.30"
```

**Feature flags:**
- `wgsl` (enabled by default) - WGSL shader support
- `spirv` - SPIR-V shader support  
- `glsl` - GLSL shader support
- Backend features: `vulkan`, `metal`, `dx12`, `gles` (all enabled by default on respective platforms)

## Core Architecture

### Entry Points

The main API follows this hierarchy:

```
Instance → Adapter → Device + Surface
```

1. **Instance** - Entry point to interact with system GPUs
2. **Adapter** - Physical GPU/device handle  
3. **Device** - Open connection to graphics/compute device
4. **Surface** - Presentable surface (window, canvas)

### Basic Setup Pattern

```rust
use wgpu;

// 1. Create instance
let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
    backends: wgpu::Backends::all(), // or specific backends
    ..Default::default()
});

// 2. Request adapter (GPU)
let adapter = instance.request_adapter(
    &wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::HighPerformance,
        compatible_surface: None, // set if using a surface
        force_fallback_adapter: false,
    }
).await?;

// 3. Request device and queue
let (device, queue) = adapter.request_device(
    &wgpu::DeviceDescriptor {
        label: Some("Main Device"),
        required_features: wgpu::Features::empty(),
        required_limits: wgpu::Limits::default(),
        memory_hints: wgpu::MemoryHints::Performance,
    },
    None // trace path for debugging
).await?;

// 4. Configure surface (if using window)
let surface = instance.create_surface(window)?;
let surface_config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface.get_capabilities(&adapter).formats[0],
    width: window_size.width.max(1),
    height: window_size.height.max(1),
    present_mode: wgpu::PresentMode::Fifo, // VSync
    alpha_mode: wgpu::CompositeAlphaMode::Auto,
    view_formats: vec![],
    desired_maximum_frame_latency: 2,
};
surface.configure(&device, &surface_config);
```

## Rendering Workflow

### 1. Create Shader Module

WGSL (WebGPU Shading Language) is the primary shader language:

```rust
let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    label: Some("Main Shader"),
    source: wgpu::ShaderSource::Wgsl(include_wgsl!("shader.wgsl").into()),
});
```

**Example WGSL (`shader.wgsl`):**
```wgsl
@vertex
fn vs_main(@builtin(vertex_index) in_vertex: u32) -> @builtin(position) vec4<f32> {
    var positions = array<vec2<f32>, 3>(
        vec2<f32>(0.0, -0.5),
        vec2<f32>(0.5, 0.5),
        vec2<f32>(-0.5, 0.5)
    );
    
    return vec4<f32>(positions[in_vertex], 0.0, 1.0);
}

@fragment
fn fs_main() -> @location(0) vec4<f32> {
    return vec4<f32>(1.0, 0.0, 0.0, 1.0); // Red color
}
```

### 2. Create Pipeline Layout

Defines how shaders access resources (uniforms, textures):

```rust
let pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
    label: Some("Main Pipeline Layout"),
    bind_group_layouts: &[&bind_group_layout],
    push_constant_ranges: &[],
});
```

### 3. Create Render Pipeline

```rust
let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    label: Some("Main Render Pipeline"),
    layout: Some(&pipeline_layout),
    vertex: wgpu::VertexState {
        module: &shader,
        entry_point: "vs_main",
        buffers: &[vertex_buffer_layout],
        compilation_options: Default::default(),
        cache: None,
    },
    fragment: Some(wgpu::FragmentState {
        module: &shader,
        entry_point: "fs_main",
        targets: &[Some(wgpu::ColorTargetState {
            format: surface_config.format,
            blend: Some(wgpu::BlendState::REPLACE),
            write_mask: wgpu::ColorWrites::ALL,
        })],
        compilation_options: Default::default(),
    }),
    primitive: wgpu::PrimitiveState {
        topology: wgpu::PrimitiveTopology::TriangleList,
        strip_index_format: None,
        front_face: wgpu::FrontFace::Ccw,
        cull_mode: Some(wgpu::Face::Back),
        unclipped_depth: false,
        polygon_mode: wgpu::PolygonMode::Fill,
        force_multisample: false,
    },
    depth_stencil: None, // or Some for depth testing
    multisample: wgpu::MultisampleState {
        count: 1,
        mask: !0,
        alpha_to_coverage_enabled: false,
    },
    multiview: None,
    cache: None,
});
```

### 4. Render Loop

```rust
loop {
    // Get next surface texture
    let frame = surface.get_current_texture().expect("Failed to acquire next swap chain value");
    let view = frame.texture.create_view(&wgpu::TextureViewDescriptor::default());
    
    // Encode commands
    let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
        label: Some("Main Encoder"),
    });
    
    {
        let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            label: Some("Main Render Pass"),
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view: &view,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Clear(wgpu::Color {
                        r: 0.1, g: 0.2, b: 0.3, a: 1.0,
                    }),
                    store: wgpu::StoreOp::Store,
                },
            })],
            depth_stencil_attachment: None,
            timestamp_writes: None,
            occlusion_query_set: None,
        });
        
        render_pass.set_pipeline(&render_pipeline);
        render_pass.draw(0..3, 0..1); // Draw triangle (3 vertices)
    }
    
    queue.submit(std::iter::once(encoder.finish()));
    frame.present();
}
```

## Key Concepts

### Buffers

**Vertex Buffer:**
```rust
let vertex_buffer = device.create_buffer(&wgpu::BufferDescriptor {
    label: Some("Vertex Buffer"),
    size: (vertices.len() * std::mem::size_of::<Vertex>()) as wgpu::BufferAddress,
    usage: wgpu::BufferUsages::VERTEX | wgpu::BufferUsages::COPY_DST,
    mapped_at_creation: false,
});

queue.write_buffer(&vertex_buffer, 0, bytemuck::cast_slice(&vertices));
```

**Uniform Buffer (CBU):**
```rust
let uniform_buffer = device.create_buffer(&wgpu::BufferDescriptor {
    label: Some("Uniform Buffer"),
    size: std::mem::size_of::<Uniforms>() as wgpu::BufferAddress,
    usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
    mapped_at_creation: false,
});

queue.write_buffer(&uniform_buffer, 0, bytemuck::cast_slice(&uniforms));
```

### Textures

**Creating a texture:**
```rust
let texture = device.create_texture(&wgpu::TextureDescriptor {
    label: Some("Render Texture"),
    size: wgpu::Extent3d { width: 800, height: 600, depth_or_array_layers: 1 },
    mip_level_count: 1,
    sample_count: 1,
    dimension: wgpu::TextureDimension::D2,
    format: wgpu::TextureFormat::Bgra8UnormSrgb,
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT | wgpu::TextureUsages::TEXTURE_BINDING,
    view_formats: &[],
});

let texture_view = texture.create_view(&wgpu::TextureViewDescriptor::default());
```

### Bind Groups

Bind resources to pipeline layout slots:

```rust
let bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
    label: Some("Main Bind Group Layout"),
    entries: &[
        wgpu::BindGroupLayoutEntry {
            binding: 0,
            visibility: wgpu::ShaderStages::FRAGMENT,
            ty: wgpu::BindingType::Buffer {
                ty: wgpu::BufferBindingType::Uniform,
                has_dynamic_offset: false,
                min_binding_size: None,
            },
            count: None,
        },
    ],
});

let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    label: Some("Main Bind Group"),
    layout: &bind_group_layout,
    entries: &[
        wgpu::BindGroupEntry {
            binding: 0,
            resource: uniform_buffer.as_entire_binding(),
        },
    ],
});
```

## Compute Shaders

For GPU compute operations:

```rust
let compute_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
    label: Some("Compute Pipeline"),
    layout: Some(&compute_pipeline_layout),
    module: &compute_shader,
    entry_point: "main",
    compilation_options: Default::default(),
    cache: None,
});

let mut encoder = device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: Some("Compute Encoder"),
});

{
    let mut compute_pass = encoder.begin_compute_pass(&wgpu::ComputePassDescriptor {
        label: Some("Compute Pass"),
        timestamp_writes: None,
    });
    
    compute_pass.set_pipeline(&compute_pipeline);
    compute_pass.set_bind_group(0, &bind_group, &[]);
    compute_pass.dispatch_workgroups(workgroup_count_x, workgroup_count_y, workgroup_count_z);
}

queue.submit(std::iter::once(encoder.finish()));
```

## Important Constants and Limits

- `COPY_BUFFER_ALIGNMENT` - Buffer copy alignment requirement
- `VERTEX_ALIGNMENT` - Vertex buffer offset/stride alignment  
- `MAP_ALIGNMENT` - Minimum buffer mapping alignment
- `QUERY_SIZE` - Size of query data in bytes

## Debugging Tips

1. **Enable validation layers** (default on debug builds)
2. **Use error scopes:**
   ```rust
   device.push_error_scope(wgpu::ErrorFilter::Validation);
   // ... do GPU work ...
   let err = queue.poll(wgpu::Maintain::Wait).await;
   if let Some(err) = err.into_errors().into_iter().next() {
       eprintln!("GPU error: {:?}", err);
   }
   device.pop_error_scope();
   ```
3. **Trace API calls** for debugging:
   ```rust
   let (device, queue) = adapter.request_device(
       &wgpu::DeviceDescriptor { ... },
       Some("trace/path") // enables tracing
   ).await?;
   ```

## Resources

- **Official Docs:** https://docs.rs/wgpu/latest/wgpu/
- **Learn WGPU Tutorial:** https://sotrh.github.io/learn-wgpu/
- **WebGPU Fundamentals:** https://webgpufundamentals.org/
- **Examples Repository:** https://github.com/gfx-rs/wgpu/tree/v30/examples
- **WGSL Spec:** https://gpuweb.github.io/gpuweb/wgsl/

## Common Pitfalls

1. **Resource lifetime** - wgpu uses refcounting, but ensure resources aren't dropped before GPU finishes using them
2. **Buffer alignment** - Respect `COPY_BUFFER_ALIGNMENT` and `VERTEX_ALIGNMENT`
3. **Texture format** - Use `*Srgb` formats for sRGB color spaces (wgpu applies transfer function automatically)
4. **Present modes** - `Fifo` = VSync, `Mailbox` = reduced latency, `Immediate` = no sync
5. **Shader compilation errors** - Check `CompilationInfo` after creating shader modules

