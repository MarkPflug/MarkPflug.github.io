# Building a Better Resources File

In my previous post I discussed some approaches to creating a DSL in .NET. Specifically, I looked at the .resx file as an example, and described some of the issues with how it worked. Let's take a look at how easy it is to create a simple DSL in .NET.

What are we going to build? Well, I've decided I don't like the resx file for reasons I explained before. The xml format for the resx file is too verbose and complex for me to want to author it by hand. I want to create a replacement. I also want to keep the replacement as simple as possible. The format isn't important. If I felt more adventurous I could create my own lexer/parser and define a true custom DSL, but that is beyond the scope of what I want to demonstrate here. So instead, I'm going to use JSON and create a "Json resources file" .resj.

## MSBuild Basics

To understand how this works requires a bit of background in MSBuild. Unfortunatley, I've found that many .NET developers don't really know how MSBuild works. I've also found that most developers that *do* know how it works, wish they didn't, because it probably means they are responsible for managing a complex build system. If you've ever looked at the source for a csproj file (likely because you are diffing or merging it), then you've looked at an MSBuild script.

An MSBuild script is just an xml file. It is constructe from a small number of primitive types: the most important of these primitives are: Properties, Items, Task, Targets, and Projects. Properties are roughly global variables that contain primivite values. Items are collections of objects, typically files. Tasks are individual units of work, most often implemented in a .NET assembly. Targets are essentially named functions that invoke tasks. Projects are the files that contain all these primitives, they usually have an extension like: .csproj, .targets, or .props. When you invoke a build, you are telling MSBuild to invoke a specific target, usually by default, one named "Build". The standard "Build" target is defined by Microsoft as part of the common .NET MSBuild scripts. You normally get this target in your project by importing Microsoft.CSharp.targets. There are usually many targets in a build script, and each target might have dependencies. The targets are executed in topological order based on dependency. The Build target itself probably doesn't do anything, rather it has a series of other targets that it is dependent on, and these other targets are invoked prior to the Build target being run.

This design allows MSBuild scripts to be very easily extensible. If you want to run code at a specific step in the build process, you can define your own target in your own project file, specify which targets it should run before and after, without having to edit the files that contain those other target definitions. This allows Microsoft to define the steps required to build the common 99% case, and allows us to customize it on a per-project basis without having to edit the scripts that Microsoft provided.

## Building the .RESJ DSL

To create our DSL we are going to create an MSBuild task in a C# library, create some MSBuild scripts to invoke our task, and then package these up into a nuget package. By including this nuget package in our project we will be able to use our new resj DSL. 

We'll start by creating a new CSharp class library project. We'll need to add Nuget package references for Microsoft.Build.Framework and Microsoft.Build.Utilities.Core. These packages define the types we need to create our Task. Of course, we'll also grab Newtonsoft.Json, because what project would be complete without it. Oh yeah, and we are going to be parsing JSON.

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

The first thing we'll do is create our MSBuild Task. A Task is imply a class that implements the ITask interface. Usually, this is accomplished by deriving from the Task abstract base type, which provides some functionality for us. When we derive from Task the only thing we are responsible for is providing an imlementation of the Execute method. This method takes no parameters and returns a boolean indicating if it was successful or not. If it throws an exception that is also interpreted as failure, unsurprisingly. We pass input to the task, and return output from the task by defining properties on the Task class. For output, we attach the `[Output]` attribute. We can attach the `[Required]` attribute to arguments that are mandatory; MSBuild will present a standard error if they are omitted.

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
        [Required]
        // The folder where we will write all of our generated code.
        public String OutputPath { get; set; }

        // All of the .resj files in our projct.
        public ITaskItem[] InputFiles { get; set; }

        // Will contain all of the generated coded we create
        [Output]
        public ITaskItem[] OutputCode { get; set; }

        // Will contain all of the resources we generate.
        [Output]
        public ITaskItem[] OutputResources { get; set; }
        
        // The method that is called to invoke our task.
        public override bool Execute()
        {
            var outCodeItems = new List<TaskItem>();
            var outResItems = new List<TaskItem>();

            // loop over all the .resj files we were given
            foreach (var iFile in InputFiles)
            {
                // load the Json from the file
                var text = File.ReadAllText(iFile.ItemSpec);
                JObject obj = JObject.Parse(text);

                // prepare the generated files we are about to write.
                var codeFile = Path.Combine(OutputPath, iFile.ItemSpec + ".g.cs");
                var resFile  = Path.Combine(OutputPath, iFile.ItemSpec + ".resources");

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
            // put the artifacts we created in the output properties.
            OutputCode = outCodeItems.ToArray();
            OutputResources = outResItems.ToArray();
            return true;
        }
    }
}
```

This code should be pretty self explanatory. We define inputs and outputs to our task by creating properties on our C# class. We override the Task.Execute method to implement the logic of our task. Inside the Execute method we simply loop over all of the input files and generate some extremely poorly formatted C# code, and use the BCL ResourceWriter to create a .resources file. This gets compiled into a JsonResourceGenerator.dll assembly.

## Invoking the build Task

Now that we have created a simple MSBuild task to do the work, we can author some build scripts to actually wire this task into our build process. Typically, MSBuild scripts like this will be broken into two parts: a .props file that contains property and items definitions, and a .targets file that contains the task and targets definitions. The reason these are broken into two parts is because of how they get imported into the consuming project. The content of MSBuild scripts are processed in the order they are imported, and some definitions need to be included before others to satisfy dependencies.

Let's look at the targets file first. First we'll need a reference to the task we just wrote. We do this with the `UsingTask` element:

`<UsingTask AssemblyFile="JsonResourceBuild.dll" TaskName="JsonResourceGenerator"/>`

This will expose our task so we can invoke it in our target. The target itself will be defined in the same project. The important thing about defining a target to customize a build is figuring out where in the build process it needs to be invoked. For our target, we need to have it run before we call the C# compiler, because we are going to be generating code that we want to include in the compilation. We also need to run after the standard resource generation, because if we had any .resx files in our project we want them to be processed before us; the `ResGen` target is the target responsible for this. With that, we'll define our target, this element will include the tasks.

`<Target Name="GenerateJsonResource" AfterTargets="ResGen" BeforeTargets="CoreCompile">`

Our task required two inputs: a property containing the path to the directory where we'll write our output, and the items list containing the .resj files that we'll process. We also want to capture the output of our task.

``` xml
<JsonResourceGenerator
    OutputPath="$(CodeGenerationRoot)"
    InputFiles="@(JsonResource)"
