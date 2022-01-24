# Using the MonoGame Compute Fork in your own Projects
The [custom compute fork](https://github.com/cpt-max/MonoGame) is based on the current (as of this writing) development branch of MonoGame.<br>
You have the same options of target frameworks to choose from for your project:
- .Net Framework (4.52 or higher for WindowsDX, minimum for OpenGL?)
- .Net Core (DesktopGL only: netcoreapp3.1)
- .Net 5
- .Net 6

Outside of the [migrations needed for custom OpenGL shaders](https://github.com/cpt-max/Docs/blob/master/Migrating%20shaders%20to%20ShaderConductor.md) and the switch from .Net Core to .Net 5 for WindowsDX, there shouldn't be any breaking changes compared to the stable 3.8 release.
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
- dotnet-mgcb-editor -> dotnet-mgcb-editor-compute   (Windows and Linux only, a Mac version exists as a direct download)

You can switch packages by right clicking a project in Visual Studio, then select <b>Manage NuGet Packages</b>, or by editing the csproj file in a text editor:
```XML
<ItemGroup>
  <PackageReference Include="MonoGame.Framework.Compute.DesktopGL" Version="3.8.1.2" />
  <PackageReference Include="MonoGame.Content.Builder.Task.Compute" Version="3.8.1.2" />
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
Supported through DesktopGL, however the OpenGL version is currently limited to 2.1. This limits you to shader model 2 and 3. 
The main reason for using this fork is to get shader model 4 and 5 support, rendering the Mac version almost useless in it's current state.<br>
A higher OpenGL version (4.1) can easily be created by switching from the legacy to a core OpenGL context, but this causes follow-up problems that still need to be resolved.
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


### MGCB Editor for Windows and Linux
The modified MGCB editor is called dotnet-mgcb-editor-compute instead of dotnet-mgcb-editor. This tool only works on Windows and Linux. For Mac there is a direct download, see below. You can install the tool like this:
```
dotnet tool install --global dotnet-mgcb-editor-compute
```
The two editor versions can coexist happily on the same machine.<br>
To launch it from the command line:
```
mgcb-editor-compute
```
You can also find/launch it by navigating to the default installation folder for .Net tools on your OS. 
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


### MGCB Editor for Mac
On Mac the editor can be downloaded [here](https://www.dropbox.com/s/h4448bafzyul49m/MGCB-Editor.tgz?dl=1).
You need to have .Net 5 installed. On M1 that means installing the x64 runtime, as there is no .Net 5 SDK for Arm. 
This is not a signed app, so on newer OSX versions you need to remove the quarantine attribute after extracting, otherwise it won't launch and you get a message that the app is damaged.
```
xattr -d com.apple.quarantine /PathTo/MGCB-Editor.app 
``` 
<br><br>


### Templates 
No new templates have been created for compute, but since you only need to swap the NuGet packages, the old templates should work just fine. For Windows you may need to update the target framework to .Net 5+ in the csproj file:
```XML
<TargetFramework>.net5.0-windows</TargetFramework>
```
<br>


### Wine for effect compilation on Linux and Mac
This is not needed anymore, as ShaderConductor is platform independent. If you still want to compile shaders using MojoShader as well, you will still need it though.
<br><br>
