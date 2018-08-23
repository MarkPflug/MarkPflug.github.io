---
layout: post
title:  "Domain Specific Language in dotNET"
date:   2018-08-22 17:23:17 -0700
---

# Domain Specific Language in dotNET

In my previous post I discussed some approaches to creating a DSL in .NET. Specifically, I looked at the .resx file as an example, and described some of the issues with how they are implemented. In this article I want to take a look at a better way to implement DSLs in dotnet. Since I've decided I don't like the resx file for reasons I explained in the previous post, I'd like to take a look at implementing a replacement. If I was adventurous I could create my own domain specific language with a lexer and a parser and implement some code to transform it into C#, but that is beyond the scope of what I want to demonstrate here. So instead, I'm going to use JSON and create a "Json resources file" .resj. What the DSL is used for, and the source language of the DSL aren't the important point. The important point is the technique for integrating it in MSBuild and Visual Studio. In this way, we can apply this technique to any DSL.

## MSBuild Basics

To understand how this works requires a bit of background in MSBuild. Unfortunately, I've found that most .NET developers don't really know how MSBuild works. I think this is a testament to how good the integration with Visual Studio is. MSBuild is used behind the scenes when you build in VS, but you rarely have to spend much time thinking about it, unless you need to so something non-standard. If you have ever looked inside a .csproj file (or .vbproj if you're one of *those* people), then you've seen an MSBuild script. That's all the csproj file is, an MSBuild script.

An MSBuild script is just an xml file. It is constructed from a small number of primitive elements. The most important of these primitives are: Properties, Items, Task, Targets, and Projects. Properties are roughly global variables that contain primivite values. Items are collections of objects, typically files. Tasks are individual units of work, most often implemented in a .NET assembly. Targets are essentially named functions that invoke tasks. Projects are the containers that contain all these primitives, they usually have an extension like: .csproj, .targets, or .props. When you invoke a build, you are telling MSBuild to invoke a specific target, usually by default the one named "Build". The standard "Build" target is defined by Microsoft as part of the common .NET MSBuild scripts. You normally get this target in your project by importing Microsoft.CSharp.targets. There are usually many targets in a build script, and each target might have dependencies. The targets are executed in topological order based on dependency. The Build target itself probably might not do anything itself, rather it has a series of other targets that it is dependent upon, and these other targets are invoked prior to the Build target.

This design allows MSBuild scripts to be very easily extensible. If you want to run code at a specific step in the build process, you can define your own target in your own project file, specify which targets it should run before and after, without having to edit the files that contain those other target definitions. This allows Microsoft to define the steps required to build the common 99% case, and allows us to customize it on a per-project basis without having to edit the scripts that Microsoft provided.

## Building the .RESJ DSL

To create our DSL we are going to create an MSBuild task in a C# library, create some MSBuild scripts to invoke our task, and then package these up into a nuget package. By including this nuget package in our project we will be able to use our new resj DSL. 

We'll start by creating a new CSharp class library project. We'll need to add Nuget package references for Microsoft.Build.Framework and Microsoft.Build.Utilities.Core. These packages define the types we need to create our Task. We'll also grab Newtonsoft.Json, because we'll be implementing our DSL as JSON.

Our task is going to simultaneously generate C# code to access our resources, and generate the resources binary that will be embedded into our assembly.

Before we start looking at the implementation lets look at an example .resj file:

```JSON
{
  "Strings": {
    "FormLabelDescription": "Description",
    "FormLabelAddress": "Address"
  }
}
```

In this example, we are defining two string resources that will be used for labels on a form. I'm going to keep this example super-simple and just deal with string resources. Of course, a complete implementation should support file resources, icons, images, etc. 

## Building the Task

The first thing we'll do is create our MSBuild Task. A Task is imply a class that implements the ITask interface. Usually, this is accomplished by deriving from the Task abstract base type, which provides some base functionality for us. When we derive from Task the only thing we are responsible for is providing an implementation of the Execute method. This method takes no parameters and returns a boolean indicating it was successful or not. If it throws an exception that is also, unsurprisingly, interpreted as failure. We pass input to the task, and return output from the task by defining properties on the Task derived class. For output, we attach the `[Output]` attribute. We can attach the `[Required]` attribute to parameters that are mandatory; MSBuild will then present a standard error if those parameters are omitted.

For our task, we need two inputs: the path to the folder where we will generate our output files, and the list of resj files that we are going to process. These will be the `OutputPath` and `InputFiles` properties. When a property will be handling a collection of files, the property type should be `ITaskItem[]`. Our task will return two values: the list of C# code files that were generated and need to included in the compilation, and the list of resource files that were generated and need to be embedded in our assembly.

The resource files will be creating using the `ResourceWriter` from the .NET base class library. We will generate C# in a very rudimetary way, using a StreamWriter. For this demonstration, the generated code is going to be ugly-formatted, to keep the code simple. Remember, the generated code won't be included in our source repository, so it doesn't matter what it looks like anyway.

``` CSharp
using Microsoft.Build.Framework;
using Microsoft.Build.Utilities;
using Newtonsoft.Json.Linq;
using System;
using System.Collections.Generic;
using System.IO;

namespace JsonResource
{
    public class JsonResourceGenerator : Task
    {
        // The folder where we will write all of our generated code.
        [Required]
        public String OutputPath { get; set; }

        // All of the .resj files in our projct.
        public ITaskItem[] InputFiles { get; set; }

        // Will return all of the code files generated
        [Output]
        public ITaskItem[] OutputCode { get; set; }

        // Will return all of the resources generated.
        [Output]
        public ITaskItem[] OutputResources { get; set; }
        
        // The method that is called to invoke our task.
        public override bool Execute()
        {
            // collections to gather our output
            var outCodeItems = new List<TaskItem>();
            var outResItems = new List<TaskItem>();

            // loop over all the .resj files we were given
            foreach (var iFile in InputFiles)
            {
                // load the Json from the file
                var text = File.ReadAllText(iFile.ItemSpec);
                JObject obj = JObject.Parse(text);

                // define the generated files we are about to write.
                var codeFile = Path.Combine(OutputPath, iFile.ItemSpec + ".g.cs");
                var resFile  = Path.Combine(OutputPath, iFile.ItemSpec + ".resources");

                // add them to the output collections
                outCodeItems.Add(new TaskItem(codeFile));
                outResItems .Add(new TaskItem(resFile));

                Directory.CreateDirectory(Path.GetDirectoryName(codeFile));
                string resName = Path.GetFileName(iFile.ItemSpec);
                string className = Path.GetFileNameWithoutExtension(iFile.ItemSpec);

                // write the generated C# code and resources file.
                using (var w = new StreamWriter(codeFile))
                using (var rw = new System.Resources.ResourceWriter(resFile))
                {
                    // very simplistic resource accessor class mostly duplicated from resx output.
                    w.WriteLine("public static partial class " + className + " { ");
                    w.WriteLine("static global::System.Resources.ResourceManager rm;");
                    w.WriteLine("static global::System.Globalization.CultureInfo resourceCulture;");
                    w.WriteLine("static global::System.Resources.ResourceManager ResourceManager {");
                    w.WriteLine("get {");
                    w.WriteLine("if(object.ReferenceEquals(rm, null)) {");
                    w.WriteLine("rm = new global::System.Resources.ResourceManager(\"" + resName + "\", typeof(" + className + ").Assembly);");
                    w.WriteLine("}");
                    w.WriteLine("return rm;");
                    w.WriteLine("}");
                    w.WriteLine("}");
                    w.WriteLine("static global::System.Globalization.CultureInfo Culture {");
                    w.WriteLine("get {");
                    w.WriteLine("return resourceCulture;");
                    w.WriteLine("}");
                    w.WriteLine("set {");
                    w.WriteLine("resourceCulture = value;");
                    w.WriteLine("}");
                    w.WriteLine("}");

                    // loop over all the strings in our resj file.
                    foreach (var kvp in (JObject)obj["Strings"])
                    {
                        var key = kvp.Key;
                        var value = (JValue)kvp.Value;

                        // write our string to the resources binary
                        rw.AddResource(key, (string)value.Value);

                        // generate a C# property to access the string by name.
                        w.WriteLine("public static string " + key + " {");
                        w.WriteLine("get {");
                        w.WriteLine("return ResourceManager.GetString(\"" + key + "\", resourceCulture);");
                        w.WriteLine("}");
                        w.WriteLine("}");
                    }

                    w.WriteLine("}");
                }
            }
            // put the results we created in the output properties.
            OutputCode = outCodeItems.ToArray();
            OutputResources = outResItems.ToArray();
            // success!
            return true;
        }
    }
}
```

This code should be pretty self explanatory. We are simply looping over the input resj files that were passed to us, and writing a C# code file, and resource file for each one. The structure of the generated code mimics the code that the resx code generator produces. I'm also omitting any error handling in this code for simplicity. It would be good to provide some amount of validation for the formatting of the json, for example.

## Invoking the build Task

Now that we have created the task, we need to create the script to actually wire this task into our build process. Typically, MSBuild scripts like this will be broken into two parts: a .props file that contains property and items definitions, and a .targets file that contains the task and targets definitions. The reason these are broken into two parts is because of how they get imported into the consuming project. The content of MSBuild scripts are processed in the order they are imported, and some definitions need to be included before others to satisfy dependencies.

Let's look at the targets file first, since it will dictate what we need to define in our props file. First we'll need a reference to the task we just wrote. We do this with the `UsingTask` element:

`<UsingTask AssemblyFile="JsonResourceBuild.dll" TaskName="JsonResourceGenerator"/>`

This will expose our task so we can invoke it in our target. The target itself will be defined in the same file. The important thing about defining a target is figuring out where in the build process it needs to be invoked. For our target, we need to have it run before we call the C# compiler, because we are going to be generating code that we want to include in the compilation. The target that invokes the compiler is named "CoreCompile". I'm not sure if there is any documentation around the standard build targets, I've always figured this stuff out by simply looking at the build scripts that ship with MSBuild, and looking at build logs with diagnostic tracing enabled.

I also determined that it would be best to run after the standard resource target, `ResGen` is invoked. This target is responsible for producing a .resources file from a .resx input. The .resources file is the binary image that actually gets embedded in the assembly. We want to run after it runs, because of a detail in how it works that we want to stay out of the way of.
 
We can use the `AfterTargets` and `BeforeTargets` attributes to indicate to MSBuild where we want our target to be invoked in the build process. The name of the target serves no purpose other than allowing other targets to refer to it, and for logging output.

`<Target Name="GenerateJsonResource" AfterTargets="ResGen" BeforeTargets="CoreCompile">`

The task we created required two inputs: a property containing the path to the directory where we'll write our output, and the items list containing the .resj files that we'll process. We also want to capture the output of our task. The `$()` syntax is a Property reference, and the `@()` is an ItemGroup reference. These references will be defined in our .props file later. The output for our Task is defined using the Output element, which requires the name of the task property to capture and the name of the item to put the output into.

``` xml
<JsonResourceGenerator
    OutputPath="$(CodeGenerationRoot)"
    InputFiles="@(JsonResource)"
>
    <Output TaskParameter="OutputCode" ItemName="ResourceCodeFile"/>
    <Output TaskParameter="OutputRes"  ItemName="ResourceResFile"/>
</JsonResourceGenerator>
```

We haven't defined what these input properties contain yet. For the `CodeGenerationRoot` we want to use the "obj" folder, which is essentially a scratch area for temporary build files. By putting the files here we can ensure that they don't get added to source control, which was one of our goals. The path to this folder is defined as a path relative to the project folder, using the `IntermediateOutputPath` property. We can define this property inside of our target with this code:

``` xml
<PropertyGroup>
  <CodeGenerationRoot>$(IntermediateOutputPath)</CodeGenerationRoot>
</PropertyGroup>
```

Typically, properties would be defined in a .props file, but we need to put this in the .targets file, because the `IntermediateOutputPath` won't be defined at the point that our .props file is imported; it is defined by the standard MSBuild scripts that get imported later.

Our task will create some .cs and .resources files that we need to hand off to the C# compiler. We do this by adding them to the `Compile` and `_CoreCompileResourceInputs`. The `Compile` item group should be familiar, as it is how all C# source files are included in a .csproj file. The `_CoreCompileResourceInputs` is an item group defined by Microsoft. The leading underscore makes me concerned that I shouldn't be using it directly, but it was the only way I found to include my own resources. This `<ItemGroup>` will be nested inside our Target, after the task is invoked.

``` xml
<ItemGroup>
    <Compile Include="@(ResourceCodeFile)"/>
    <_CoreCompileResourceInputs Include="@(ResourceResFile)"/>
</ItemGroup>
```
That is everything that is required in the .targets file: invoking our task and passing the output to the compiler. All that is left is to define the inputs to our task, the .resj files. We do this in the .props file, by adding an `<ItemGroup>` element. The name of the element, `JsonResource` is the name we use to refer to this item group. The `Include` attribute value, `**\*.resj`, is a globbing path that will include all .resj files recursively in our project folder. By default all files (**/*.*) are included in the `None` ItemGroup, excluding files that are added to a different ItemGroup, like `Compile`. We'll do the same, and exclude our `JsonResource` items from the `None` group.

``` xml
<ItemGroup>
    <JsonResource Include="**/*.resj"/>
    <None Remove="**/*.resj"/>
</ItemGroup>
```

This is the new, standard convention with the SDK-style project system. SDK style project files don't list all of the files explicitly anymore, but instead use default globbing paths. This means that the csproj file doesn't need to change as often, and so less opportunity for merge conflicts.

These are all the moving parts that we need to make this work in our build. However, the key piece of magic that I haven't discussed yet is how we get this to integrate with Intellisense. If we only include the pieces defined above, our build will produce the output that we expect, however we won't get Intellisense for the code that we are generating to expose our resources. This is a key feature that we get from .resx files, and we should expect that with the .resj format as well. 

## Enabling Intellisense

Up until recently, I didn't think this was even possible without creating a CustomTool to integrate into Visual Studio. Implementing a CustomTool, is much more involved than implementing MSBuild scripts, and also creates a dependency on Visual Studio, which would be good to avoid. I recently stumbled upon this project that was claiming to do exactly what I wanted: [CodeGeneration.Roslyn](https://github.com/AArnott/CodeGeneration.Roslyn). This project allows you to write code generators that get attached to your existing C# code by adding attributes to your types that enables a pre-compilation rewrite of the syntax tree. The thing that caught my eye was in the introductory paragraph: "generating new code that shows up to Intellisense as soon as the file is saved to disk". I poked around in the source for this project until I found the magic puzzle piece that enables this. The props file for that project includes the following metadata definition: `<Generator>MSBuild:GenerateCodeFromAttributes</Generator>`. This `Generator` metadata is something I've seen before, as it is also used to associate CustomTools (SFG) with files. However, I'd never seen this magic prefix "MSBuild:" which appears to associate an MSBuild target to do the work. This seems to be a new feature added to support the SDK-style project system, and does not work with old-style projects.

With this new-found knowledge I set out to apply it to my json resource generator.

``` xml
  <ItemDefinitionGroup>
    <JsonResource>
      <Generator>MSBuild:GenerateJsonResourceCode</Generator>
    </JsonResource>
  </ItemDefinitionGroup>
  ```

This says: add this Generator metadata to all items in the JsonResource item group, such that any time a .resj file is saved run the `GenerateJsonResourceCode` target. I thought this would "*just work*", as it was the same pattern I'd seen in this other project. Sadly, it did not. Turns out, there is some additional "goo" required to inform Visual Studio about the new item type.

With [help from some folks at Microsoft](https://github.com/dotnet/project-system/issues/2853) I learned what was needed to inform Visual Studio of our new item type. We must provide a ContentType definition for the JsonResource item type. This is provided in an xml (xaml) file (Elemental.JsonResource.ContentType.xaml), which will sit adjacent to the targets file:

```xml
<ProjectSchemaDefinitions 
    xmlns="http://schemas.microsoft.com/build/2009/properties">

    <ContentType
        Name="JsonResource" 
        DisplayName="Json resource file" 
        ItemType="JsonResource" />

    <ItemType 
        Name="JsonResource" 
        DisplayName="Json Resource File"/>

    <FileExtension 
        Name=".resj" 
        ContentType="JsonResource" />

</ProjectSchemaDefinitions>
```

Ths content type definition is then referenced from the `Elemental.JsonResource.targets` file by adding the following 

```xml
	<ItemGroup>		
		<PropertyPageSchema Include="$(MSBuildThisFileDirectory)Elemental.JsonResource.ContentType.xaml" />
	</ItemGroup>
```

It isn't totally clear to me *why* this is needed, but it is critical to making Intellisense work for the DSL generated code within Visual Studio.

## Packaging with NuGet

Now that we have fully implemented the build script, we want to package it up to be easily reusable. We'll do this by creating a NuGet package.

NuGet includes the ability to provide MSBuild scripts in your package. This is accomplished by including your scripts in a folder called "build", and following a naming convention that allows the scripts to be handled automatically. The naming convention is [PackageName].(props/targets). The .props file will be automatically included at the top of the consuming project, and the .targets will be included at the end. Of course, if you have custom tasks, the assembly that implement the tasks must also be included in the build folder so it can be referenced. You can read more about MSBuild nuget integration in the [nuget documentation](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package#including-msbuild-props-and-targets-in-a-package).

Creating a NuGet package used to involve creating a `.nuspec` file that that would instruct nuget.exe how to assemble the package. This has been improved recently, and incorporated into the SDK-style project system. Now we can define all of the package details within the csproj file instead. Microsoft has provided [fairly detailed documentation](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj#nuget-metadata-properties) about the new MSBuild properties to control this.

The new project system will produce a NuGet package without having to specify anything at all, but by default the output is somewhat incomplete. The default configuration will include the assembly in the package "lib" folder which will include our assembly as a reference in the consuming project. We don't want that. The consuming project shouldn't reference our assembly, we want to modify the build process but not be consumed at runtime. To prevent our assembly from being referenced, we set the `IncludeBuildOutput` property to `false`. However, once we do that, we are entirely responsible for the file contents of our package. This was one detail that wasn't clearly spelled out in the documentation. This is accomplished by adding items to the `Content` item group and defining the `PackagePath` metadata to specify where the file should be packaged. There are a number of other properties that affect the packaging process, but I'm only going to include the `Authors` and `Description` values for this example.

The entire csproj for my project ends up looking like the following.

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net461</TargetFramework>
    <AssemblyName>JsonResourceBuild</AssemblyName>
    <Authors>Mark Pflug</Authors>
    <Description>Json Resource DSL</Description>
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>
  <ItemGroup>
    <Content Include="build\*" PackagePath="build\" />
    <Content Include="$(TargetPath)" PackagePath="build\" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Build.Framework" Version="15.3.409" />
    <PackageReference Include="Microsoft.Build.Utilities.Core" Version="15.3.409" />
    <PackageReference Include="Newtonsoft.Json" Version="10.0.3" />
  </ItemGroup>
</Project>
```

My props and targets files are in a build folder in my project, so I specify that they should be copied to the build folder of the package. I also copy the `$(TargetPath)` item which is the path the compiled assembly for our project. I can then invoke `dotnet pack` from the command line to cause my package to be built.

As a tip, to test nuget packages before you publish them, you can define a local package source that is simply pointed to a directory location on your file system. You can use that source to test installation of your package. Note also, that you will need to delete your package from the local package cache in order to refresh. The local package cache is at `%userprofile%\.nuget\packages`. Simply delete the folder for you package from that folder to cause it to be pulled from your local repository instead.

## Conclusion

I've used this .resj Json resource DSL as an example of how to create what I consider an "ideal DSL". The benefits over using .resx is that the .resj file is human-read/writable and the generated code doesn't need to be included in source control. The resj file also doesn't require modifying the .csproj file, while .resx still do, even in the new SDK project system. It does this while continuing to integrate with Intellisense in Visual Studio.

I really want to highlight that the important takeaway isn't this json resource specifically, but the technique for creating a DSL that can be delivered via nuget.  The Json resource DSL is just a simple demonstration of what is possible.

For my employer, I have implemented a CustomTool code generator that currently generates over 50k lines of code to define a data access layer. I implemented it as a CustomTool (SingleFileGenerator) in a Visual Studio Integrated eXtension (VSIX). I did it this way, because exposing the data model to Intellisense was a primary goal, and this was the only way to do it at the time. Unfortunately, this has presented problems, because it requires that a developer install the VSIX, the 50k lines of code live in source control, and any time I want to modify the code-generation process it requires that they update the VSIX and regenerate all of the code; which is a tedious set of steps. Of course, this also means that Visual Studio is required for the code generator to even function. If I were to reimplement this using this new MSBuild DSL technique, all of these issues go away: 50k fewer lines in source control, consumers can use any delopment environment that they want, and all a developer would need to do is update the nupkg and rebuild the solution to update to a new generator. The only thing that would get modified in source, would be the version of the nupkg in the project file. This is the DSL utopia that I've been hoping for, and I can finally realize it *today*!

If you are using any code generation techniques, hopefully this has illustrated a technique you can use to make it cleaner and easier to manage.
