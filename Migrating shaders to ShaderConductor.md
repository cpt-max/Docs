
# Migrating shaders to ShaderConductor

ShaderConductor is used when shaders are compiled for OpenGL platforms like DesktopGL, Android or iOS. 
In order to compile successfully some modifications might be needed to the HLSL source code.

## COLOR semantic on pixel shader becomes SV_TARGET
```HLSL
float4 MyPixelShader(VertexOut input) : COLOR
{ ... }
```
becomes
```HLSL
float4 MyPixelShader(VertexOut input) : SV_TARGET
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

This equivalent DX10 code will also work with ShaderConductor.   
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

## Binding textures and sampler states by register
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

## Unsigned int for legacy targets 
array indexing

## Array parameters for legacy targets

## Matrix negate
https://github.com/microsoft/DirectXShaderCompiler/commit/8ffccecea1ed20eba397a304e482772d3bc5777f
