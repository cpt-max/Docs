# Compute Shader Guide for MonoGame


## Table of Contents

1. [Compute Shader Basics](#compute-shader-basics)
    1. [Add Compute Shader HLSL](#add-compute-hlsl)
    2. [Execute Compute Shader](#execute-compute-shader)
2. [Structured Buffers](#structured-buffers)  
    1. [Add Structured Buffer in HLSL](#structured-buffer-hlsl)
    2. [Add Structured Buffer in C#](#create-structured-buffer)
    3. [Download Structured Buffer Data to CPU](#download-to-cpu)
    4. [Consume Structured Buffer Data in other Shader Stage](#consume-in-other-stage)
    5. [Append and Counter Buffers](#append-counter-buffers)
3. [Textures](#textures)
    1. [Add Texture in HLSL](#texture-hlsl)
    2. [Add Texture in C#](#create-texture)
4. [Vertex and Index Buffers](#vertex-index-buffers)
    1. [Add Vertex and Index Buffers in HLSL](#vertex-index-hlsl)
    2. [Add Vertex and Index Buffers in C#](#create-vertex-index)
5. [Indirect Drawing](#indirect-drawing)
    1. [Add Indirect Draw Buffer to HLSL](#indirect-draw-hlsl)
    2. [Add Indirect Draw Buffer in C#](#create-indirect-draw)
    3. [Draw and Dispatch using Indirect Draw Buffer](#draw-indirect-draw)
6. [Limitations](#limitations)
    

<br>



## 1. Compute Shader Basics <a name="compute-shader-basics"></a>

A compute shader performs arbitrary calculations on the GPU. The resulting output is written to a buffer or a texture, which can then be consumed by other shader stages, or downloaded to the CPU.<br>

These [sample projects](https://github.com/cpt-max/MonoGame-Shader-Samples) should cover most of the topics discussed in this guide.
<br><br>


### 1.1. Add Compute Shader HLSL <a name="add-compute-hlsl"></a>

The compute shader itself can be added to an fx file, just like a vertex or a pixel shader. A basic setup might look something like this:

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
    p.pos += p.vel; 
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

There has to be at least one output buffer/texture (RWStructuredBuffer, RWByteAddressBuffer, RWTexture), but there could be multiple. The "RW" stands for read/write. In addition to that, there can be multiple readonly input buffers/textures (StructuredBuffer, ByteAddressBuffer, Texture). In the example above the RWStructuredBuffer provides both, the input and the output.
<br><br>


### 1.2 Execute Compute Shader <a name="execute-compute-shader"></a>

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
This is a bad idea, as it might lead to frequent switches between regular drawing and compute, which is very expensive. Ideally you do all your compute for the entire frame, followed by all the drawing for the entire frame.
<br><br><br>



## 2. Structured Buffers <a name="structured-buffers"></a>

A structured buffer is basically a struct array, accessible to compute shaders. Since the struct fields are fully customizeable, they are very flexible.

A simple sample project involving a structured buffer can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_gpu_particles).
<br><br>


### 2.1. Add Structured Buffer in HLSL <a name="structured-buffer-hlsl"></a>

A structured buffer is created in HLSL using the StructuredBuffer (readonly) or RWStructuredBuffer (read/write) keyword.

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
    p.pos += p.vel; 
    Particles[globalID.x] = p; 
}
```
<br><br>


### 2.2. Add Structured Buffer in C# <a name="create-structured-buffer"></a>

The creation of a structured buffer in C# is almost identical to the creation of a VertexBuffer, in fact, they share a common base class.

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
If a structured buffer is only read from in the compute shader, the last constructor parameter can also be set to ShaderAccess.Read.<br>
The structured buffer is bound to the compute shader by assigning it to the corresponding effect parameter:
```C#
effect.Parameters["Particles"].SetValue(particleBuffer);
```
<br><br>


### 2.3 Download Structured Buffer Data to CPU <a name="download-to-cpu"></a>

If the goal is to use the compute shader results for graphical effects, you should, if possible, avoid downloading the data to the CPU, and use it directly in other shader stages, as shown in the next step. 
If the data needs to be used by the CPU, or you want to take a look at it for debugging purposes, you can get the data, just like you would with a vertex buffer:

```C#
var particles = new Particles[ParticleCount];
particleBuffer.GetData(particles, 0, ParticleCount);
```
<br><br>


### 2.4 Consume Structured Buffer Data in other Shader Stage <a name="consume-in-other-stage"></a>

You can't access an RWStructuredBuffer directly from non-compute stages, at least not in DirectX 11. 
Instead you need to add a regular StructuredBuffer. This means a single HLSL file may contain two definitions for the same buffer. 

```HLSL
StructuredBuffer<Particle> ParticlesReadOnly;

VertexOut VS(in VertexIn input)
{
    uint particleID = input.VertexID / 4;
    Particle p = ParticlesReadOnly[particleID];
    
    ...
}
```

While in DX you can read from a StructuredBuffer using shader model 4, in OpenGL you neeed to use shader model 5 (e.g. vs_5_0). 
<br><br>


### 2.5. Append and Counter Buffers <a name="append-counter-buffers"></a>

Append and counter buffers are a special kind of structured buffer, containing an internal counter value. 
They exist purely for convenience, making it easier to push and pop objects in a stack-like fashion, saving you from having to manage an atomic counter by yourself.

A sample project using an append buffer can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/object_culling_indirect_draw_append).

To create an append buffer in HLSL you use the AppendStructuredBuffer keyword, instead of RWStructuredBuffer. This enables you to use the Append() function on the buffer, in order to add objects to it.
```HLSL
AppendStructuredBuffer<Particle> Particles;

[numthreads(GroupSize, 1, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{   
    ...
    Particles.Append(p);
}
```
Respectively you can use the ConsumeStructuredBuffer keyword, if you want to pop objects from the buffer using the Consume() function.

With counter buffers you get the IncrementCounter() and DecrementCounter() HLSL functions, which return the counter value before the increment/decrement happend. The returned counter value can then be used to index the buffer. Counter buffers don't require a special keyword, they are defined as a regular RWStructuredBuffer in HLSL. 

In C# you turn a structured buffer into an append or counter buffer by supplying the optional constructor parameters at the end:
```C#
particleBuffer = new StructuredBuffer(GraphicsDevice, typeof(Particle), ParticleCount, BufferUsage.None, ShaderAccess.ReadWrite, StructuredBufferType.Append, 0);
```
StructuredBufferType.Append will create an append buffer, StructuredBufferType.Counter will create a counter buffer. 
The last parameter is the reset count. Whenever a compute dispatch happens invloving this buffer, the internal counter will be reset to the specified value. If you do not wish to reset the counter, use a value of -1. A new reset count can be assigned any time via StructuredBuffer.CounterResetValue.

Using the CopyCounterValue() function the internal counter can be copied into another buffer at the specified byte offset. This can be useful for setting the instance count in an indirect draw buffer for example (see indirect drawing):
```C#
particleBuffer.CopyCounterValue(indirectDrawBuffer, DrawInstancedArguments.ByteOffsetInstanceCount);
 ```

<br><br><br>



## 3. Textures <a name="textures"></a>

A sample project demonstrating writing to textures can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_write_to_texture).<br>
For 3D textures there's another sample [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_write_to_3d_texture).
<br><br>


### 3.1. Add Texture in HLSL <a name="texture-hlsl"></a>

A compute shader for texture output looks very much like a compute shader for buffer output. 
```HLSL
#define GroupSizeXY 8

RWTexture2D<float4> Texture;

[numthreads(GroupSizeXY, GroupSizeXY, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    float4 pixel = Texture[globalID.xy];
    
    // update pixel

    Texture[globalID.xy] = pixel;
}
```

The only real difference is that an RWTexture2D/RWTexture3D is used, instead of an RWStructuredBuffer. 
<br><br>


### 3.2. Add Texture in C# <a name="create-texture"></a>

The output texture is created in C# just like a normal texture, but you have to add the ShaderAccess parameter, and set it to ReadWrite:
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
<br><br><br>



## 4. Vertex and Index Buffers <a name="vertex-index-buffers"></a>

A sample project demonstrating writing to vertex and index buffers can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_write_to_vertex_buffer).
<br><br>


### 4.1. Add Vertex and Index Buffers in HLSL <a name="vertex-index-hlsl"></a>
    
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
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    uint indexID = globalID.x; 
    uint index = Indices.Load(indexID * 4); // with IndexElementSize.ThirtytwoBits every index is 4 bytes. 
    
    // modify index

    Indices.Store(indexID * 4, index);
}
```

Beware that things get a bit more complicated with 16 bit indices (see sample project), as bitwise operations are needed to extract the 16 bit indices out of the 32 bit values read from the buffer.
<br><br>



### 4.2. Add Vertex and Index Buffers in C# <a name="create-vertex-index"></a>

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
<br><br><br>



## 5. Indirect Drawing <a name="indirect-drawing"></a>

When objects are beeing processed in a compute shader, the CPU sometimes doesn't know how many objects need to be drawn, especially when the compute shader is responsible for spawn and destroy. Possible ways to deal with situations like these could be:

- always draw the maximum number of objects possible, making sure unused objects are not visible in the end. Disadvantage: Drawing more objects than needed and complicating  shaders, as they need to discard unused objects (e.g. collapsing vertices into a single point)
- write the number of active objects into a buffer in a compute shader, then download that data to the CPU for drawing. Disadvantage: GPU and CPU are forced to synchronize, which can be very bad for performance.

With indirect draw a 3rd option becomes available. Like with option 2 the total number of active objects is written into a buffer from a compute shader, but that data never gets downloaded to the CPU. When it's time to draw, instead of passing the number of objects directly to the draw call, the buffer containing the instance count is passed.

A simple sample project demonstrating indirect draw can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/object_culling_indirect_draw).
A more complex indirect draw sample can be found [here](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/indirect_draw_instances).
See the description in the readme for more details.
<br><br>


### 5.1. Add Indirect Draw Buffer to HLSL <a name="indirect-draw-hlsl"></a> 

An indirect draw buffer contains the arguments for a draw or dispatch call.
 
Indirect draw buffers have to be declared as ByteAddressBuffer's in HLSL. That means positions in the buffer have to be given in bytes. Every uint in the buffer is 4 bytes. If the instance count is the 2nd argument in the buffer, the corresponding byte position is 4, as the first uint before it will take 4 bytes. 
The instance count in an indirect draw buffer is commonly updated using an atomic/interlocked counter:
```HLSL
#define GroupSize 64

RWByteAddressBuffer IndirectDraw;

[numthreads(GroupSize, 1, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 groupID : SV_GroupID,
        uint  localIndex : SV_GroupIndex, uint3 globalID : SV_DispatchThreadID)
{
    bool isObjectVisible = ...
    if (isObjectVisible)
    {    
        uint outID;
        IndirectDraw.InterlockedAdd(4, 1, outID); // increment the instance count in the indirect draw buffer (starts at byte 4) 
        ObjectsToDraw[outID] = ...; // add the object to the output buffer, this buffer will later be read by the vertex shader that draws the objects
    }
}
```
<br><br>


### 5.2. Add Indirect Draw Buffer in C# <a name="create-indirect-draw"></a>

The indirect draw buffer is created just like other buffer types:
```C#
indirectDrawBuffer = new IndirectDrawBuffer(GraphicsDevice, BufferUsage.None, ShaderAccess.ReadWrite);
```
By default the indirect draw buffer will have space for 5 unsigned integers, which is enough for all of the 3 possible indirect draw/dispatch calls. You can also provide the number of uint's in the buffer as the last parameter. That way you can fit multiple draw calls into the same buffer, or have space for some extra variables.

You probably want to initialize the indirect draw buffer on the CPU. Usually not all draw parameters need to be dynamic, very often only the instance count is. Also you may need to reset the instance count in the buffer to zero every frame, so the compute shader can then increment it by one for every active instance.
```C#
indirectDrawBuffer.SetData(new DrawIndexedInstancedArguments
{
    IndexCountPerInstance = indexBuffer.IndexCount,
    InstanceCount = 0,
    StartIndexLocation = 0,
    BaseVertexLocation = 0,
    StartInstanceLocation = 0,
});
```
For every draw/dispatch call variant (DrawIndexedInstanced, DrawInstanced and DispatchCompute), there's a SetData variant taking the respective argument struct as a parameter (DrawIndexedInstancedArguments, DrawInstancedArguments and DispatchComputeArguments). Alternatively you can also set the data using an array of uint's, just like with other buffer types. 

Sometimes you want to fit multiple draw or dispatch calls into a single indirect draw buffer. You can use the following pattern to initialize the buffer:
```C#
indirectDrawBuffer = new IndirectDrawBuffer(GraphicsDevice, BufferUsage.None, ShaderAccess.ReadWrite, DispatchComputeArguments.Count + DrawInstancedArguments.Count);

var dispatchIndirectArgs = new DispatchComputeArguments
{
    GroupCountX = groupCount,
    GroupCountY = 1,
    GroupCountZ = 1,
};
            
var drawIndirectArgs = new DrawInstancedArguments
{
    VertexCountPerInstance = numVertices,
    InstanceCount = numInstances,
    StartVertexLocation = 0,
    StartInstanceLocation = 0,
};

var data = new uint[indirectDrawBuffer.ElementCount];
int offset = 0;

offset += dispatchIndirectArgs.WriteToArray(data, offset);
offset += drawIndirectArgs.WriteToArray(data, offset);

buffer.SetData(data);
```
<br><br>


### 5.3. Draw and Dispatch using Indirect Draw Buffer  <a name="draw-indirect-draw"></a>

In order to draw objects indirectly you use one of the two indirect draw calls DrawIndexedInstancedPrimitivesIndirect or DrawInstancedPrimitivesIndirect
```C#
GraphicsDevice.SetVertexBuffer(vertexBuffer);
GraphicsDevice.Indices = indexBuffer;
    
foreach (var pass in effect.CurrentTechnique.Passes)
{
    pass.Apply();
    GraphicsDevice.DrawIndexedInstancedPrimitivesIndirect(PrimitiveType.TriangleList, indirectDrawBuffer);
}
```
or make an indirect dispatch call like this
```C#
foreach (var pass in effect.CurrentTechnique.Passes)
{
    pass.ApplyCompute();
    GraphicsDevice.DispatchComputeIndirect(indirectDispatchBuffer);
}
```
<br><br><br>



## 6. Limitations <a name="limitations"></a>

- When you call GetData() on a structured buffer, where the corresponding struct contains a bool parameters, you will currently get an exception, telling you that bool is not a blittable type. Just use int instead.
- Writing to cubemaps, texture arrays, or into specific texture mipmap levels is currently not possible. However, you can use the new CopyData() functions for textures as a workaround. Write to a regular texture and then copy the data into cubemap faces, mipmap levels, or array slices. Since the copying happens on the GPU, it's relatively efficient.

<br><br><br>
