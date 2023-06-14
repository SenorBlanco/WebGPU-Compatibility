# Problem

WebGPU is a good match for modern explicit graphics APIs such as Vulkan, Metal and D3D12. However, there are a large number of devices which do not yet support those APIs. In particular, on Windows the support for D3D12 is at 59%, while D3D11 gives 77% reach. On Android, [23% of Android users do not have Vulkan 1.1 (15% do not have 1.0)](https://developer.android.com/about/dashboards) On ChromeOS, Vulkan penetration is still quite low, while OpenGL ES 3.1 is ubiquitous.

# Goals

The primary goal of WebGPU/Compat is to increase the reach of WebGPU by providing an opt-in, slightly restricted subset of WebGPU which will run on older APIs such as D3D11 and OpenGL ES. This will increase adoption of WebGPU applications via a wider userbase.

# Proposed IDL changes

```
dictionary GPURequestAdapterOptions {
    ...
    boolean compatibilityMode = false;
}

```

When calling `GPU.RequestAdapter()`, passing `compatibilityMode = true` in the `GPURequestAdapterOptions` will indicate to the User Agent to select the Compatibilty subset of WebGPU. Any Devices created from the resulting Adapter will support only Compatibility mode. Calls to APIs unsupported by Compatibility mode will result in validation errors.

Note that a User Agent may return a `compatibilityMode = true` Adapter which is backed by a fully WebGPU-capable hardware adapter, such as D3D12, Metal or Vulkan, so long as it validates all subsequent API calls made on the Adapter and the objects it vends against the Compatiblity subset.

```
interface GPUAdapter {
    ...
    readonly attribute boolean isCompatibilityMode;
}
```

As a convenience to the developer, the Adapter returned will have the `isCompatibilityMode` property set to `true`.


```
dictionary GPUTextureDescriptor : GPUObjectDescriptorBase {
    ...
    GPUTextureViewDimension viewDimension;
}
```

When specifying a texture in a GPUTextureDescriptor, a viewDimension property determines the views which can be created from that texture. Creating a view of a different dimension than specified at texture creation time will cause a validation error.

# Compatiblity mode restrictions

### 1. Texture view dimension must be specified 

When specifying a texture, a viewDimension property determines the views which can be created from that texture. Creating a view of a differernt dimension than specified at texture creation time will cause a validation error.

**Justification**: OpenGL ES does not support texture views.

**Alternatives considered:**
- make a view dimension guess at texture creation time, and perform a texture-to-texture copy at bind time if the guess was incorrect.
- make a view dimension guess as above, but also provide the viewDimension property as an (optional) hint
- disallow 6-layer 2D textures (always cube maps)

### 2. Emulate copyTextureToBuffer() of depth/stencil textures with a compute shader

**Justification**: OpenGL ES does not support glReadPixels() of depth/stencil textures.

**Alternatives considered**:
- use CPU readback and re-upload
- disallow via validation
- use NV_read_stencil (<1% support)

### 3. Emulate copyTextureToBuffer() of SNORM textures with a compute shader

**Justification**: OpenGL ES does not support glReadPixels() of SNORM textures

**Alternatives considered**:
- disallow via validation

### 4. disallow `CommandEncoder.copyTextureToBuffer()` for compressed texture formats

`CommandEncoder.copyTextureToBuffer()` of a compressed texture is disallowed, and will result in a validation error.

**Justification**: Compressed texture formats are non-renderable in OpenGL ES, and
glReadPixels() on works on a framebuffer-complete FBO.

**Alternatives considered**: 
- implement a shadow copy buffer, and upload both the compressed and uncompressed data

### 5. Emulate separate sampler and texture objects with a cache of combined texture/samplers.

**Justifcation**: OpenGL ES does not support separate sampler and texture objects.

**Alternatives considered**:

- allow only a single sampler to be used with a given texture

### 6. views of the same texture used in a single draw may not differ in mip level or array layer 

A draw call may not reference the same texture with two views differing in mip level. Only a single mip level per texture is supported. This is enforced via validation at encode time.

**Justification**: OpenGL ES does not support texture views.

### 7. color state alphaBlend, colorBlend and writeMask may not differ in a single draw.

Color state descriptors used in a single draw must have the same alphaBlend, colorBlend and writeMask, or else an encode-time validation error will occur.

**Justification**: OpenGL ES does not support indexed draw buffer state until OpenGL ES 2.0, and GL_EXT_draw_buffers_indexed has limited support

### 9. `GPUTextureViewDimension` `CubeArray` is unsupported

**Justification**: OpenGL ES does not support Cube Array textures
