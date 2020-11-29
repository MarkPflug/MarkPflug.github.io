

writing a cmd line util.

it evolves.
I want to change the entry point. but.... I want to preserve that previous entry point too, it was useful after all.

What do I do.



Insert cursor at top of Main(). Insert '}'. Type 'void OldMain(){'.


Now I have a new entry point 'Main' which does nothing, and my old entry point 'OldMain' is inaccessible from the cmdline. I now need to implement some parsing and dispatching logic between the cmdline and my two methods.

 Okay, no problem, for now I'll just implement that as positional parameters, args[0] and args[1] baby. Switch on args[0] maybe, and done.

 Assess current libraries.

 Propose ideal solution:

 add attribute to "Main" (Program) class. If the attribute is defined, the Program class will subscribe to a new cmdline binder. This cmdline binder will allow for multiple mains, and change the behavior of Main. This binder will have the following properties:

Inside Main will now have a context class, this context class will describe the state between the binder and the bound dispatch and give the user opportunity to intercept the invocation and do something else.
Normally, users would just call context.Invoke() to call whatever the binder had determined. This would trigger the default bound dispatch that the cmdline binder normally provides, but gives the user freedom to do whatever it wanted instead of that. Additionally, Main could accept any number of *bindable* arguments. Bindable meaning they are in a set that can be bound, which is currently underifined, but certainly includes all C# primitives and array of primitives (and 'dynamic' different spec, discuss? would allow "newing" up a new parameter out of whole cloth by just "dotting" off the main arg in a context where the type could be determined)". 

 