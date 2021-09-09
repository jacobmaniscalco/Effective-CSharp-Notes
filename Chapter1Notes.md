# Chapter 1 (C# Idioms) Notes

## Item 1: Difference between IEnumerable and IQueryable
- IEnumerable 
    - Returning a query as an IEnumerable<T> sequence means that subsequent operations use the LINQ to Objects implementation and are executed using delegates. This results in the sort operation occuring locally. 
    - Enumerable executes locally, the lambda expressions have been compiled into methods, and must execute on the local machine. This results in pulling more data from whatever source you are querying. 
- IQueryable
    - When the results of a query are executed, the LINQ to SQL libararies compose the results from all the query statements. In the example, that means one call is made to the database. It also means that one SQL query performs both the where clause and the orderby clause.
    - Queries work much more efficiently using IQueryable
    - Queryable uses an expression tree, a data structure that holds all the logic that makes up the actions in the query.
    - IQueryable is restricted in that it cant parse any arbitrary method, meaning, if a query contains other methods calls "not supported" by the IQueryable provider.

## Using implicitly typed variables
- Why would you want to use `var`?
    - Helps readability of the code
    - When using local type inference, you basically tell the compiler it knows more about your types than you do.
- Why would you not want to use `var`?
    - A developer might have an idea of the type just looking at the code, but the compiler might interpret it differently
    - Be careful when working with built-in types, as there are many implicit conversions available on numeric types
    - By explicitly declaring the types of all numeric variables, you retain control over the types used, and you help you compiler warn you about possible dangerous conversions.

Example:
```
var f = GetMagicNumber();
var total = 100 * f / 6;
Console.WriteLine($"Declared Type: {total.GetType().Name}, Value: {total}");
```

The above variables `total` and `f` are ambiguous, as `GetMagicNumber()` could return multiple types of values, therefore the declared type depends on the return type of the `GetMagicNumber()` function. However, just simply using `var` is not the entire issue. It is not clear from reading the code which type is returned from `GetMagicNumber()`

### Instance of allowing the compiler to take over typing
```
public IEnumerable<string> FindCustomersStartingWith1(
string start)
{
    IEnumerable<string> q =
        from c in db.Customers
        select c.ContactName;
 var q2 = q.Where(s => s.StartsWith(start));
    return q2;
}
```

The above code has a performance issue. By strongly declaring the return value, `IQueryable<T>` is no longer used implicitly. Because `IQueryable<T>` derives from `IEnumerable<T>` the compiles does not warn you of this conversion.

### TL;DR
- Explicitly declare all numeric types (int, float, double, and others) rather than using a var declaration. For everything else, just use var.

## Item 2: Prefer readonly to const 
- C# has two different versions of constants: compile-time constants and runtime constants
- You should prefer runtime constants to compile-time constants. Though compile time-constants are slighly faster, they are less flexible. 
- Reserve compile-time constants for when performance is critical and the value of the constant will never change between releases.
- You can declare runtime constants with the `readonly` keyword and compile-time constants with the `const` keyword.
- Compile-time constants can be declared inside methods, whereas read-only constants cannot.

```
// Compile-time constant:
public const int Millennium = 2000;
 // Runtime constant:
public static readonly int ThisYear = 2004;
```

- A compile-time constant is replaced with the value of that constant in the object code.

This construct: 
```
if (myDateTime.Year == Millennium)
```
compiles to the same Microsoft Intermediate Language (MSIL, or IL) as if this had been written:
```
if (myDateTime.Year == 2000)
```
Runtime constants, however are evaluated at runtime, so the IL generated when you reference a read-only constant references the readonly variable, not the value.

- Compile-time constants can be used only for the built-in integral and floatin-point types, enums, or strings. These are the only types that enable you to assign meaningful constant values in initializers. These primitive types are the only ones that can be replaced with literal values in the compiler-generated IL.

