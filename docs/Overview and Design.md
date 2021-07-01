# AutoWeave

## Introduction

This contains my random thoughts and ideas for AutoWeave, which adds metaprogramming capabilities to C# using standard Compiler API idioms and techniques. To be clear, **this is experimental** and I have no intention of making this any more than that.

## Idea

Right now, the Compiler API exposes a number of ways to inspect and modify code (sort of). This includes:
 
* Analyzers - These look for certain conditions in code and allow a developer to specify diagnostics to inform users that these conditions exist.
* Code Fix - These are related to analyzers in that if an analyzer with a specific ID exists, the code fix will create a new syntax tree with new code to address the analyzer.
* Source Generators - These were introduced with C# 9/.NET 5. You can create new code based on conditions in code, though you cannot edit existing code.
 
While these capabilities are extremely powerful, what would take this to the next level is to add some kind of automation to this. For example, let's say you had this method definition:
 
```
public Customer GetCustomer(string firstName, string lastName)
{
  // ...
}
```
 
You can define method parameters as `in` such that you can't change the parameter {TODO: May want to include why this is a "bad" idea, I think it's discouraged via "Framework Design Guidelines"}. You can use members on the parameter, but you can't reassign the parameter value itself. Now, let's say you want **all** parameters to be `in`, like this:
 
```
public Customer GetCustomer(in string firstName, in string lastName)
{
  // ...
}
```
 
This is an example of a coding "standard" that is easy to do, and easy to forget. I *could* write an analyzer that looks for parameters that aren't defined with `in`, and create a related code fix that would add `in` to the definition. But, even though tools like VS will provide an option to apply the existence of a code fix to everything in the solution, I don't even want to know that this happened. I just *want* it to happen.
 
So, AutoWeave is an experimental idea to try and automate this. Maybe I'll build a [.NET tool](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools) called "dotnet tachyon" to start. Maybe at some point I create a "mini" IDE to make it a bit more reactive. The **core** idea is that it would do the following steps:
 
* Take a .sln or .csproj file
* Either call "dotnet build" (or just use the Compiler API to build things directly) to compile the code
* Find all occurences of diagnostics with targeted ID values (these can be created from either source generators or analyzers, see below)
* Run the code fixes on them
* Use those new syntax trees for the resulting compilation

What do I mean by "targeted ID values"? Basically, I only want to automatically run code fixes for specific ones. Either I create an attribute called `AutomaticAttribute` that is only for classes (technically, just for code fix classes, that's an analyzer in itself), and I look for that to autorun the code fix. Or, I have some config file that specifies all the code fixes that will automatically run.

This could be expanded to be kind of an allow/denylist scenario for *any* code fix. Even if a code fix isn't deemed to be `[Automatic]`, it could be configured to automatically run. Conversely, if a code fix is set to *not* autorun, it won't even if it has the attribute. This opens up any code fix to automatically run, and lets the developer stop automatic runs (I could even use wild cards, like "CA1\*", which would match any code fix looking for a diagnostic ID starting with "CA1". Or, put "\*" in the denylist, which would shut everything off).

What about infinite loop conditions? Meaning, I make a compilation, I find a diagnostic with R1234, a code fix exists for that diagnostic that I automatically run. That in turns would fire CA1234. So, do I run the compiler again? What if CA1234 reintroduces R1234? Do I keep going? How many passes would I do? That would start to really affect performance, and how would I stop an infinite loop? I don't have an answer here. Maybe the rule is one pass is done. If any errors still exist, it's just a compilation failure.

There's also the issue of a code fix having the ability of returning `n` code fixes. How would you be able to tell which one you'd want, if you even care? There is an [`EquivalenceKey`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codeactions.codeaction.equivalencekey?view=roslyn-dotnet) property that we might be able to use. For example, if one of the resulting actions has a specific `EquivalenceKey` value, that's the one we pick. This could be configured as well. If the developer doesn't configure anything, we'd automatically apply the first one. Or, maybe we have a `ICodeProviderIdentifier` interface with a `string Id { get; }` property, and have a custom `CodeAction`, but that would be tricky because that's created from a static factory method, `Create()`, which sucks because we'd have to emulate how that's making a `CodeAction`. But that would give us the ability to identify code fixes.

The *ultimate* goal would be that the Compiler API do this directly with support from tools like VS, VS Code, Rider, etc. But that's probably too ambitious. The source generator feature was the result of years of experimentation from Microsoft teams, which at one point did allow code modification, but rejected it because of performance issues and experience problems (see [this issue](https://github.com/dotnet/csharplang/issues/107) for more details). However, this would allow us to get as close to an automated solution as possible.
 
Maybe a related "goal" would be to create a mini-IDE via Blazor so it could be delivered via the browser. We would detect changes in a code editor, and do the diagnostic-to-code-fixes-to-new-syntax-tree dance there. This sounds like a viable alternative. Or, as another angle to experimentation, is to do this using [MAUI](https://github.com/dotnet/maui). Even if I can't use a decent code editor component, just using a multi-line text editor would suffice.
 
## Existing Tools

* [Fody](https://github.com/Fody/Fody)
* [PostSharp](https://www.postsharp.net/) - note that PostSharp recently announced they're moving to a Roslyn-based framework. [Link](https://blog.postsharp.net/post/announcing-caravela-preview.html)

## Code Editors

* [Monaco](https://github.com/microsoft/monaco-editor)
* Try.NET
  * [GitHub](https://github.com/dotnet/try)
  * [Web Site](https://dotnet.microsoft.com/platform/try-dotnet)

## Outcomes

* Create a .NET tool that will perform code weaving based on diagnostics and code fixes
* Use this tool in an experimental small IDE done in Blazor or MAUI (or both?)
* Profit