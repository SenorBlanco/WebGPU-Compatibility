# Problem

WebGPU is a good match for modern explicit graphics APIs such as Vulkan, Metal and D3D12. However, there are a large number of devices which do not yet support those APIs. In particular, on Chrome on Windows, 34% of Chrome users do not have D3D12. On Android, [23% of Android users do not have Vulkan 1.1 (15% do not have 1.0)](https://developer.android.com/about/dashboards). On ChromeOS, Vulkan penetration is still quite low, while OpenGL ES 3.1 is ubiquitous.

# Goals

The primary goal of WebGPU Compatibility mode is to increase the reach of WebGPU by providing an opt-in, slightly restricted subset of WebGPU which will run on older APIs such as D3D11 and OpenGL ES. This will increase adoption of WebGPU applications via a wider userbase.

Since WebGPU Compatibility mode is a subset of WebGPU, all valid Compatibility mode applications are also valid WebGPU applications. Consequently, Compatibility mode applications will also run on user agents which do not support Compatibility mode. Such user agents will simply ignore the option requesting a Compatibility mode Adapter and return a Core WebGPU Adapter instead.

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

# Compatiblity mode issues and restrictions

### 1. Texture view dimension must be specified 

When specifying a texture, a `viewDimension` property determines the views which can be created from that texture (see "Proposed IDL changes", above). Creating a view of a different dimension than specified at texture creation time will cause a validation error.

**Justification**: OpenGL ES does not support texture views.

**Alternatives considered:**
- make a view dimension guess at texture creation time, and perform a texture-to-texture copy at bind time if the guess was incorrect.
  - pros:
    - wider support of WebGPU content without modification
  - cons:
    - unexpected performance cliff for developers
    - potentially increased VRAM usage (two+ copies of texture data)
- disallow 6-layer 2D arrays (always cube maps)
  - cons:
    - poor compatibility, limits applications
- disallow cube maps (always create 6-layer 2D arrays)
  - cons:
    - poor compatibility, limits applications
- make a view dimension guess as above, but make the viewDimension property optional (a hint)
  - pros:
    - wider support of WebGPU content without modification
  - cons:
    - if developer doesn't provide the hint, it's still a performance cliff
    - potentially increased VRAM usage (two+ copies of texture data)

### 2. Emulate `copyTextureToBuffer()` of depth/stencil textures with a compute shader

**Justification**: OpenGL ES does not support `glReadPixels()` of depth/stencil textures.

**Alternatives considered**:
- use CPU readback and re-upload
  - pros:
    - wide support
  - cons:
    - large performance cliff
- disallow via validation
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility
- require [`GL_NV_read_depth_stencil`](https://registry.khronos.org/OpenGL/extensions/NV/NV_read_depth_stencil.txt)
  - pros:
    - good performance
  - cons:
    - poor support (<1% on gpuinfo.org)

### 3. Emulate `copyTextureToBuffer()` of SNORM textures with a compute shader

**Justification**: OpenGL ES does not support `glReadPixels()` of SNORM textures

**Alternatives considered**:
- disallow via validation
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility; limits applications

### 4. Disallow `CommandEncoder.copyTextureToBuffer()` for compressed texture formats

`CommandEncoder.copyTextureToBuffer()` of a compressed texture is disallowed, and will result in a validation error.

**Justification**: Compressed texture formats are non-renderable in OpenGL ES, and
glReadPixels() on works on a framebuffer-complete FBO.

**Alternatives considered**: 
- implement a shadow copy buffer, and upload the compressed data to both a buffer and a texture
  - pros:
    - good compatibility
  - cons:
    - performance overhead, even when readbacks are not required
    - VRAM overhead

### 5. Views of the same texture used in a single draw may not differ in mip level or array layer parameters.

A draw call may not reference the same texture with two views differing in `baseMipLevel`, `mipLevelCount`, `baseArrayLayer`, or `arrayLayerCount`. Only a single mip level range and array layer range per texture is supported. This is enforced via validation at encode time.

**Justification**: OpenGL ES does not support texture views.

**Alternatives considered**:

- when two bindings exist with different mip levels or array layers, do a texture-to-texture copy
  - pros:
    - good compatibility
  - cons:
    - a performance cliff for developers
    - higher VRAM usage

### 6. Emulate separate sampler and texture objects with a cache of combined texture/samplers.

**Justifcation**: OpenGL ES does not support separate sampler and texture objects.

**Alternatives considered**:

- allow only a single sampler to be used with a given texture
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility

### 7. Color state `alphaBlend`, `colorBlend` and `writeMask` may not differ between color attachments in a single draw.

Color state descriptors used in a single draw must have the same alphaBlend, colorBlend and writeMask, or else an encode-time validation error will occur.

**Justification**: OpenGL ES 3.1 does not support indexed draw buffer state.

**Alternatives considered**
- require [`GL_EXT_draw_buffers_indexed`](https://registry.khronos.org/OpenGL/extensions/EXT/EXT_draw_buffers_indexed.txt) 
  - pro: ease of implementation
  - con: poor reach: `GL_EXT_draw_buffers_indexed` has [limited support (~42%)](https://opengles.gpuinfo.org/listreports.php?extension=GL_EXT_draw_buffers_indexed)
- expose as a WebGPU extension when the OpenGL ES extension is present (this could be a followup change)
  - pros:
    - ease of implementation
    - good performance
  - cons:
    - if this is the only implementation, it has poor reach

### 8. Disallow `sample_mask` builtin in WGSL.

**Justification**: OpenGL ES 3.1 does not support `gl_SampleMask`, `gl_SampleMaskIn`.

**Alternatives considered**
- require [`GL_OES_sample_variables`](https://registry.khronos.org/OpenGL/extensions/OES/OES_sample_variables.txt) 
  - pro: ease of implementation
  - con: poor reach: `GL_OES_sample_variables` has [limited support (~48%)](https://opengles.gpuinfo.org/listreports.php?extension=GL_OES_sample_variables)
- expose as a WebGPU extension when the OpenGL ES extension is present (this could be a followup change)
  - pros:
    - ease of implementation
  - cons:
    - poor reach, unless this is built on top of the proposed solution

### 9. Inject hidden uniforms for `textureNumLevels()` and `textureNumSamples()` where required.

**Justification**: OpenGL ES 3.1 does not support textureQueryLevels() (only added to desktop GL in OpenGL 4.3).

**Alternatives Considered**:

- disallow `textureNumLevels()` and `textureNumSamples()` in WGSL via validation.
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility

### 10. Emulate 1D textures with 2D textures.

**Justification**: OpenGL ES does not support 1D textures.

**Alternatives Considered**:

- disallow 1D textures in WGSL and API
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility

### 11. `GPUTextureViewDimension.CubeArray` is unsupported

**Justification**: OpenGL ES does not support Cube Array textures.

**Alternatives Considered**:
- none

### 12. Disallow `textureLoad()` of depth textures in WGSL via validation.

**Justification**: OpenGL ES does not support `texelFetch()` of a depth texture.

**Alternatives considered**:
- bind to an RGBA8 binding point and use shader ALU
  - pros:
    - compatibility, performance
  - cons:
    - untried (does this work?)
- use `texture()` with quantized texture coordinates; massage the results
  - pros:
    - compatibility, performance
  - cons:
    - untried
    - complexity of implementation

### 13. Disallow `texture*()` of a `texture_depth_2d_array` with an offset

**Justification**: OpenGL ES does not support `textureOffset()` on a sampler2DArrayShadow.

**Alternatives considered**:

- emulate with a `texture()` call and use ALU for offset
  - pros:
    - compatibility, performance
  - cons:
    - untried

### 14. Manually pad out GLSL structs and interface blocks to support explicit `@offset`, `@align` or `@size` decorations.

**Justification**: OpenGL ES does not support offset= interface block decorations on anything but `atomic_uint`.

**Alternatives considered**:

- disallow `@offset`, `@align` and `@size` on WGSL structs via validation
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility

### 15. Emit `dpdx()` and `dpdy()` for all derivative functions (include Coarse and Fine variants).

**Justification**: GLSL does not support `dFd*Coarse()` or `dFd*Fine()` functions. However, these variants can be interpreted as a hint in WGSL, and emitted as `dFd*()`.

**Alternatives considered**:

- disallow `Coarse` and `Fine` variants via validation in WGSL
  - cons:
    - poor compatibility
- `Coarse` is allowed; `Fine` is disallowed via validation

### 16. Use [`GL_ext_texture_format_BGRA8888`](https://registry.khronos.org/OpenGL/extensions/EXT/EXT_texture_format_BGRA8888.txt) to support BGRA `copyBufferToTexture()` and swizzle workarounsd and RGBA textures where unavailable.

**Justification**: OpenGL ES does not support BGRA texture formats.

`GL_ext_texture_format_BGRA8888` supports texture uploads and the BGRA8888 texture format, and has [99%+ support](https://opengles.gpuinfo.org/listreports.php?extension=GL_EXT_texture_format_BGRA8888). The vast majority of devices which do not support it are GLES 3.0 implementations, and so would not support Compatibility mode anyway, but if an important device emerges, a CPU- or GPU-based swizzle workaround and RGBA textures should be implemented.

**Alternatives considered**
- disallow BGRA8888 as a texture format through validation
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility

### 17. Work around lack of BGRA support in copyTextureToBuffer() via compute or sampling.

**Justification**: OpenGL ES does not support BGRA texture formats for `glReadPixels()`, even with the `GL_ext_texture_format_BGRA8888` extension.

There is no corresponding gl

### 18. Disallow bgra8unorm-srgb textures.

**Justification**: OpenGL ES does not support BGRA texture formats.

**Alternatives considered**:
- use a compute shader to swizzle bgra8unorm-srgb to rgba8unorm-srgb on `copyBufferToTexture()` and the reverse on `copyTextureToBuffer()`
  - pros:
    - wide compatibility
  - cons:
    - a performance cliff for developers
    - increased VRAM usage

### 19. Use emulation workaround to support BaseVertex / BaseInstance in direct draws. Disallow via validation in indirect draws.

**Justification**: OpenGL ES 3.1 does not support `baseVertex` or `baseInstance` parameters in Draw calls.

**Alternatives considered**

- require `OES_draw_elements_base_vertex` (21% support) or `EXT_draw_elements_base_vertex` (21%) and `GL_EXT_base_instance` (1.7%)
  - pros:
    - ease of implementation
  - cons:
    - poor compatibility
