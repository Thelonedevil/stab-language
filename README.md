# Design Goals #
The goals used to design Stab were as follows:
  * The syntax should be readable without difficulties by Java/C# programmers
  * The execution of a Stab program should be as efficient as the execution of the same program written in Java
  * The generated bytecode should be usable transparently from Java
  * The Java libraries should be usable from Stab programs without adaptations

# Current State #
All the features listed below are implemented and tested.
The compiler is stable enough to compile its own source code (Stab is written 100% in Stab).

A beta version is available for download.

# Features #
This section presents an overview of the features found in the Stab language. Additional informations can be found here: StabLanguage.

## JVM compatibility ##
Stab has almost all the features that can be found in Java:
  * The same primitive types and operators with automatic boxing/unboxing
  * Arrays, classes, interfaces, enums, constructors, fields and methods
  * Methods with variable number of arguments
  * C-like `if`/`else`, `for`, `while`, `do`, `switch` statements
  * Exceptions handling with `try`/`catch`/`finally` and `throw`
  * `synchronized` blocks and methods
  * Compile-time generic programming including constraints and wildcards
  * Annotations
Stab supports nested classes but not inner classes. To access a non-static member of an outer class, an explicit reference must be used.

## Implicitly typed variables ##
A local variable declared using the contextual keyword `var` has its type inferred from the type of the initial value.
```
var s = "abc"; // 's' is now a string each time it is used in this scope
var i = 0; // 'i' is an int
var m = new HashMap<String, String>();
int var = 0; // 'var' is a contextual keyword. It can be used as an identifier
```

## Delegates ##
To provide a partial replacement of anonymous classes the language has delegates. Delegates can be used to store and combine pairs of objects and methods for a later invocation.
For example:
```
using java.lang;
public class Test {
    delegate void D(int i);
    public static void main(String[] args) {
        var obj = new Test();
        var d = new D(obj.m1);
        d += new D(obj.m2);
        d(123);
    }
    void m1(int i) {
        System.out.println("From m1: " + i);
    }
    void m2(int i) {
        System.out.println("From m2: " + i);
    }
}
```
The output is:
```
From m1: 123
From m2: 123
```
## Lambda expressions ##
Another step to replace inner classes is to provide lambda expressions. Lambda expressions can be assigned to delegate types but also to interfaces with a single method. This way they can be used directly in numerous places with existing libraries.
```
using java.lang;
public class Test {
    delegate int Func(int n);
    public static void main(String[] args) {
        Func fact = null;
        fact = p => (p < 1) ? 1 : p * fact(p - 1);
        var arg = Integer.parseInt(args[0]);
        System.out.println("fact(" + arg + ") = " + fact(arg));
    }
}
```
## Iterator blocks ##
Implementations of `java.lang.Iterable<T>` and `java.util.Iterator<T>` can be done easily using the `yield return` and `yield break` statements.
```
using java.lang;
using stab.lang;
public class Test {
    public static void main(String[] args) {
        var result = 1;
        foreach (var i in range(1, 5)) {
            result *= i;
        }
        System.out.println("fact(5) = " + result);
    }
    static IntIterable range(int start, int count) {
        while (count-- > 0) {
            yield return start++;
        }
    }
}
```
Note: The interface `stab.lang.IntIterable` (a sub-interface of `java.lang.Iterable<Integer>`) is used here to avoid any boxing/unboxing when the iterable is used in a `foreach` statement.

## Extension methods ##
Extensions method are static methods called using the instance method syntax. For example a `where` method to filter the contents of a `java.lang.Iterable<T>` can be implemented the following way:
```
using java.lang;
using java.util;
interface Filter<T> {
    boolean invoke(T item);
}
static class Extensions {
    // Extension methods are declared using the 'this' modifier
    // on the first parameter
    static Iterable<T> where<T>(this Iterable<T> source, Filter<T> filter) {
        foreach (var s in source) {
            if (filter.invoke(s)) {
                yield return s;
            }
        }
    }
}
class Test {
    public static void main(String[] args) {
        var list = new ArrayList<String> { "a", "aa", "aaa" };
        // Lambda expressions can be used to implement simple interfaces
        foreach (var s in list.where(p => p.length() > 1)) {
            System.out.println(s);
        }
    }
}
```
The output is:
```
aa
aaa
```
## Properties and indexers ##
To continue with syntactic sugar, properties and indexers can be declared in classes and methods (or specified on third party libraries).
Here is the syntax:
```
class C {
    // Manually implemented property
    private int myProperty;
    public int MyProperty {
        get {
            return myProperty;
        }
        set {
            myProperty = value;
        }
    }
    // Indexer
    private int[] array;
    public int this[int i] {
        get {
            return array[i];
        }
        set {
            array[i] = value;
        }
    }
    // Automatically implemented property
    public int MyProperty2 {
        get;
        set;
    }
}
```
The compiler generates methods following the Java beans convention for properties, and a pair of `getItem()` / `setItem()` methods for indexers. Theses methods are decorated with annotations to allow the compiler to detect properties and indexers in compiled code.
The compiler also use a mecanism to add annotations on external libraries. A library is provided to annotate some classes of the Java platform.
For example `java.lang.String.charAt()` and `java.util.Map.put()` are specified as indexers.

