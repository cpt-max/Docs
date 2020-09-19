# Shader models for OpenGL
With MojoShader the available shader models for OpenGL platforms were vs_2_0, ps_2_0 and vs_3_0, ps_3_0. Those are still available in ShaderConductor, the target GLSL version is identical, in order to support the same hardware as MojoShader did.<br> 
Additionally all the shader models currently available in DirectX can now also be used with OpenGL. The target OpenGL/GLSL versions are chosen automatically, trying to closely match feature levels between DirectX and OpenGL.<br>  
If this default mapping is not satisfactory, conditional code can still be used. ESSL is available as an additional define when targeting mobile platforms (Android, iOS)
```HLSL
#if ESSL
    #define VS_SHADERMODEL vs_4_0
    #define PS_SHADERMODEL ps_4_0
#else
    #define VS_SHADERMODEL vs_5_0
    #define PS_SHADERMODEL ps_5_0
#endif
```
<br>
The following table shows how shader models are mapped to GLSL and OpenGL versions, as well as the available shader stages.
<br>
<br>

|Shader model                                                                                                                                               | GLSL <br> (desktop)    | ESSL <br> (mobile) | OpenGL <br> (desktop) | OpenGL ES <br> (mobile) | Shader stages <br> (desktop) | Shader stages <br> (mobile) | 
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------|--------------------|-----------------------|-------------------------|------------------------------|-----------------------------|
|vs_2_0, ps_2_0 <br> vs_3_0, ps_3_0 <br> vs_4_0_level_9_1, ps_4_0_level_9_1 <br> vs_4_0_level_9_2, ps_4_0_level_9_2 <br> vs_4_0_level_9_3, ps_4_0_level_9_3 | 110                    | 100                | 2.0                   | 2.0                     |                              |                             |
|vs_4_0, ps_4_0 <br> vs_4_1, ps_4_1                                                                                                                         | 330                    | 300 es             | 3.3 + SSO*            | 3.0 + SSO*              | GS, TESS                     |                             |
|vs_5_0, ps_5_0                                                                                                                                             | 430                    | 320 es             | 4.3                   | 3.2                     | GS, TESS, CS                 | GS, TESS, CS                |

\* SSO: requires separate shader objects extension.

### Shader stages
- GS: Geometry shader
- TESS: Tesselation shader (Hull and Domain)
- CS: Compute shader

Compute shaders are not yet functional.
