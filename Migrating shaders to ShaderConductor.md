
# Migrating shaders to ShaderConductor

ShaderConductor is used when shaders are compiled for OpenGL platforms like DesktopGL, Android or iOS. 
In order to compile successfully some modifications to the HLSL source code may be neccessary.

## SV prefix for shader semantics
Pixel shaders in MojoShader used the COLOR and DEPTH output semantic, now it's SV_POSITION and SV_DEPTH.
In order to pass vertex positions from the vertex to the pixel shader, the POSITION semantic was used in the past, this needs to be SV_POSITION now.
```HLSL
float4 MyVertexShader(float4 inputPos: POSITION) : SV_POSITION
{ ... }

float4 MyPixelShader() : SV_TARGET
{ ... }
```

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

sampler MySampler = sampler_state 
{
    Texture = (MyTexture);
};

float4 MyPixelShader(VertexOut input) : SV_TARGET
{
    return MyTexture.Sample(MySampler, input.texCoord);
}
```
Note the three things that changed:
- **texture** becomes **Texture2D**. **Texture1D**, **Texture3D** and **TextureCube** are also possible.
- **sampler2D** becomes just **sampler** without the dimension, same for **sampler1D**, **sampler3D** and **samperCUBE**
- **tex2D()** becomes **MyTexture.Sample()**, same for **tex1D()**, **tex3D()** and **texCUBE()**. There are also DX10 equivalents for the more specialized texture sampling functions like **tex2Dlod()** which becomes **MyTexture.SampleLevel(MySampler, TexCoord, Level)**. More details about the available sampling functions can be found [here](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-type).

## Binding textures and samplers by register
The preferred method should generally be to bind by name whenever possible. With MojoShader there were cases where textures wouldn't get an effect parameter, so binding by register was the only option. This should not be the case anymore. You should always be able to bind textures like this.
```C#
// in HLSL
Texture2D MyTexture;

// in C#
effect.Parameters["MyTexture"].SetValue(myTexture);
```
Unfortunately there are no effect parameters for samplers currently. If you want to assign a sampler state in C#, you have to bind by register. You should never rely on automatic register assignment in this case, even if there is only a single sampler in your shader. If you want to guarantee your sampler is bound to a specific register, you need to specify the register explicitly in HLSL.
```HLSL
// in HLSL
sampler MySampler : register(s0);

// in C#
GraphicsDevice.SamplerStates[0] = SamplerState.PointClamp;
```

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

## Non square matrices are not supported with shader model 2 and 3

