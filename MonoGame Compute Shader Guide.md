# Compute Shader Guide for MonoGame

A compute shader performs arbitrary calculations on the GPU. The resulting output is written to a buffer or a texture, which can then be consumed by other shader stages, or downloaded to the CPU.<br>

The code snippets in the beginning of this guide are mostly taken from this [sample project](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_gpu_particles), which uses a compute shader to fill a StructuredBuffer. 
While using a StructuredBuffer is probably the most common and flexible use case, you can also write to textures, vertex or index buffers directly. More information about those can be found at the end of this guide.
<br><br>

## 1.) Add the compute shader HLSL

The compute shader itself can be added to an fx file, just like any other shader. A basic setup might look something like this:

```HLSL
#define GroupSize 64

struct Particle
{
    float2 pos;
    float2 vel;
};

RWStructuredBuffer<Particle> Particles;

[numthreads(GroupSize, 1, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    Particle p = Particles[globalID.x]; 
    
    // update p.pos and p.vel 
    
    Particles[globalID.x] = p; 
}

technique Tech0
{
    pass Pass0
    {
        ComputeShader = compile cs_5_0 CS();
    }
}
```
A compute shader can be alone in a separate technique, but it can also be part of a teqchnique, that already contains a vertex or a pixel shader. 
Just because the compute shader shares a technique with a vertex shader, does not mean it will automatically execute, whenever the vertex shader executes. 
Compute shaders are differen in this regard from other shader stages. They are not tied into the normal shader pipeline, they always run separately.

There has to be at least one output buffer (RWStructuredBuffer), but there could be multiple. The "RW" stands for read/write. In addition to that, there can be multiple readonly input buffers (StructuredBuffer). In the example above the RWStructuredBuffer also provides the input.
<br><br>


## 2.) Create the StructuredBuffer in C#

The creation of a StructuredBuffer is almost identical to the creation of a VertexBuffer, in fact, they share a common base class.

```C#
struct Particle
{
    public Vector2 pos;
    public Vector2 vel;
};

StructuredBuffer particleBuffer;

protected override void LoadContent()
{
    particleBuffer = new StructuredBuffer(GraphicsDevice, typeof(Particle), MaxParticleCount, BufferUsage.None, ShaderAccess.ReadWrite);

    // optionally initialize the buffer
    var particles = new Particle[MaxParticleCount];
    for (int i = 0; i < MaxParticleCount; i++)
        particles[i] = ...

    particleBuffer.SetData(particles);
}
```
If a StructuredBuffer is only read from in the compute shader, the last constructor parameter can also be set to ShaderAccess.Read.
<br><br>


## 3.) Execute the compute shader

Executing a compute shader works very similar to drawing primitives.<br>
When drawing primitives you would call ```pass.Apply``` followed by ```GraphicsDevice.Draw...```.<br>
For executing a compute shader you call ```pass.ApplyCompute``` followed by ```GraphicsDevice.DispatchCompute```.

The StructuredBuffer gets bound to the compute shader by assigning it to the corresponding effect parameter, just like other shader parameters.

```C#
effect.Parameters["Particles"].SetValue(particleBuffer);

foreach (var pass in effect.CurrentTechnique.Passes)
{
   pass.ApplyCompute();
   GraphicsDevice.DispatchCompute(groupCountX, 1, 1);
}
```

The total number of particles computed in this case is ```groupCountX``` times ```ComputeGroupSize``` from the compute shader HLSL. 
```groupCountX``` could be calculated like this:

```C#
// if ParticleCount is a multiple of ComputeGroupSize
int groupCountX = ParticleCount / ComputeGroupSize;

// otherwise
int groupCountX = (int)Math.Ceiling((double)ParticleCount / ComputeGroupSize);
```

If a pass does not contain a compute shader, the ```ApplyCompute``` and ```DispatchCompute``` calls will be ignored.

If the compute shader is part of the same technique as a vertex shader, you might be tempted to execute the compute shader in the same loop, that also draws the primitives. 
This is a bad idea, as it might lead to frequent switches between regular drawing and compute, which is very expensive. Ideally you do all your compute for the entire frame, then do all the drawing for the frame.
<br><br>


## 4.) Download the data in the StructuredBuffer to the CPU

If the goal is to use the compute shader results for graphical effects, you should, if possible, avoid downloading the data to the CPU, and use it directly in other shader stages, as shown in the next step. 
If the data needs to be used by the CPU, or you want to take a look at it for debugging purposes, you can get the data, just like you would with a VertexBuffer:

```C#
var particles = new Particles[ParticleCount];
particleBuffer.GetData(particles, 0, ParticleCount);
```
<br><br>


