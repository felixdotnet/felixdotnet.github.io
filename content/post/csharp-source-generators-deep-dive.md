---
title: "C# Source Generators: Generating Code at Compile Time Without Losing Your Sanity"
date: 2025-06-20T11:20:00+01:00
draft: false
categories:
  - .NET
tags:
  - C#
  - .NET
  - Compilers
  - Advanced
  - Performance
description: "A deep dive into C# source generators, the compile-time code generation feature that's more powerful than you think."
---

I'll be honest: when source generators were first announced for C# 9, I thought they were a solution looking for a problem. Then I spent a week manually writing boilerplate code for a JSON API, and I realized I was the problem. Source generators aren't just a fancy feature-they're a game-changer for eliminating repetitive code while maintaining performance.

## What Are Source Generators, Really?

Source generators are compile-time code generators that run during compilation and can inspect your code to generate additional C# source files. Unlike runtime code generation (like `System.Reflection.Emit`), source generators produce code that's compiled normally, meaning you get full IntelliSense, debugging, and performance.

The key insight is that source generators run *during* compilation, not before or after. They're part of the compilation pipeline, which means they can see your code's syntax tree and semantic model, but they can't modify existing code-only add new files.

## The Anatomy of a Source Generator

A source generator is a class that implements `ISourceGenerator` and is decorated with `[Generator]`:

```csharp
[Generator]
public class MySourceGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        // Register syntax receivers or other initialization
    }

    public void Execute(GeneratorExecutionContext context)
    {
        // Generate code here
    }
}
```

The `Initialize` method runs once per generator and is where you set up syntax receivers. The `Execute` method runs for each compilation and is where you actually generate code.

## Syntax Receivers: The Secret Sauce

Syntax receivers let you filter the syntax tree before the generator runs. This is crucial for performance-you don't want to process every single syntax node in your solution.

```csharp
class MySyntaxReceiver : ISyntaxReceiver
{
    public List<ClassDeclarationSyntax> Classes { get; } = new();

    public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
    {
        if (syntaxNode is ClassDeclarationSyntax classDecl &&
            classDecl.AttributeLists.Count > 0)
        {
            Classes.Add(classDecl);
        }
    }
}
```

Then register it in `Initialize`:

```csharp
public void Initialize(GeneratorInitializationContext context)
{
    context.RegisterForSyntaxNotifications(() => new MySyntaxReceiver());
}
```

But here's the gotcha: syntax receivers run on *every* syntax node, so keep them fast. Don't do heavy semantic analysis here-just collect nodes that look interesting.

## A Real Example: Generating API Clients

Let's build something useful: a source generator that creates HTTP client code from interface definitions. This is the kind of boilerplate that source generators excel at eliminating.

```csharp
// User writes this:
[ApiClient("https://api.example.com")]
public interface IUserService
{
    [HttpGet("/users/{id}")]
    Task<User> GetUserAsync(int id);

    [HttpPost("/users")]
    Task<User> CreateUserAsync([FromBody] User user);
}

// Generator produces this (conceptually):
public class UserServiceClient : IUserService
{
    private readonly HttpClient _httpClient;
    
    public UserServiceClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<User> GetUserAsync(int id)
    {
        var response = await _httpClient.GetAsync($"/users/{id}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<User>();
    }
    
    // ... etc
}
```

The generator implementation:

