
# Migrating Shaders from MojoShader to ShaderConductor

This only applies to OpenGL platforms.<br>
You don't neccessarily have to migrate existing shaders, as MojoShader is still included. If all shaders in a file are shader model 2 or 3 MojoShader will automatically be used for shader compilation. This should ensure full backwards compatibility for existing projects.
You can mix MojoShader and ShaderConductor shaders in the same project, only ShaderConductor can handle shader model 4 or 5 though.
<br>

The moment a single shader in the file is set to shader model 4 or higher, the entire file will be compiled through ShaderConductor. So if you mix SM 3 and SM 4 shaders in the same file, even the SM 3 shaders will be compiled through ShaderConductor. It is recommended to switch all shaders in the file to SM 4 or higher simultaneously, because SM 2 and 3 support is not that great with ShaderConductor (see SM 2 and 3 limitations below). 
<br>

ShaderConductor compilation does not require Wine on Linux or Mac, so forcing ShaderConductor compilation for pure SM 2 and 3 files can still be useful.
You can force ShaderConductor compilation by adding the <b>CONDUCTOR</b> define to the effect processor by: 

- using the MGCB editor UI: Effect Properties -> Processor Parameters -> Defines
- adding <b>/processorParam:Defines=MOJO</b> to the mgcb file in the corresponding effect block
- adding the <b>/Defines:MOJO</b> argument when calling MGFXC directly via command line
<br>

In order to compile using ShaderConductor the following modifications to the HLSL source are neccessary, mainly because the new HLSL compiler from Microsoft (DXC) does not support the old DX9-style syntax anymore. 
<br>
<br>

## SV prefix for shader semantics
Pixel shaders in MojoShader used the COLOR and DEPTH output semantic, now it's SV_TARGET and SV_DEPTH.
In order to pass vertex positions from the vertex to the pixel shader, the POSITION semantic was used in the past, this needs to be SV_POSITION now.
```HLSL
float4 MyVertexShader(float4 inputPos: POSITION) : SV_POSITION
{ ... }

float4 MyPixelShader() : SV_TARGET
{ ... }
```
<br>
<br>

## Convert DX9 texture sampling to DX10
This is what texture sampling might look like in DX9.

```C#
texure MyTexture;
    
sampler2D MySampler = sampler_state 
{
    Texture = (MyTexture);
};

float4 MyPixelShader(VertexOut input) : COLOR
{
    return tex2D(MySampler, input.texCoord);
}
```

The equivalent DX10 code will work with ShaderConductor.   
```HLSL
Texure2D MyTexture;
SamplerState MySampler;

float4 MyPixelShader(VertexOut input) : SV_TARGET
{
    return MyTexture.Sample(MySampler, input.texCoord);
}
```
Note the three things that changed:
- **texture** becomes **Texture2D**. **Texture1D**, **Texture3D** and **TextureCube** are also possible.
- **sampler2D** becomes just **sampler** without the dimension, same for **sampler1D**, **sampler3D** and **samperCUBE**
- **tex2D()** becomes **MyTexture.Sample()**, same for **tex1D()**, **tex3D()** and **texCUBE()**. There are also DX10 equivalents for the more specialized texture sampling functions like **tex2Dlod()** which becomes **MyTexture.SampleLevel(MySampler, TexCoord, Level)**. More details about the available sampling functions can be found [here](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-type).
<br>
<br>

## Binding textures and samplers by register
The preferred method should generally be to bind by name whenever possible. MojoShader messes up the names for texture effect parameters, this is not the case anymore with ShaderConductor. You should always be able to bind textures like this.
```C#
// in HLSL
Texture2D MyTexture;

// in C#
effect.Parameters["MyTexture"].SetValue(myTexture);
```
Unfortunately there are no effect parameters for samplers currently. If you want to assign a sampler state in C#, you have to bind by register. You should never rely on automatic register assignment in this case, even if there is only a single sampler in your shader. If you want to guarantee your sampler is bound to a specific register, you need to specify the register explicitly in HLSL.
```HLSL
// in HLSL
SamplerState MySampler : register(s0);

// in C#
GraphicsDevice.SamplerStates[0] = SamplerState.PointClamp;
```
<br>
<br>

## Bool parameters get converted to int
Boolean shader parameters are represented by integers in GLSL. As a consequence the parameter type in the effect's parameter collection will be EffectParameterType.Int32. In DirectX that same parameter will be of type EffectParameterType.Bool. Generally you won't notice this difference, because the standard pattern for setting bool parameters still works, even though under the hood it's an int:
```C#
effect.Parameters["EnableLighting"].SetValue(true);
```
You will notice the difference if you reflect on the parameter type though. A shader editor might do such a thing in order to create a checkbox for booleans, and a value field for integers. Hopefully this limitation can be resolved in the future.  

<br><br><br>
<hr>

## Shader model 2 and 3 limitations
Some extra limitations apply when shader model 2 or 3 is used. This is necessary for supporting OpenGL 2. OpenGL 2 is considered a legacy target now. You should always use shader models 4 or higher if you can afford it (vs_4_0, ps_4_0). 
<br>
<br>

## No unsigned int with shader model 2 and 3
This is a current limitation of SPIRV-Cross. Unfortunately array indices are converted to unsigned int by DirectXShaderCompiler, so this will generate an error.
```HLSL
for(int i=0; i<10; i++)
    sum += someArray[i];
```
However, indexing arrays by constants does work, so manual loop unrolling can save the day.
```HLSL
sum += someArray[0];
sum += someArray[1];
...
```

## Array parameters not yet functional with shader model 2 and 3
Updating a shader array via effect parameters only works for vs_4_0 and ps_4_0 or newer. This is no ShaderConductor limitation, it's just not yet implemented on the MonoGame side. 
<br>
<br>

## Non square matrices are not supported with shader model 2 and 3
<br>
<br>

## Modulo operator is not supported with shader model 2 and 3
You need to replace modulo with something like this:
```HLSL
float Modulo(float x, float m) 
{
    return x - floor(x/m) * m;
}
```
If you need specific behaviour for negative numbers it might get more complicated than this. 
<br>
<br>

