
# Migrating shaders to ShaderConductor

In order to successfully compile shaders with ShaderConductor some modifications might be needed to the HLSL source code.

## COLOR semantic on pixel shader not allowed
The COLOR semantic has to be replaced with SV_TARGET.
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
Here's an example of what texture sampling might look like in DX9.

```HLSL
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

 The equivalent DX10 version that also works with ShaderConductor looks like this.   
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

## Binding textures or sampler states by register
The preferred method should generally be to bind by name whenever possible. With MojoShader there were cases where textures wouldn't get an effect parameter, so you had to bind by register. This should not be the case anymore. You should always be able to bind textures like this
```C#
// in HLSL
Texture2D MyTexture;

// in C#
effect.Parameters["MyTexture"].SetValue(myTexture);
```
Unfortunately there are no effect parameters for samplers currently, which means if you want to assign a sampler state in C# you have to bind by register. In the past, when you had only a single sampler in your shader, you could assume it would be bound to register index 0. 
```HLSL
// in HLSL
sampler MySampler;

// in C#
GraphicsDevice.SamplerStates[0] = SamplerState.PointClamp;
```
This is not the case anymore. If you want to guarantee your sampler is bound to a specific register, you need to specify the register explicitly.
```HLSL
sampler MySampler : register(s0);
```