>
    <Output TaskParameter="OutputCode" ItemName="ResourceCodeFile"/>
    <Output TaskParameter="OutputRes"  ItemName="ResourceResFile"/>
</JsonResourceGenerator>
```

The `CodeGenerationRoot` property needs to be defined. We want all of our output files to be generated in a scratch location, so that it doesn't get included in source control. The standard build scripts include such a location, the "obj" folder. The path to this folder is defined as a path relative to the project folder, using the `IntermediateOutputPath` property. We can define this property inside of our target with this code:

``` xml
<PropertyGroup>
  <CodeGenerationRoot>$(MSBuildProjectDirectory)\$(IntermediateOutputPath)</CodeGenerationRoot>
</PropertyGroup>
```

This is enough to cause our task to be executed. However, we haven't told it anything about the `JsonResource` items, the .resj files that we want to process. Additionally, we need to make sure the C# files and resources that we generated are passed to the compiler. We pass the content to the compiler by including the files in the item groups that the compilation task processed. These item groups are `Compile` and `_CoreCompileResourceInputs`, these are defined in the standard `Microsoft.CSharp.Common.targets` MSBuild scripts.

``` xml
<ItemGroup>
    <Compile Include="@(ResourceCodeFile)"/>
    <_CoreCompileResourceInputs Include="@(ResourceResFile)"/>
</ItemGroup>
```
That is everything that needs to be defined in the .targets file. In the .props file, we will define the items that will be processed as Json resources.

``` xml
<ItemGroup>
    <JsonResource Include="**\*.resj"/>
</ItemGroup>
```

This will recursively include all .resj files that exist under our project folder. Note, that using a globbing path like this, means that .resj files will be processed, even if they aren't explicitly listed in our .csproj file. This is the "new normal" with the addition of the SDK feature in MSBuild. New style project files don't list all of the files explicitly anymore, but instead use globbing paths like this. This means that if you use an old style project, and exclude a .resj file that it will still be processesed. You would need to delete it from the file system, just something to be aware of.

These are all the moving parts that we need to make this work in our build. However, the key piece of magic that I haven't discussed yet is how we get this to integrate with Intellisense. If we only include the pieces defined above, our build will produce the output that we expect, however we won't get Intellisense for the code that we are generating to expose our resources. This is a key feature that we get from .resx files, and we want to provide parity with that. 

Up until recently, I didn't think this was even possible without using a SingleFileGenerator to integrate into Visual Studio. Then I stumbled upon a project doing something similar to what I wanted, [CodeGeneration.Roslyn](https://github.com/AArnott/CodeGeneration.Roslyn). This project allows you to write code generators that get attached to your existing C# code by adding attributes to your types that enables a pre-compilation rewriting. The thing that caught my eye was in the introductory paragraph: "generating new code that shows up to Intellisense as soon as the file is saved to disk". I poked around in the source for this project until I found the puzzle piece that enables this, in the props file for his project he includes the following metadata definition: `<Generator>MSBuild:GenerateCodeFromAttributes</Generator>`. This `Generator` metadata is something I've seen before, a it is also used to associate single file generators (custom tool). However, I'd never seen this magic prefix "MSBuild:" which appears to associate an MSBuild target to do the work.

With this new found knowledge I set out to apply it to my json resource generator.

``` xml
  <ItemDefinitionGroup>
    <JsonResource>
      <Generator>MSBuild:GenerateJsonResourceCode</Generator>
    </JsonResource>
  </ItemDefinitionGroup>
  ```

This says, add this Generator metadata to all items in the JsonResource item group, such that any time a .resj file is saved run the code generator. I thought this would just work as it was the same pattern I'd seen in this other project, sadly it didn't

## The Hack

What project is complete without a hacky workaround? Dismayed that the `Generator` metadata didn't work like I expected, I poked and prodded to see if I could figure out how to trigger it. I discovered that if I applied the metadata to the `Compile` item group, that any time I modified a .cs file (which are included in the Compile group), it would regenerate the code. It still wouldn't regenerate when I modified a .resj file though. So, what if I include all the .resj files in the `Compile` group. Voila, this worked. However, I don't want to actually compile the .resj files, since they aren't C# code, they clearly can't be compiled. I just need them to bein this item group to trigger the code generator in VS.

The hack is to include them in the `Compile` item group, then after we generate the code remove them from `Compile` before we actually invoke the C# compile task. Our complete props and targets files end up looking like this:

### JsonResourceGenerator.props
``` xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemDefinitionGroup>
    <Compile>
      <Generator>MSBuild:GenerateJsonResourceCode</Generator>
    </Compile>
    <JsonResource>
      <Generator>MSBuild:GenerateJsonResourceCode</Generator>
    </JsonResource>
  </ItemDefinitionGroup>
  
  <ItemGroup>
    <JsonResource Include="**\*.resj"/>
    <Compile Include="@(JsonResource)"/>
  </ItemGroup>
