# Authoring Roslyn Code Analyzers

Roslyn Code Analyzers are very powerful tools that allow you to enforce certain semantics in your code beyond what the compiler itself can provide. This allows you to extend the compiler to provide warnings or errors about structures in your code that might result in runtime issues, but to detect those potential problems at compile, and even design time.

This article isn't about authoring the analyzers themselves. There are many great resources for getting started with authoring anayzers. One of the best such resources is the xunit.analyzers package that provides analyzers for constructing xunit unit tests. These analyzers enforce semantics that are expected by the xunit testing framework; semantics that, if violated, will result in test failures, or runtime exceptions. The source for these analyzers is open, and provide a great example of how to structure your own analysis code base.

Rather, what I want to demonstrate here is how to have a code analysis project that can be developed in parallel with a codebase.

The Visual Studio template for a code analysis project will provide an analyzer project and a unit test project. These projects can be added to a solution, but the problem is that there is no way to add a project reference to an analyzer project. Analyzer references, which show up under project dependencies, are added by adding a reference to the alaysis dll itself. Adding a project reference to an analyzer project will add a normal project reference, which isn't what we want. Analyzers are only used at compile time, not runtime.


How can we make this work? 

1. When the analysis project is built, copy the dll to a well know location that the other projects can refer to.

2. Add a global MSBuild targets file that will include the analysis dll as an analyzer reference to all the projects.


Add a custom target to the analyzer project:

```xml
<Target Name="CopyAnalyzer" AfterTargets="Build" Inputs="$(TargetPath)" Outputs="$(CodeAnalysisLib)">
    <Error Condition="'$(AssemblyName)' != '$(CodeAnalysisLibName)'" Text="Code analysis assembly $(AssemblyName) does not match expected name $(CodeAnalysisLibName)"/> 
    <MakeDir Condition="!Exists($(LibFolder))" Directories="$(LibFolder)" />
    <Copy SourceFiles="$(TargetPath)" DestinationFolder="$(LibFolder)" />
  </Target>
```

The first thing this does is validate that the code analysis assembly name is the same name that we define in the global targets. This will report an error that would cause things to fail if the project was renamed, for example.
This target will create a folder "LibFolder" to hold the analyzer, we will define this location in the next part. It then copies the analyzer dll (TargetPath) to that folder.

Create a global targets file named "Directory.Build.targets" in the root of your solution. This file needs to be *above* all the other csproj files in your solution. By using this exact name, the standard MSBuild scripts will cause it to be automatically imported into all the csproj files, including the code analysis project. In this targets file we will define the values needed to locate and include the analysis project into every csproj in the solution.

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  
  <PropertyGroup>
    <!-- 
    define the name and location of the lib folder and the code analysis dll. 
    The Huron.CodeAnalysis project will copy to this location when it is built,
    and it is configured to build before other projects.
    -->
    <CodeAnalysisLibName>CodeAnalysis<CodeAnalysisLibName>
    <LibFolder>$(MSBuildThisFileDirectory)../Lib/</LibFolder>
    <CodeAnalysisLib>$(LibFolder)$(CodeAnalsysLibName).dll</CodeAnalysisLib>
  </PropertyGroup>

  <ItemGroup>
    <!-- add the custom analyzer to all projects under this directory, if the .dll exists -->
    <Analyzer Condition="Exists($(CodeAnalysisLib))" Include="$(CodeAnalysisLib)"/>
  </ItemGroup>
</Project>
```

The LibFolder location is defined to be a folder named "Lib" that is one directory level up from where the Directory.Build.targets file is. This can be adjusted to be in a different location if needed. This lib folder should be added to .gitignore to prevent these dlls from being added to the source repository. Since they will be compiled from source, there is no need to include them in the repository.

The ItemGroup/Analyzer element includes this code analysis assembly as an analyzer reference for any project that includes this Directory.Build.targets file, which should be all projects in our solution.

The only remaining issue is build order. Since the code analysis project is not a project dependency, the solution build system doesn't identify it as a dependency when determining build order. What we can do, is to manually configure our solution to make the code analysis assembly a project dependency for every other project that consumes it. This will be part of the .sln configuration.