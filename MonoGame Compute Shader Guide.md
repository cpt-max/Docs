# Compute Shader Guide for MonoGame

A compute shader performs arbitrary calculations on the GPU. The resulting output is written to a buffer, which can then be consumed by other shader stages, or downloaded to the CPU.<br>
Currently this buffer has to be a StructuredBuffer, eventually it should also be possible to write to a VertexBuffer or a Texture directly, and maybe other, less commonly used, buffer types.

The shown code snippets are mostly taken from this [sample project](https://github.com/cpt-max/MonoGame-Shader-Samples/tree/compute_gpu_particles).
<br><br>

## 1.) Adding the compute shader HLSL

The compute shader itself can be added to an fx file, just like any other shader. A basic setup might look something like this:

```HLSL
#define ComputeGroupSize 64

struct Particle
{
    float2 pos;
    float2 vel;
};

RWStructuredBuffer<Particle> Particles;

[numthreads(ComputeGroupSize, 1, 1)]
void CS(uint3 localID : SV_GroupThreadID, uint3 dispatchID : SV_GroupID,
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
    particleBuffer = new StructuredBuffer(GraphicsDevice, typeof(Particle), MaxParticleCount, BufferUsage.None, true);

    // optionally initialize the buffer
    var particles = new Particle[MaxParticleCount];
    for (int i = 0; i < MaxParticleCount; i++)
        particles[i] = ...

    particleBuffer.SetData(particles);
}
```
<br>


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
If the data needs to be used the CPU, or you want to take a look at it for debugging purposes, you can get the data just like you would with a VertexBuffer.

```
var particles = new Particles[ParticleCount];
particleBuffer.GetData(particles, 0, ParticleCount);
```
<br>


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

 and assign it the same buffer in C#.
 
```C#
effect.Parameters["ParticlesReadOnly"].SetValue(particleBuffer);
```