```csharp
[Generator]
public class ApiClientGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForSyntaxNotifications(() => new ApiClientSyntaxReceiver());
    }

    public void Execute(GeneratorExecutionContext context)
    {
        if (!(context.SyntaxReceiver is ApiClientSyntaxReceiver receiver))
            return;

        var compilation = context.Compilation;
        
        foreach (var interfaceDecl in receiver.Interfaces)
        {
            var semanticModel = compilation.GetSemanticModel(interfaceDecl.SyntaxTree);
            var symbol = semanticModel.GetDeclaredSymbol(interfaceDecl) as INamedTypeSymbol;
            
            if (symbol == null || !HasApiClientAttribute(symbol))
                continue;

            var source = GenerateClientClass(symbol, context);
            context.AddSource($"{symbol.Name}Client.g.cs", source);
        }
    }

    private bool HasApiClientAttribute(INamedTypeSymbol symbol)
    {
        return symbol.GetAttributes()
            .Any(attr => attr.AttributeClass?.Name == "ApiClientAttribute");
    }

    private string GenerateClientClass(INamedTypeSymbol interfaceSymbol, GeneratorExecutionContext context)
    {
        var sb = new StringBuilder();
        var ns = interfaceSymbol.ContainingNamespace.ToDisplayString();
        var className = $"{interfaceSymbol.Name}Client";
        var baseUrl = GetBaseUrl(interfaceSymbol);

        sb.AppendLine("using System.Net.Http;");
        sb.AppendLine("using System.Net.Http.Json;");
        sb.AppendLine("using System.Threading.Tasks;");
        sb.AppendLine();
        sb.AppendLine($"namespace {ns}");
        sb.AppendLine("{");
        sb.AppendLine($"    public class {className} : {interfaceSymbol.Name}");
        sb.AppendLine("    {");
        sb.AppendLine("        private readonly HttpClient _httpClient;");
        sb.AppendLine();
        sb.AppendLine($"        public {className}(HttpClient httpClient)");
        sb.AppendLine("        {");
        sb.AppendLine("            _httpClient = httpClient;");
        sb.AppendLine($"            _httpClient.BaseAddress = new System.Uri(\"{baseUrl}\");");
        sb.AppendLine("        }");
        sb.AppendLine();

        foreach (var method in interfaceSymbol.GetMembers().OfType<IMethodSymbol>())
        {
            GenerateMethod(sb, method);
        }

        sb.AppendLine("    }");
        sb.AppendLine("}");

        return sb.ToString();
    }

    private void GenerateMethod(StringBuilder sb, IMethodSymbol method)
    {
        // Extract HTTP method and route from attributes
        var httpMethod = GetHttpMethod(method);
        var route = GetRoute(method);
        
        var returnType = method.ReturnType.ToDisplayString();
        var methodName = method.Name;
        var parameters = string.Join(", ", method.Parameters.Select(p => 
            $"{p.Type.ToDisplayString()} {p.Name}"));

        sb.AppendLine($"        public async {returnType} {methodName}({parameters})");
        sb.AppendLine("        {");
        
        // Generate method body based on HTTP method
        if (httpMethod == "GET")
        {
            sb.AppendLine($"            var response = await _httpClient.GetAsync($\"{route}\");");
        }
        else if (httpMethod == "POST")
        {
            var bodyParam = method.Parameters.FirstOrDefault(p => 
                p.GetAttributes().Any(a => a.AttributeClass?.Name == "FromBodyAttribute"));
            if (bodyParam != null)
            {
                sb.AppendLine($"            var response = await _httpClient.PostAsJsonAsync($\"{route}\", {bodyParam.Name});");
            }
        }
        
        sb.AppendLine("            response.EnsureSuccessStatusCode();");
        
        if (method.ReturnType.Name != "Task" && method.ReturnType is INamedTypeSymbol taskType)
        {
            var resultType = taskType.TypeArguments.FirstOrDefault();
            if (resultType != null)
            {
                sb.AppendLine($"            return await response.Content.ReadFromJsonAsync<{resultType.ToDisplayString()}>();");
            }
        }
        
        sb.AppendLine("        }");
        sb.AppendLine();
    }

    // Helper methods omitted for brevity...
}
```

This is simplified, but you get the idea. The generator inspects the interface, extracts attributes, and generates a concrete implementation.

## The Incremental Generator API

C# 10 introduced incremental generators, which are more efficient. Instead of processing everything on every compilation, incremental generators only process what changed:

```csharp
[Generator]
public class IncrementalApiClientGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var provider = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (node, _) => node is InterfaceDeclarationSyntax,
                transform: static (ctx, _) => (InterfaceDeclarationSyntax)ctx.Node)
            .Where(static m => m is not null);

        context.RegisterSourceOutput(provider, (spc, source) =>
        {
            // Generate code
        });
    }
}
```