</Project>
```

### JsonResourceGenerator.targets
``` xml
<?xml version="1.0" encoding="utf-8" ?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <UsingTask AssemblyFile="JsonResourceBuild.dll" TaskName="JsonResourceGenerator"/>
  <Target Name="GenerateJsonResource" AfterTargets="ResGen" BeforeTargets="CoreCompile">
    <PropertyGroup>
      <CodeGenerationRoot>$(MSBuildProjectDirectory)\$(IntermediateOutputPath)</CodeGenerationRoot>
    </PropertyGroup>

    <JsonResourceGenerator
      OutputPath="$(CodeGenerationRoot)"
      InputFiles="@(JsonResource)"
    >
      <Output TaskParameter="OutputCode" ItemName="ResourceCodeFile"/>
      <Output TaskParameter="OutputRes"  ItemName="ResourceResFile"/>
    </JsonResourceGenerator>
    
    <ItemGroup>
      <Compile Include="@(ResourceCodeFile)"/>
      <_CoreCompileResourceInputs Include="@(ResourceResFile)"/>
      <Compile Remove="@(JsonResource)"/>
    </ItemGroup>
    
  </Target>
</Project>
```

I don't see any reason why this Generator metadata shouldn't work on user-defined Item types, but it currently doesn't appear to. Hopefully at some point in the future VS will be updated to support it, but until then I've got to live with this hack, oh well.

## Packaging with Nuget

Nuget includes the ability to provide MSBuild scripts in your package. This is accomplished by including your scripts in a folder called "Build", and following a naming convention that allows the scripts to be handled automatically. The naming convention is [PackageName].(props|targets). The .props file will be automatically included at the top of the consuming project, and the .targets will be included at the end. Of course, if you have custom tasks, the assembly that implements them also needs to be included. This is done by providing explicit file defintions in a .nuspec file:

``` xml
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
    <metadata>
        <id>JsonResourceGenerator</id>
        <version>0.1.0</version>
        <description>Json Resource Generator</description>
        <authors>Mark Pflug</authors>
    </metadata>
  <files>
	  <file src="build\**" target="build" />
      <file src="bin\Debug\net462\CodeGenerator.dll" target="build"/>
      <file src="bin\Debug\net462\Newtonsoft.Json.dll" target="build"/>
	</files>
</package>
```

## Conclusion

I've used this .resj Json resource DSL as an example of how to create an "ideal DSL". The benefits over using .resx is that the .resj file is human-read/writable and the generated code doesn't need to be included in source control. It does this while continuing to integrate with Intellisense in Visual Studio.

I really want to highlight that the important takeaway is that this demonstrates a technique for creating a DSL that can be delivered via nuget. The Json Resource Generator is just a simple example of what is possible.

At work, I have implemented a code generator that currently generates over 50k lines of code to define a data access layer. I implemented it as an SFG in a Visual Studio Integrated eXtension (VSIX). This has presented problems, because it requires that a developer install the VSIX, the 50k lines of code live in source control, and any time I want to modify the code-gen it requires that they update the VSIX and regenerate all of the code. Of course, this also means that Visual Studio is required for the code generator to even function. If I reimplement this using this new MSBuild DSL technique, all of these issues go away: 50k fewer lines in source control, can use any delopment environment that they want, and all a developer would need to do is update the nupkg and rebuild the solution. The only thing that would get modified in source, would be the version of the nupkg in the project file. This is the DSL utopia that I've been hoping for, and I can realize it *today*!

If you are using any code generation techniques, hopefully this has given you the tools to make it cleaner and easier to manage.

