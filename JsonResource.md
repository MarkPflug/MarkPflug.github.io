# Building a Better Resources File

In my previous post I discussed some approaches to creating a DSL in .NET. Specifically, I looked at the .resx file as an example, and described some of the issues with how it worked. Let's take a look at how easy it is to create a simple DSL in .NET.

What are we going to build? Well, I've decided I don't like the resx file for reasons I explained before. The xml format for the resx file is too verbose and complex for me to want to author it by hand. I want to create a replacement. I also want to keep the replacement as simple as possible. The format isn't important. If I felt more adventurous I could create my own lexer/parser and define a true custom DSL, but that is beyond the scope of what I want to demonstrate here. So instead, I'm going to use JSON and create a "Json resources file" .resj.

## MSBuild Basics

To understand how this works requires a bit of background in MSBuild. Unfortunatley, I've found that many .NET developers don't really know how MSBuild works. I've also found that most developers that *do* know how it works, wish they didn't, because it probably means they are responsible for managing a build system. If you've ever looked at the source for a csproj file (likely because you are diffing or merging it), then you've looked at an MSBuild script.

An MSBuild script is built from a small number of primitive types: the most important of these primitives are Properties, Items, Task, Targets, and Projects. Properties are roughly global variables that contain simple strings. Items are collections of objects, usually files. Tasks are individual units of work, usually implemented in a .NET assembly. Targets are essentially named functions that invoke tasks. Projects are the files that contain all these primitives, they usually have an extension like: .csproj, .targets, or props. When you invoke a build, you are telling MSBuild to invoke a specific target, usually by default, one named "Build". The standard "Build" target is defined by Microsoft as part of the common .NET MSBuild scripts. You normally get this target in your project by importing Microsoft.CSharp.targets. There are usually many targets in a build script, and these targets have dependencies. The targets are executed in topological order based on dependency. The Build target itself probably doesn't do anything, rather it has a series of other targets that it is dependent on, and these other targets are invoked prior to the Build target being run.

This design allows MSBuild scripts to be very easily extensible. If you want to run code at a specific step in the build process, you can define your own target and specify which targets it should run before and after, without having to edit the files that contain those other target definitions.

## Building the .RESJ DSL

To create our DSL we are going to create an MSBuild task in a C# library, create some MSBuild scripts to invoke our task, and then package these up in a nuget package. By including this nuget package in our project we will be able to use our new resj DSL. 

Create a new csharp class library project. Add nuget package references for Microsoft.Build.Framework and Microsoft.Build.Utilities.Core. These packages define the types we need to create our Task. Of course, we'll also grab Newtonsoft.Json, because what project would be complete without it, oh yeah, and we are going to be parsing JSON.

Our task is going to simultaneously generate C# code to access our resources, and generate the resources binary that will be embedded into our assembly.

Before we start looking at the implementation lets look at an example .resj file:

```JSON
{
  "Strings": {
    "Form_Label_1": "Description",
    "Form_Label_2": "Address"
  }
}
```
## Building the Task

We've simply defined a "Strings" object that maps from a key to a string value. As simple as it gets. I envision there being additional sections other than "Strings" for things like including other kinds of resources like files, or icons, etc. But for this initial example we'll keep it simple.

``` C#
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

Now that we have a MSBuild task defined to do the work, we can author some build scripts to actually wire this task into our build process.