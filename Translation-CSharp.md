# Translation and metaprogramming

WebSharper provides several ways to customize the way functions and
values are compiled to JavaScript:

* [Directly providing JavaScript code](#javascript);
* [Customizing the compiled name of the value](#name);
* [Classes that directly represent JavaScipt build-in types](#types);
* [Access JavaScipt properties dynamically](#dynamic);
* [Transforming the code](#meta) during compilation, a concept
  known as metaprogramming.

<a name="javascript"></a>
## Embedding JavaScript

There are two ways of directly inserting JavaScipt code into a WebSharper project.

### JavaScript function body

The `Direct` attribute takes a JavaScript expression as a string parameter.
This will get parsed by the compiler and any syntax errors will be reported in a
compile error message.
This parsed expression will be used as a function body, `return` is automatically
added for the last value.

If you don't want the use function from .NET, the `WebSharper.JavaScript.Interop.X<T>()` method
throws an exception of type `WebSharper.JavaScript.ClientSideOnly` with the message
"This function is intended for client-side use only.".

You can use placeholders for the function or method arguments.
For named parameters, the name with a `$` prepended is recognised.
For example:

    [Direct("$x + $y")]
    public static int Add(int x, int y) => Interop.X<int>();

Also you can access parameters by index.
In let-bound functions in modules and static methods of classes, the parameters
are indexed from 0.

    [Direct("$0 + $1")]
    public static int Add(int x, int y) => Interop.X<int>();
    
In instance methods, `$0` translates to the self indentifier, and method parameters
are indexed from 1.
(You can also use `$this` for the self identifier, but this recommended against, as
a parameter named `this` can override it, and it does not work for extension members
which are actually static methods in translated form.)

    [Direct("Math.sqrt($0.x * $0.x + $0.y * $0.y)")]
    public float this.GetLength() => Interop.X<float>();

### Inlined JavaScript code

The `Inline` attribute  takes a JavaScript expression as a string parameter.
(It can also be used together with the `JavaScript` attribute to inline a function
translated from F#.)
This will be parsed, and inlined to all call sites of this function.
Only a subset of JavaScript operators and keywords can be used which can be translated
to the "core" AST used internally by WebSharper to optimize output.

Parameter placeholders work exactly as with `Direct`. 

    [Inline("$x + $y")]
    public static int Add(int x, int y) => Interop.X<int>();

## Inlines accessing global object

If you want a non-cached access to the global `window` object, use the `$global` fake variable inside inline strings.

For example, `[Inline("myLibrary.doSomething()")]` assumes that `myLibrary` is initialized before the current script starts running (WebSharper itself takes care of this, if you use the `Require` attribute) and will not change, so access is safe to shorten.
On the other hand, `[Inline("$global.myLibrary.doSomething()")]` always accesses `myLibrary` as a property of `window.` 
	
### Inline Helper

The `WebSharper.JavaScript.Pervasives.JS.Inline` function parses its first parameter at compile-time as JS code and includes
that in the result. It can contain holes, named `$0`, `$1`, ... and variable arguments will
be passed to the inline. Examples:

    using static JSI = WebSharper.JavaScipt.Pervasives.JS
    //...
    var zeroDate = JSI.Inline("new Date()");
    var date = JSI.Inline("new Date($0)", 1472226125177);	

	
### Constant

The `Constant` attribute takes a literal value as parameter.
It can annotate a property or a union case which will be translated to the literal provided.

<a name="name"></a>
## Naming

The `Name` attribute takes a string parameter and allows specifying
the name of a function or class in the translation.
For example:

    [Name("add")]
    public static int OriginalNameForAdd(int x, int y) => x + y;

### Naming abstract members

If you set a fixed translated name with the `Name` attribute on an abstract member of a class
or interface, all inheriting and overriding members will have that exact translated name.
If a class is overriding or implementing two abstract members that has the same fixed name,
it will result in a compile-time error, and you have to change one of the fixed names to resolve it.

Automatically, interface members have a unique long translated name generated that contains the full type name.
This guarantees no conflicts without using the `Name` attribute.
If you want to shorten it for readability of the JS output and making it smaller,
you can use the `Name` attribute on the interface type to specify a short name for the interface.
It is recommended that it is unique across your solution.
You can also use `[Name("")]` on the interface type to make all interface methods have the same translated name
as their original .NET name (if not specified otherwise by a `Name` on the member).

If you use the `[JavaScipt(false)]` attribute on an interface or abstract class, its member's naming will not be tracked by the compiler, this is sometimes useful for divergent behavior of the code in .NET and in JavaScipt.
	
<a name="types"></a>
## Types for interacting with outside JavaScript code

There are classes in the `WebSharper.JavaScript` namespace that are direct representations of ECMA standard
library JavaScipt types.
The `WebSharper.JavaScript` namespace declares a `.ToJS()` extension method on all .NET types to safely
convert them to their JavaScipt representation.

For JavaScipt functions, the `Function` class is an untyped representation, but if you know the signature of
the JavaScipt function, there are more strongly typed alternatives:

* For any function, that do not care about the `this` argument, you can use delegates.
* For functions that work with the `this` argument, use `ThisAction` and `ThisFunc` classes.
They have a constructor that takes a delegate, for which the first argument will have the `this` value.

        var logger = new ThisAction<object>(x => WebSharper.JavaScript.Console.Log(x));

* For functions taking variadic arguments, use `ParamsAction` and `ParamsFunc` classes.
* Finally for functions using the `this` value and have variadic arguments, use `ThisParamsAction` and `ThisParamsFunc`

<a name="dynamic"></a>
## Access JavaScipt properties dynamically

The `WebSharper.JavaScript` namespace declares a `.GetJS` extension method, that can be used to get JavaScipt properties dynamically.
Example: `x.GetJS<int>("size", "width")` is translated to `x.size.width` and usable as an `int` value.
You can use `x.GetJS<T>()` to just use the value of `x` exposed as another .NET type `T`.

### C# dynamic
You can use the `dynamic` type to access JavaScipt properties and functions without any extra helpers:

    dynamic d = names;
    var name = d.getItems()[3].name; // translates directly to: d.getItems()[3].name
    
Also, operators on dynamic values are translated directly if there is a JavaScipt equivalent.

<a name="meta"></a>
## Metaprogramming

Macro and generator documentation will be available for stable release of WebSharper 4.