You cannot initialize a compile-time constant using the new operator, even when the type being initialized is a value type:
```
// Does not compile, use readonly instead
private const DateTime classCreation = new DateTime(2000, 1, 1, 0, 0, 0);
```
Compile-time constants generate the same IL as though you used the numeric constants in your code, even across assemblies: A constant in one assembly is still replaced with the value when used in another assembly.

Suppose you defined both const and readonly fields in an assembled named `Infrastructure`:
```
public class UsefulValues
{
    public static readonly int StartValue = 5;
    public const int EndValue = 10;
}
```
In another assembly, you reference these values: 
```
for (int i = UsefulValues.StartValue;
    i < UsefulValues.EndValue; i++)
    Console.WriteLine("value is {0}", i);
```

The code gives the following output: 
```
Value is 5
Value is 6
...
Value is 9
```

Now suppose you release a new version of the `Infrastructure` assembly with the following changes:
```
public class UsefulValues
{
    public static readonly int StartValue = 5;
    public const int EndValue = 10;
}
```

And reference those values here:
```
for (int i = UsefulValues.StartValue;
    i < UsefulValues.EndValue; i++)
    Console.WriteLine("value is {0}", i);
```

The expected output would be:
```
Value is 105
Value is 106
...
119
```
You actually receive no output. The loop uses the value 105 for its start and 10 for its end condition. The C# compiler placed the const value of 10 into the `Application` assembly instead of a reference to the storage used by EndValue.

**Updating the value of a public constant should be viewed as an interface change. You must recompile all code that references that constant. Updating the value of a read-opnly constant is an implementation change; it is binary compatible with existing client code.**

### TL;DR
- `const` must be used when the value must be available at compile time: attribute parameters, switch case labels, and enum definitions, and those rare times when you mean to defined a value that does not change from release to release. For everything else, prefer the increased flexibility of readonly constants.

## Item 3: Prefer the `is` or `as` Operators to Casts <a name="item3"></a>
**Use the `as` operator whenever you can because it is safer than blindly casting and is more efficient at runtime.**

- The `as` and `is` operator do not perform an user-defined conversion. They succeed only if the runtime type matches the sought type; they rarely construct a new object to satisfy a request. (The `as` operator will create a new type when converting a boxed value type to an unboxed nullable value type.)

The following code converts an arbitrary object into an instance of MyType. You can write it this way:

```
object o = Factory.GetObject();
 // Version one:
MyType t = o as MyType; 
if (t != null)
{
    // work with t, it's a MyType.
}
else
{
    // report the failure.
```
Or you could write this:
```
object o = Factory.GetObject();
 // Version two:
try
{
    MyType t;
    t = (MyType)o;
    // work with T, it's a MyType.
}
catch (InvalidCastException)
{
    // report the conversion failure.
}
 
```
- The biggest difference between the `as` operator and the `cast` operator is how user-defined conversions are treated. The `as` and `is` operators examine the runtime type of the object being converted; they do not perform any other opertations, except boxing when necessary. Casts, on the other hand, can use conversion operators to conert an object to the requested type. This includes any built-in numeric conversions.

```
public class SecondType
{
    private MyType _value;
 // other details elided
 // Conversion operator.
    // This converts a SecondType to
    // a MyType, see item 29.
    public static implicit operator
    MyType(SecondType t)
    {
        return t._value;
    }
}
```
Suppose an object of SecondType is returned by the Factory.GetObject() function in the first code snippet:
```
object o = Factory.GetObject();
 // o is a SecondType:
MyType t = o as MyType; // Fails. o is not MyType
if (t != null)
{
    // work with t, it's a MyType.
}
else
{
    // report the failure.
}
 // Version two:
try
{
    MyType t1;
    t1 = (MyType)o; // Fails. o is not MyType
    // work with t1, it's a MyType.
}
catch (InvalidCastException)
{
    // report the conversion failure.
}
```

**User-defined conversion operators operate only on the compile-time type of an object, not on the runtime.**

