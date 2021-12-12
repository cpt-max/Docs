# Using the MonoGame Compute Fork in your own Projects
The [custom compute fork](https://github.com/cpt-max/MonoGame) is based on the current (as of this writing) development branch of MonoGame.<br>
You have the same options of target frameworks to choose from for your project:
- .Net Framework (4.52 or higher for WindowsDX, minimum for OpenGL?)
- .Net Core (minimum version?)
- .Net 5
- .Net 6

Outside of the [migrations needed for custom OpenGL shaders](https://github.com/cpt-max/Docs/blob/master/Migrating%20shaders%20to%20ShaderConductor.md),
there shouldn't be any breaking changes compared to the stable 3.8 release.
You can switch an existing project over by swapping the MonoGame package references in your project with the corresponding compute packages.
<br><br>
If you are already building MonoGame from source, there's nothing special, just switch (or merge) to the compute fork, and don't foget that you have to use the updated content builder (MGCB and MGFXC) for compiling your content.
<br><br>



## NuGet Packages
NuGet packages for the MonoGame compute fork are available on nuget.org.<br>
Here is a list of MonoGame packages which already have corresponding compute packages:
- MonoGame.Framework.WindowsDX -> MonoGame.Framework.Compute.WindowsDX
- MonoGame.Framework.DesktopGL -> MonoGame.Framework.Compute.DesktopGL
- MonoGame.Content.Builder.Task -> MonoGame.Content.Builder.Task.Compute
- dotnet-mgcb-editor -> dotnet-mgcb-editor-compute

You can switch packages by right clicking a project in Visual Studio, then select <b>Manage NuGet Packages</b>, or by editing the csproj file in a text editor:
```XML
<ItemGroup>
  <PackageReference Include="MonoGame.Framework.Compute.DesktopGL" Version="3.8.1.1" />
  <PackageReference Include="MonoGame.Content.Builder.Task.Compute" Version="3.8.1" />
</ItemGroup>
```
<br>



## Platform Support

<b>- Windows</b><br>
Supported through WindowsDX and DesktopGL
<br>

<b>- Linux</b><br>
Supported through DesktopGL (only tested on Ubuntu 20.04)
<br>

<b>- Mac</b><br>
All the unit tests are passed on Mac, so you can technically use it, but it fails to create anything higher than an OpenGL 2.1 context.
This limits you to shader model 2 and 3. The new shader stages require shader model 4 and 5<br>
The issue can probably be resolved pretty easily as described [here](https://stackoverflow.com/questions/19658745/why-is-my-opengl-version-always-2-1-on-mac-os-x), 
but this has not yet been investigated any further. Also this won't make compute shaders work, as MacOS only supports OpenGL 4.1, while compute was added with 4.3.
<br>

<b>- Android and iOS</b><br>
Both Android and iOS have been tested successfully with a simple test shader, compiled through ShaderConductor.<br>
This was before compute shaders were added, no testing happened since then.<br>
There is no reason why compute shaders shouldn't work on Android, it needs to be tested though.<br>
On iOS compute shaders are not available through OpenGL at all.
<br><br>



## Setting up your development environment
The official setup guide still applies 
([windows](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_windows.html), 
[macOS](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_macos.html), 
[Ubuntu 20.04](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_ubuntu.html))
with the following modifications:
<br><br>


### MGCB Editor
The modified MGCB editor is called dotnet-mgcb-editor-compute instead of dotnet-mgcb-editor. Install it like this
```
dotnet tool install --global dotnet-mgcb-editor-compute
```
The two editor versions can coexist happily on the same machine.<br>
To launch it from the command line:
```
mgcb-editor-compute
```
To register it:
```
mgcb-editor-compute --register
```
This will override a previous registration from the non-compute editor.
On one Ubuntu machine an explicit unregister from the old editor was needed before the new one could be registerd
```
mgcb-editor --unregister
```
On another Ubuntu machine the registration always failed. Fortunately the registration isn't that important.
<br><br>


### Templates 
No new templates have been created for compute, but since you only need to swap the NuGet packages, the old templates should work just fine. 
<br><br>


### Wine for effect compilation on Linux and Mac
This is not needed anymore, as ShaderConductor is platform independent. If you still want to compile shaders using MojoShader as well, you will still need it though.
<br><br>