## Anonymous objects ##
Anonymous objects are used to easily create temporary objects inside a method. The syntax to declare anonymous objects is:
```
// Property names explicitly specified
var obj1 = new { Name = "John", Age = 30 };
// obj2 has 'Name' and 'Age' properties deduced from expressions
var obj2 = new { obj1.Name, obj1.Age };
```
`equals` and `hashCode` methods are overriden on anonymous objects. For instance `obj1.equals(obj2) == true` in the previous example because the two objects have `Name` and `Age` properties declared in the same order with the same types and the same values.
Anonymous object properties are read-only.
## Language Integrated Queries ##
All the previous features put together and it becomes possible to implement LINQ like it is specified in C#:
```
using java.lang;
using stab.query;
public class Test {
    public static void main(String[] args) {
        // Sorts the arguments starting with "-" by length and then
        // using the default string comparison
        var query = from s in Query.asIterable(args)
                    where s.startsWith("-")
                    orderby s.length(), s
                    select s;
        foreach (var s in query) {
            System.out.println(s);
        }
    }
}
```
Which is a syntactic mapping to:
```
using java.lang;
using stab.query;
public class Test {
    public static void main(String[] args) {
        var query = Query.asIterable(args).
                where(s => s.startsWith("-")).
                orderBy(s => s.length()).thenBy(s => s);
        foreach (var s in query) {
            System.out.println(s);
        }
    }
}
```
The package `stab.query` provides all the extension methods required to use LINQ on `java.lang.Iterable<T>` but queries can be used virtually on everything if the necessary methods are provided (what is called _Query expression pattern_ in the C# specification).

## Automatic resources management ##
The `using` statement can be used to automatically dispose resources:
```
using (var reader = new FileReader("test.txt")) {
    // Read the file
}
```
is equivalent to:
```
var reader = new FileReader("test.txt");
try {
    // Read the file
} finally {
    if (reader != null) reader.close();
}
```
Because the Java platform does not have a common interface to specify disposable objects, the Stab compiler uses the `stab.lang.Dispose` annotation to find the method to call to dispose the instances of a given type.

## Code generation facilities ##
### Preprocessor ###
Stab has a preprocessor similar to the preprocessor of the C# language:
  * Symbols can defined with `#define` and undefined with `#undef`.
  * Parts of the source code can be skipped using `#if`/`#elif`/`#else`/`#endif`
  * Regions can be specified with `#region` and `#endregion`
  * Errors and warnings can be triggered by `#error` and `#warning`
  * Warnings can be disabled with `#pragma warning`
  * Current line and filename can be overriden using `#line`
Symbols can be defined externally using an [option of the command line compiler](StabCompiler#define.md).

### Partial classes and methods ###
Classes with the `partial` modifier can have their definition splitted in one or more files.
Methods with the `partial` modifier can have an optional implementation.

### goto statement ###
As a complement to the preprocessor and to partial classes for automatic code generation, the goto statement is supported. It can be used to jump in the same lexical scope, or an enclosing scope.

## Miscellaneous ##
### switch statement ###
The `switch` statement is extended to work with `java.lang.String` in addition to integral and enumerated values.

It is also an error to fall through one case to another. `goto case <label>;` or `goto default;` must be used.

### Object initializers ###
The following syntax can be used to initialize object properties:
```
class C {
    String Name { get; set; }
    int    Age  { get; set; }
}
class Test {
    public static void main(String[] args) {
        var c = new C { Name = "John", Age = 30 };
        System.out.println("Name = " + c.Name + " Age = " + c.Age);
    }
}
```

### Collection initializers ###
The following syntax can be used to initialize objects with `add` or `put` methods:
```
var list = new ArrayList<String> { { "a" }, { "b" }, { "c" } };
var map = new HashMap<String, String> {
    { "key1", "value1" }, { "key2", "value2" } };
```
When the `add` or `put` method has a single argument, a more compact form can be used and the initialization of the previous list becomes:
```
var list = new ArrayList<String> { "a", "b", "c" };
```

### Verbatim identifiers ###
Identifiers can be preceded by `@` to allow keywords to be used as identifiers.

### Types and packages aliasing ###
Types and packages can be renamed locally in a file in order to avoid ambiguities, or just for cosmetic reasons.

Package aliasing:
```
using ju = java.util;
using ja = java.awt;
class C {
    ju.List list1;
    ja.List list2;
}
```
Type aliasing:
```
using StringList = java.util.ArrayList<java.lang.String>;
class C {
    StringList list = new StringList();
}
```

### Conditional annotation ###
Calls to a method decorated with the `stab.lang.Conditional` annotation can be included or removed by the compiler from the generated bytecode depending on defined preprocessor symbols.
For example simple assertions can be implemented like this:
```
using java.lang;
using stab.lang;
public class Debug {
    [Conditional({ "DEBUG" })]
    public static void assert(boolean test) {
        if (!test) throw new Exception("Assertion failed");
    }
}
```
A method can be instrumented this way:
```
#define DEBUG
public class Test {
    public void method(int arg) {
        Debug.assert(arg > 0);
    }
}
```

### Operators ###
#### Null coalescing ####
```
var a = b ?? c;
```
is equivalent to
```
var a = (b != null) ? b : c;
```

#### As ####
```
var a = b as T;
```
is equivalent to
```
var a = (b instanceof T) ? (T)b : null;
```

### XML documentation ###
Comments starting with `///` or `/**` are considered as XML comments and can be extracted using an [option of the command line compiler](StabCompiler#doc.md).