## Item 4: Replace `string.Format()` with Interpolated Strings <a name="item4"></a>
- Enables the compiler to provide better static type checking, which descreases the changes of mistakes. It also provides a richer syntax for the expressions that produce the string.

```
Console.WriteLine($"The value of pi is {Math.PI}");
```
The code generated by the string interpolation will call a formatting method whose argument is a params array of objects. The `Math.PI` is a double, which is a value types. In order to coerce that double to be an Object, it will be boxed. This can have a significant impact on performance. Therefore, you should use your expressions to convert these arguments to strings. This avoids the necessity to box any value types are are used in the expressions:

```
Console.WriteLine($"The value of pi is {Math.PI.ToString()}");
```

Formatting the string can be achieved by using the `:` symbol:
```
Console.WriteLine($"The value of pi is {Math.PI:F2}");
```

You can use the null coalescing or the null propogation operator to handle missing values clearly:
```
Console.WriteLine($"The customer's name is {c?.Name ?? "Name is missing"}");
```
This means that string nesting is supported in string interpolation.

The following code shows how you might format a string to display information about a record, or the index for a missing record:
```
string result = default(string);
Console.WriteLine($@"Record is {(records.TryGetValue(index,out
result) ? result :
$"No record found at index {index}")}");
```

You can even use LINQ queries to create the values that are used in an interpolated string.
```
var output = $@"The First five items are: {src.Take(5).Select(
n => $@"Item: {n.ToString()}").Aggregate((c, a) => $@"{c}{Environment.NewLine}{a}")})";
```

## Item 5: Prefer FormattableString for Culture-Specific Strings <a name="item5"></a>

### TL;DR
- When you need a specific culture, you must explicitly tell the string interpolation to create a `FomattableString`, and you then can convert that into a string using any specific culture that you want.

## Item 6: Avoid String-ly Typed APIs <a name="item6"></a>

The `nameof()` expression was added in C# 6.0 to replace a symbol with a string containing its name. The most common exaple of this is to implement the `INotifyPropertyChanged` interface:
```
public string Name 
{
    get { return name; }
    set 
    {
        if(value != name)
        {
            name = value;
            ProprtyChanged?.Invoke(this,
            new PropertyChangedEventArgs(nameof(Name)));
        }
    }
}
private string name;
```

- With the `nameof` operator, any changes to the property name are reflected correctly in the string used in the event arguments.
- The `nameof()` operator evaluates to the name of the symbol. It works for types, variables, interfaces, and namespaces. 

Using `nameof` inside exception arguments to replace the hard-coded text, ensuring that the names match even after any rename operation:
```
private static void ExceptionMessage(object thisCantBeNull)
    {
        if (thisCantBeNull == null)
        {
            throw new ArgumentNullException(nameof(thisCantBeNull), "Told you so.");
        }
    }
```

## Item 7: Expresss Callbacks with Delegates <a name="item7"></a>

- Anytime you need to configure the communication between classes and you desire less coupling than you get from interfaces, a delegate is the right choice. Delegates let you configure the target at runtime and notify multiple clients. 
- The .NET Framework library defines many common delegate forms using `Predicate<T>`, `Action<>`, and `Func<>`.

The `List<T>` class also contains many methods that make use of callbacks:
```
List<int> numbers = Enumerable.Range(1, 200).ToList();
 var oddNumbers = numbers.Find(n => n % 2 == 1);
var test = numbers.TrueForAll(n => n < 50);
 numbers.RemoveAll(n => n % 2 == 0);
 numbers.ForEach(item => Console.WriteLine(item));
```

- All delegates are multicast delegates. Multicast delegates wrap all the target functions that hav ebeen added to the delegate in a single call. This will mean that the return value will be the return value of the last target function invoked by the multicast delegate. 
- Client callbacks should be implemented using delegates in .NET

## Item 8: Use the Null Conditional Operator for Event Invocations <a name="item8"></a>