## 5.) Consume the StructuredBuffer directly in other shader stages

You can't access an RWStructuredBuffer directly in non-compute stages, at least not in DirectX 11. 
Instead you need to add a regular StructuredBuffer in HLSL

```HLSL
StructuredBuffer<Particle> ParticlesReadOnly;

VertexOut VS(in VertexIn input)
{
    uint particleID = input.VertexID / 4;
    Particle p = ParticlesReadOnly[particleID];
    
    ...
}
```

 and assign it the same buffer in C#:
 
```C#
effect.Parameters["ParticlesReadOnly"].SetValue(particleBuffer);
```
<br><br>


## 6.) Write directly to textures

A sample project demonstrating writing to textures can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_write_to_texture).

A compute shader for texture output looks very much like a compute shader for buffer output. 
```HLSL
#define GroupSizeXY 8

RWTexture2D<float4> Texture;

[numthreads(GroupSizeXY, GroupSizeXY, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 grouphID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    float4 pixel = Texture[globalID.xy];
    
    // update pixel

    Texture[globalID.xy] = pixel;
}
```

The only real difference is that an RWTexture2D is used, instead of an RWStructuredBuffer. 

The output texture is created in C# just like a normal texture, but you have to use the constructor overload with the ShaderAccess parameter, and set it to ReadWrite:
```C#
computeTexture = new Texture2D(GraphicsDevice, width, height, false, SurfaceFormat.Color, ShaderAccess.ReadWrite);
```

The texture is bound to the compute shader by assigning it to the corresponding effect parameter:
```C#
effect.Parameters["Texture"].SetValue(computeTexture);

foreach (var pass in effect.CurrentTechnique.Passes)
{
    pass.ApplyCompute();
    GraphicsDevice.DispatchCompute(groupCountX, groupCountY, 1);
}
```
<br><br>


## 7.) Write to vertex and index buffers

A sample project demonstrating writing to vertex and index buffers can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_write_to_vertex_buffer).

In order to access a vertex buffer from a compute shader a ByteAddressBuffer (readonly) or RWByteAddressBuffer (writeable) needs to be defined: 
```HLSL
#define GroupSizeXY 64

RWByteAddressBuffer Vertices;

[numthreads(GroupSize, 1, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    uint vertexID = globalID.x;
    uint posByteInd = vertexID * 32; // 32 bytes per vertex element: float3 position, float3 normal, float2 texCoord => 8 floats => 32 bytes 
    uint normByteInd = posByteInd + 12; // 12 is the byte size for the float3 position element
    
    float3 pos  = asfloat(Vertices.Load3(vertexByteInd));
    float3 norm = asfloat(Vertices.Load3(normByteInd)); 
    
    // modify pos and norm

    Vertices.Store3(vertexByteInd, asuint(pos));
    Vertices.Store3(normByteInd, asuint(norm));
}
```
The Load, Load2, Load3 and Load4 functions return uint's, and the corresponding Store functions expect uint's, that's why the asfloat/asuint conversions are neccessary.

A similar compute shader for index buffer access could look like this:
```HLSL
#define GroupSizeXY 64

RWByteAddressBuffer Indices;

[numthreads(GroupSize, 1, 1)]
void CS_FlipIndices(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    uint indexID = globalID.x; 
    uint index = Indices.Load(indexID * 4); // with IndexElementSize.ThirtytwoBits every index is 4 bytes. 
    
    // modify index

    Indices.Store(indexID * 4, index);
}
```

Beware that things get a bit more complicated with 16 bit indices (see sample project), as bitwise operations are needed to extract the 16 bit indices out of the 32 bit values read from the buffer.

Similar to textures, a vertex or index buffer needs to be created with the ShaderAccess parameter set to ReadWrite or Read, in order to be accessible from compute shaders:
```C#
vertexBuffer = new VertexBuffer(GraphicsDevice, VertexPositionNormalTexture.VertexDeclaration, vertexCount, BufferUsage.WriteOnly, ShaderAccess.ReadWrite);
indexBuffer = new IndexBuffer(GraphicsDevice, IndexElementSize.ThirtyTwoBits, indexCount, BufferUsage.WriteOnly, ShaderAccess.ReadWrite);
```

The buffers are bound to the compute shader by assigning them to the corresponding effect parameters:
```C#
effect.Parameters["Vertices"].SetValue(vertexBuffer);
effect.Parameters["Indices"].SetValue(indexBuffer);

foreach (var pass in effect.CurrentTechnique.Passes)
{
    pass.ApplyCompute();
    GraphicsDevice.DispatchCompute(groupCount, 1, 1);
}
```
<br><br>






