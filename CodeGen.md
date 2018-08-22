# Domain Specific Languages and Code Generation in .NET

C# is a great, general purpose language. However, as a general purpose language, it can sometimes be tedious and verbose to express simple concepts. This is where we would often like to have a domain specific language (DSL) that can express the concept more succinctly.

There are examples of DSLs in .NET that most developers will already be familiar with, .xaml and .resx files. Both of these DSLs are based on XML, and are intended to support a graphical design surface to define part of the application.

Xaml is typically used to define GUI elements in WPF or UWP applications. Xaml files, even when created using the Visual Studio designer, are fairly human-readable. Readability is important when you need to reach beyond the capabilities of the xaml designer, and when you need to make sense of the file while merging changes.

Resx files are a domain specific language for defining localized strings, and other types of resources. The implementation language is xml, but in reality no one looks at the raw xml. Typically the Visual Studio resource design surface is used, so resx is essentially a "visual DSL". I want to look at resx files, specifically, because I want to discuss some drawbacks to the way that they are implemented.

## Single File Generators

The .resx file uses the "Custom Tool" feature of Visual Studio, which is also known as a "Single File Generator" (SFG): the name of the API that is used to implement them. An SFG consumes an input file to produces a single output file, typically C# source code, to implement a DSL. The output source file gets included in the project, nested underneath the input file in the Solution Explorer. For resx, this is a generated C# source file for accessing the resources, named with a "Designer.cs" suffix.

Since SFGs are implemented as an extension to Visual Studio (VS), they *require* Visual Studio to function. This can be a problem if someone on your team likes to work in a different development environment, VS Code for instance. You cannot generate SFG output from a command line build, they are a design time feature. Since the output file is included in the project and source, it allows anyone to build and consume the resources, but they might be unable to easily edit them. Without the design surface provided by Visual Studio, the raw xml format is not easy to manually edit. Adding basic string resources to an existing resx file isn't too bad, but writing a new file from scratch, or adding a non-string resource is pretty difficult to do manually.

I would argue that the output C# file is not really *source* code; it is *generated* code. The source for the generated code is the .resx xml; that is all that should be checked into the source repository. I don't want generated code checked into my source repository, because the differences in the output should be more clearly expressed by the input DSL. A merge conflict in the resx implies a merge conflict in the generated code, and I like to minimize those conflicts as much as possible. Duplicating information doesn't serve that goal. Having generated code in source code also presents the problem of accidental edits. If the generated code is edited by hand, those edits will be lost the next time the code is generated. Typically, generated code files will have an indicator that they are generated: a .g.cs extension, and a block comment at the top warning that the code should not be edited directly. Occasionally, these warnings can be missed and accidental edits can happen, the result is usually a compilation failure, or worse a runtime exception. Generated code has a tendancy to be verbose, which is usually the reason it is generated, and having to merge that as well adds overhead.

## MSBuild Tasks

Single File Generators are one approach to implementing a DSL. Another approach is to create a custom MSBuild task. An MSBuild task eliminates some of the problems of SFGs, specifically that they don't require Visual Studio. It still requires MSBuild, but anyone using .NET is also likely using MSBuild for their build whether they realize it or not.

Xaml files use MSBuild tasks to implement their DSL. If you look in the "obj" folder for a WPF project, you will see "g.cs" files that are associated with the xaml files in your project. These files don't get included in the project's source repository, which is nice, because they can be quite complex for large xaml files.

It turns out that .resx files also use an MSBuild task to perform part of their work. While the SFG generates the C# code which provides the API for accessing the resources, the actual resource gets embedded in the assembly by an MSBuild task. This generated .resources file doesn't get included in your source code; it is also hidden away within the temporary output (obj) folder when you build. This is good, because it is a binary format, so would be un-diffable and unmergable.

So, why doesn't .resx use an MSBuild task to generate the C# code as well? If it did, then it wouldn't require Visual Studio to work. It could also output the generated code to the temporary output folder so it wouldn't be included in source. The reason for this is Intellisense. For Intellisense to function, the IDE needs to be able to read all of the source files for a project. The IDE doesn't do a full rebuild every time a file is changed. This would be way too slow, and would result in an unpleasant developer experience. By generating output using an SFG, the output is regenerated any time the .resx is modified in Visual Studio, and the output code file is then visible to the IDE and starts showing up in autocomplete lists. Since a full build isn't performed, your custom MSBuild tasks won't have a chance to run to produce their output. No output, no Intellisense.

Xaml files get around this limitation by *also* using an SFG; the XamlIntelliSenseFileGenerator. If you remove this value from the "Custom Tool" property for a xaml file, the intellisense won't be updated until you do a full build. It feels kinda broken when you see red-squigglies that dissapear when you compile.

## CSProj

The final failing of resx files is the verbosity they add to the csproj file. Adding a single resx file will add at least nine lines to your csproj file. This is true for old-style as well as the new "sdk-style" project files. These lines aren't particularly interesting to us, but are required to drive the Visual Studio project system.

```xml
  <ItemGroup>
    <Compile Update="Resource1.Designer.cs">
      <DesignTime>True</DesignTime>
      <AutoGen>True</AutoGen>
      <DependentUpon>Resource1.resx</DependentUpon>
    </Compile>
  </ItemGroup>

  <ItemGroup>
    <EmbeddedResource Update="Resource1.resx">
      <Generator>ResXFileCodeGenerator</Generator>
      <LastGenOutput>Resource1.Designer.cs</LastGenOutput>
    </EmbeddedResource>
  </ItemGroup>
```

## Can we do better?

It turns out, it is now possible to implement a DSL in a more ideal way in dotNET. All of the drawbacks mentioned above can now be remedied. The new approach is MSBuild based, but importantly allows Intellisense to light up for the generated code.

In my next post I will illustrate how to implement a resx-like feature that uses json to define a localizable resource DSL using this new technique.