Incremental generators are faster because they cache intermediate results and only recompute what's necessary. For large codebases, this can mean the difference between a 30-second compile and a 2-second compile.

## Debugging Source Generators

Debugging source generators is... interesting. You can't just set a breakpoint and debug normally. Here are the techniques I use:

**Method 1: Debugger.Launch()**

```csharp
public void Execute(GeneratorExecutionContext context)
{
    if (!Debugger.IsAttached)
        Debugger.Launch();
    
    // Your code
}
```

This will prompt you to attach a debugger when the generator runs. It's clunky but works.

**Method 2: File Output**

Write intermediate results to files:

```csharp
File.WriteAllText(@"C:\temp\generator-debug.txt", generatedCode);
```

**Method 3: Diagnostic Messages**

Use `ReportDiagnostic` to output information:

```csharp
context.ReportDiagnostic(Diagnostic.Create(
    new DiagnosticDescriptor(
        "SG001",
        "Generator Debug",
        "Processing interface: {0}",
        "Debug",
        DiagnosticSeverity.Info,
        isEnabledByDefault: true),
    Location.None,
    interfaceName));
```

Then check the Error List in Visual Studio or the build output.

## Common Pitfalls

**Pitfall #1: Circular Dependencies**

Source generators can't reference code generated by other source generators in the same compilation. If Generator A needs output from Generator B, you're out of luck. The workaround is to split them into separate projects or use a different approach.

**Pitfall #2: Attribute Location**

If your generator looks for attributes, make sure those attributes are defined in a separate assembly that's referenced by both the generator and the consuming code. Otherwise, the generator won't see them.

**Pitfall #3: Syntax vs Semantic**

Don't do heavy semantic analysis in syntax receivers. Use syntax receivers to filter, then do semantic analysis in `Execute`. The performance difference is significant.

**Pitfall #4: Generated Code Quality**

Generated code should be readable and debuggable. Use proper formatting, include comments, and follow coding standards. You (or someone else) will need to debug this code eventually.

## Real-World Use Cases

Beyond API clients, source generators are great for:

1. **JSON Serialization** - Generate serializers at compile time (like System.Text.Json source generators)
2. **Dependency Injection** - Generate service registration code
3. **Mapping Code** - Generate object-to-object mappers
4. **Validation** - Generate validation code from attributes
5. **Mock Generation** - Generate test mocks from interfaces
6. **Performance-Critical Code** - Generate optimized code paths that would be too verbose to write manually

## Performance Considerations

Source generators run during compilation, so they affect build time. Keep them fast:

- Use incremental generators when possible
- Cache semantic model lookups
- Avoid processing unnecessary syntax nodes
- Don't do I/O operations (except for reading source files)
- Keep syntax receivers lightweight

For a large codebase, a poorly written generator can add minutes to your build time. A well-written incremental generator might add seconds.

## Best Practices

1. **Start with a prototype** - Write the code you want to generate manually first, then automate it.

2. **Use incremental generators** - They're more complex but much faster for large codebases.

3. **Handle edge cases** - What happens with generic types? Nullable reference types? Partial classes? Test thoroughly.

4. **Provide good diagnostics** - If the generator can't process something, tell the user why with a clear error message.

5. **Version your generators** - If you change the generator's output format, version it so existing code doesn't break.

6. **Document the generated code** - Add XML comments or at least a header explaining what was generated and why.

## Conclusion

Source generators are one of those features that seem like overkill until you need them. Once you start using them, you'll find more and more places where they eliminate boilerplate while maintaining performance and type safety.

The learning curve is steep-you need to understand Roslyn's syntax and semantic APIs-but the payoff is worth it. Just remember: generators are a tool, not a solution to every problem. Use them when they make sense, not just because they're cool.

And for the love of all that's holy, test your generators thoroughly. Nothing is worse than a generator that silently produces broken code.