Old code to handle a safe event invocation would look like this:
```
public class EventSource
{
    private EventHandler<int> Updated;
 public void RaiseUpdates()
    {
        counter++;
        Updated(this, counter);
    }
    private int counter;
}
```
Using a null conditional operator results in:
```
public void RaiseUpdates()
{
    counter++;
    Updated?.Invoke(this, counter);
}
```
This code uses the null conditional operator `?.` to safely invoke the event handles and results in thread-safe operations.

## Item 9: Minimize Boxing and Unboxing <a name="item9"></a>
- The .NET Framework was deisnged with a single reference type, System.Object, at the root of the entire object hierarchy. 
- Boxing places a value type in an untyped refernce object to allow the value type to be used where a reference type is expected. 
- Unboxing extracts a copy of that value type from the box. 
- Boxing and unboxing are performance intensive operations. They can result in creating temporary copies of objects. Anytime you need to retrieve anything from the box, a copy of the value type is created and returned. 

The following code even uses boxing:
```
Console.WriteLine($"A few numbers:{firstNumber},{secondNumber}, {thirdNumber}");
```
The work done to create the interpolated string uses an array of System.Object references. Integers are vaule types and must be boxed sothat they can be passed to the compiler-generated method that creates the string from the values. In addition, inside the method, code reaches inside the box to call the `ToString()` method of the object in the box.
To avoid this penalty, you should convert your tpes to string instances yourself before you send them to WriteLine:
```
Console.WriteLine($@"A few numbers:{firstNumber.ToString()}, {secondNumber.ToString()}, {thirdNumber.ToString()}");
```
Since this code uses the known type of integer, and value types (integers) are never implicitly converted to System.Object. 

```
public struct Person
{
    public string Name { get; set; }
 public override string ToString()
    {
        return Name;
    }
}
// Using the Person in a collection:
var attendees = new List<Person>();
Person p = new Person { Name = "Old Name" };
attendees.Add(p);
 // Try to change the name:
// Would work if Person was a reference type.
Person p2 = attendees[0];
p2.Name = "New Name";
 // Writes "Old Name":
Console.WriteLine(attendees[0].ToString( ));
 int i = 25;
object o = i; // box
Console.WriteLine(o.ToString());

```

## Item 10: Use the `new` Modifier only to React to Base Class Updates <a name="item10"></a>
You would assume that these two block of code do the same thing, if the two classes are related by inheritance
```
object c = MakeObject();
 // Call through MyClass reference:
MyClass cl = c as MyClass;
cl.MagicMethod();
 // Call through MyOtherClass reference:
MyOtherClass cl2 = c as MyOtherClass;
cl2.MagicMethod();
```
However, when the new modifier is involved, that isn't the case:
```
public class MyClass
{
    public void MagicMethod()
    {
        Console.WriteLine("MyClass");
        // details elided.
    }
}
 public class MyOtherClass : MyClass
{
    // Redefine MagicMethod for this class.
    public new void MagicMethod()
    {
        Console.WriteLine("MyOTherClass");
        // details elided
    }
}
```

The only timw when you want to use the `new` modifier is when you incorporate a new version of a base class that contains a method name that you already use. 
Assume you have created the following class in your library, using BaseWidget which is defined in another library:
```
public class MyWidget : BaseWidget
{
    public new void NormalizeValues()
    {
        // details elided.
    }
}
```
After a while, the BaseWidget class has implemented their own version of `NormalizeValues()` method. The solution to this would be either to change the name of your `NormalizeValues` method...
```
public class MyWidget : BaseWidget
{
    public void NormalizeAllValues()
    {
        // details elided.
        // Call the base class only if (by luck)
        // the new method does the same operation.
        base.Normalizevalues();
    }
}
```
Or use the `new` keyword:
```
public class MyWidget : BaseWidget
{
    public new void NormalizeValues()
    {
        // details elided.
        // Call the base class only if (by luck)
        // the new method does the same operation.
        base.Normalizevalues();
    }
}
```
The `new` keyword handles the case when an upgrade to a base class collides with a member that you previously declared in your class.