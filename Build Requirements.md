# Using the MonoGame Compute Fork in your own Projects
The [custom compute fork](https://github.com/cpt-max/MonoGame) is based on the current (3.8.1) development branch of MonoGame.<br>
You can switch an existing project over by swapping the MonoGame package references in your project with the corresponding compute packages.
You can switch packages by right clicking a project in Visual Studio, then select <b>Manage NuGet Packages</b>, or by editing the csproj file in a text editor:
```XML
<ItemGroup>
  <PackageReference Include="MonoGame.Framework.Compute.DesktopGL" Version="3.8.2.0" />
  <PackageReference Include="MonoGame.Content.Builder.Task.Compute" Version="3.8.2.0" />
</ItemGroup>
```

If you are already building MonoGame from source, there's nothing special, just switch (or merge) to the compute fork, and don't foget that you have to use the updated content builder (MGCB and MGFXC) for compiling your content.
<br><br>
You get the updated content builder automatically through Nugets if you have a proper [dotnet-tools.json](https://github.com/cpt-max/MonoGame-Shader-Samples/blob/overview/.config/dotnet-tools.json) file in your projects .config folder, as well as a tool restore section in your [.csproj file](https://github.com/cpt-max/MonoGame-Shader-Samples/blob/overview/ShaderSampleGL.csproj).
```xml
  <Target Name="RestoreDotnetTools" BeforeTargets="Restore">
    <Message Text="Restoring dotnet tools" Importance="High" />
    <Exec Command="dotnet tool restore" />
  </Target>
```
<br>

## NuGet Packages
NuGet packages for the MonoGame compute fork are available on nuget.org.<br>
Here is a list of MonoGame packages which already have corresponding compute packages:
- MonoGame.Framework.WindowsDX -> MonoGame.Framework.Compute.WindowsDX
- MonoGame.Framework.DesktopGL -> MonoGame.Framework.Compute.DesktopGL
- MonoGame.Framework.Android -> MonoGame.Framework.Compute.Android
- MonoGame.Content.Builder.Task -> MonoGame.Content.Builder.Task.Compute
<br>

These are the MGCB editor related Nugets, which should get downloaded automatically, if you set up tool restore as described above:
- dotnet-mgcb -> dotnet-mgcb-compute
- dotnet-mgcb-editor -> dotnet-mgcb-editor-compute
- dotnet-mgcb-editor-linux -> dotnet-mgcb-editor-compute-linux
- dotnet-mgcb-editor-windows -> dotnet-mgcb-editor-compute-windows
- dotnet-mgcb-editor-compute -> dotnet-mgcb-editor-compute-mac
<br>

## Platform Support

<b>- Windows</b><br>
Supported through WindowsDX and DesktopGL
<br>

<b>- Linux</b><br>
Supported through DesktopGL (only tested on Ubuntu 20.04)
<br>

<b>- Android</b><br>
Supported through Android
<br>

<b>- Mac</b><br>
Supported through DesktopGL, however the OpenGL version is currently limited to 2.1. This limits you to shader model 2 and 3. 
The main reason for using this fork is to get shader model 4 and 5 support, rendering the Mac version almost useless in it's current state.<br>
A higher OpenGL version (4.1) can be created by switching from the legacy to a core OpenGL context, but this causes follow-up problems that still need to be resolved.<br>
Compute shaders won't be available through OpenGL at all, since MacOS doesn't support OpenGL 4.3.
<br>

<b>- iOS</b><br>
iOS has been tested successfully with a simple test shader, compiled through ShaderConductor.<br>
This was before compute shaders were added, no testing happened since then. There is no published Nuget for iOS yet.<br>
Compute shaders for iOS won't be available through OpenGL at all, since iOS doesn't support OpenGL 4.3.
<br><br>

## Setting up your development environment
The official setup guide still applies 
([windows](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_windows.html), 
[macOS](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_macos.html), 
[Ubuntu 20.04](https://docs.monogame.net/articles/getting_started/1_setting_up_your_development_environment_ubuntu.html))
with the following modifications:
<br><br>


### MGCB Editor
The modified MGCB editor Nugets all got "compute" added to their names. The same applies when you use them on the command line.<br>
From the projects directroy you can type ```dotnet mgcb-editor-compute``` in order to launch the MGCB editor.<br>
```dotnet tool install -g dotnet-mgcb-compute``` will install MGCB as a global tool, which can then be used to build content using the ```mgcb-compute``` command.
<br><br>


### Templates 
No new templates have been created for compute, but since you only need to swap the NuGet packages, the existing templates should work just fine. 
<br><br>


### Wine for effect compilation on Linux and Mac
As long as ShaderConductor is used for compiling shaders Wine is not needed anymore, as ShaderConductor is platform independent. Shaders compiled through MojoShader will still need Wine though. A shader file is compiled through ShaderConductor if any of the shaders in the file uses shader model 4 or higher. If all shaders in the file are shader model 2 or 3, you can still force ShaderConductor by adding a CONDUCTOR define. Have a look at the [Migration Guide](https://github.com/cpt-max/Docs/blob/master/Migrating%20shaders%20to%20ShaderConductor.md) for more details.
<br><br>
