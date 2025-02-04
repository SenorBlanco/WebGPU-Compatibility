WebGPU Compatibility Mode is an opt-in, lightly restricted subset of WebGPU capable of running older graphics APIs such as OpenGL and Direct3D11. The goal is to expand the reach of WebGPU applications to older devices that do not have the modern, explicit graphics APIs (e.g., Vulkan, Metal and Direct3D12) that core WebGPU requires. This requires Compatibility Mode to prohibit the use of some WebGPU features that the older APIs do not support.

In order to permit a Web developer to opt in to the Compatibility Mode subset, the WebGPU community group has added the featureLevel attribute to GPURequestAdapterOptions. This attribute supports the strings `"core"` and `"compatibility"`. To use it, developers set:

```
options.featureLevel = "compatibility"
```

And call `navigator.gpu.requestAdapter(options)`.


Other than support for the `featureLevel` attribute, the main web-facing changes required for a user agent to support Compatibility Mode are validation of the mode’s restrictions. If a web developer attempts to use a feature not supported by Compatibility Mode, a validation error will be generated (as is done currently for other invalid WebGPU content).

Some user agents, such as Safari, may choose not to implement Compatibility Mode, since their target devices all support a modern graphics API. For this reason, WebGPU Compatibility Mode applications are valid WebGPU applications, and will run unmodified on a WebGPU Core-only user agent. Such a user agent will not perform the validation against compatibility mode restrictions, but it will run the Compatibility application the same as any Core WebGPU app. Such a user agent would return a `GPUAdapter` with a Feature indicating that it is actually a Core-level adapter.

