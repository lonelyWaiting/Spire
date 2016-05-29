# Spire
Spire is a shader compiler that generates optimized shader code that spans the entire pre-baking and rendering pipeline. A shader written in Spire exposes the optimization space of compute rate assignments and algorithm alternatives, and the compiler generates the lower level shader code (e.g. GLSL) that implements the desired optimization choice for all stages of your engine's renderer.  Spire is designed to facilitate rapid exploration of optimization choices and code generation for various platforms and purposes from a single shader definition.
## Using Spire
The Spire compiler is distributed as a single-source-file C++ library under "LibraryRelease" directory with no external dependencies. To integrate the compiler into your engine, simply grab "Spire.h" "Spire.cpp" and place them into your project.
## Getting Started
To experience Spire, compile `"Source/Spire.sln"` and run the `SceneViewer` project. In `SceneViewer`, select "File"->"Open", and open "TestShaders/couch/couch.world". This loads up a couch scene. The couch is rendered using "TestShaders/couch/couch.shader".
## Understanding Spire
The following is a Spire shader that computes the albedo color by blending two textures.
```
using "../EnginePipeline.shader";

shader AlbedoMappedSurface
{
  @mesh vec3 vert_pos;
  @mesh vec3 vert_normal;
  @mesh vec3 vert_tangent;
  @mesh vec2 vert_uv; 
  @viewUniform mat4 viewProjectionMatrix;
  @viewUniform vec3 lightDir;
  @perInstanceUniform sampler2D baseMap;
  @perInstanceUniform sampler2D baseMap2;
	
	vec4 projPos = viewProjectionMatrix * vec4(vert_pos, 1.0);
  vec2 uvCoord = vert_uv * 3.0;
  vec3 Normal = vert_normal;
  vec3 Albedo = texture(baseMap, uvCoord).xyz 
	              * texture(baseMap2, uvCoord  * 1.2).xyz;
	
  float lighting = max(dot(Normal, lightDir), 0.0);
  vec4 outputColor = vec4(Albedo * lighting, 1.0);
}
```
Here, `Lighting` is a predefined shader modules in `"EnginePipeline.shader"` that computes lighting effects. 
### What does `@mesh` etc. mean?
This is a rate specifier telling the compiler that `vert_pos` is defined at the `mesh` world, which corresponds to the vertex attributes. Similarly, the component `viewProjectionMatrix` is defined at `viewUniforms`, which hold all uniform shader inputs per view. `mesh`,`viewUniform` and `perInstnaceUniform` are defined by the pipeline in `"EnginePipeline.shader"`.

### Where did you define the output of this shader?
The pipeline definition provided in `EnginePipeline.shader` tells the compiler that `outputColor` is the final fragment output and `projPos` is the projected vertex output from. Here is the definition:
```
pipeline EnginePipeline
{
  world vs;
  world fs;
  ...
  require @vs vec4 projPos;
  require out @fs vec4 outputColor;
  ...
}
```
The `require` keyword specifies that all shaders need to provide a `outputColor` component, and `out` specifies that this component is an pipeline output.
### Does the compiler know the meaning of `vs` `fs` and `mesh` etc.?
The compiler does not know what `vs` means, but it will generate a lower level (GLSL) shader code named "vs" containing all components placed in `vs` world. Since you as the engine writer defines the pipeline, you know `vs` means vertex shader, so you can use the generated "vs" code as a vertex shader.
### How does the compiler figure out what to compute at each pipeline stage?
The compiler applies a seris of rules to infer the rate of each component. In this example, since `EnginePipeline` requires `projPos` to be at `vs`, the compiler knows to put it in the `vs` world. Similarly it knows that `outputColor` needs to be computed at `fs` world. The compiler resolves the world placement of the rest of components using a default policy that computes components at latest pipeline stage.
If the default behavior is not desired, you can always manually specify the rates, such as
```
@vs vec3 Albedo = texture(baseMap, uvCoord).xyz 
	              * texture(baseMap2, uvCoord  * 1.2).xyz;
```
This forces `Albedo` to be computed at `vs` world. One goal of Spire is to make writing mult-rate shaders easy. Imagine a pipeline that features prebaking, screen-space half resolution rendering and per-pixel rendering passes, the shading of an asset may contain logic that spans all these stages. Spire allows you to define all these logic in one place, and you can generate very different shaders that computes the shading logic at different rates by simply changing the rate specifier. Instead of knowing only a fixed set of stages such as vertex and fragment shaders, Spire lets the user to define the pipeline and provide a code generation service that takes care of stage dependency and proper interface generation.
