# Working With LINQ

## Item 29: Prefer Iterator Methods to Returning Collections
**Whenever you create a method that returns a sequence, you should create an iterator method**
- An interator method is a method that uses the `yield return` syntax to generate elements of the sequence as they are asked for.
```
public static IEnumerable<char> GenerateAlphabet()
{
    var letter = 'a';
    while (letter <= 'z')
    {
        yield return letter;
        letter++;
    }
}
```

The code above generates a class that is similar to the following construct:
```
public class EmbeddedIterator : IEnumerable<char>
{
    public IEnumerator<char> GetEnumerator() =>
        new LetterEnumerator();
    IEnumerator IEnumerable.GetEnumerator() =>
        new LetterEnumerator();
    public static IEnumerable<char> GenerateAlphabet() =>
        new EmbeddedIterator();
    private class LetterEnumerator : IEnumerator<char>
    {
        private char letter = (char)('a' - 1);
        public bool MoveNext()
        {
            letter++;
            return letter <= 'z';
        }
        public char Current => letter;
        object IEnumerator.Current => letter;
        public void Reset() =>
            letter = (char)('a' -1);
        void IDisposable.Dispose() {}
    }
}
```

## Item 30: Prefer Query Syntax to Loops

- Query syntax enables you to move your program logic from a more imperative model to a declarative model.

The following code shows an imperative method of filling an array and then printing its contents to the console:
```
var foo = new int[100];
 for (var num = 0; num < foo.Length; num++)
    foo[num] = num * num;
 foreach (int i in foo)
    Console.WriteLine(i.ToString()); 
```

Refactoring this code to use queryt syntax creates more readable code and enables reuse of different building blocks. As a first step, you can change the generation of the array to a query result: 
```
var foo = (from n in Enumerable.Range(0, 100)
    select n * n).ToArray();
```
You can do a similar change to the second loop, although you'll need to write an extension method to perform some actions on all elements:
```
foo.ForAll((n) => Consolw.WriteLine(n.ToString()));
```
As the operation becomes more complex, the imperative version becomes much more difficult to comprehend. 

- Every query syntax has a corresponding method call syntax. Sometimes the query is more natural, and sometimes the method call is more natural. Furthernmore, some methods do not have equivalent query syntax. Methods such as `Take`, `Take-While`, `Skip`, `SkipWhile`, `Min`, and `Max` require you to use the method syntax at some level.

## Item 31: Create Composable APIs for Sequences 
- Often times, you will write algorithms to transform some data between the source collection and the ultimate result, with intermediate collections along the way. This strategy means interating the collection once for every transformation, inducing a execution time cost for algorithms that contain many transformations of the elements. An alternative to this would be creating one method that processes each transformation in one loop, thereby reducing the number of loops and the memory footprint. However, this reduces code reusability.

- ***C# iterators** enable you to creat emethod that operate on a sequence but process and return each element as it is requested. The `yield` return statement lets you create methods that return sequences. These iterator methods have a sequence as one input and produce a sequence as output. By utilizing the `yield` return statement, these iterator methods do not need to allocate storage for the entire sequence of elements. 

- This **deferred execution model** means that your algorithms use less storage space and compose better than tranditional imperative methods. 

The following method takes as its input an array of integers and writes all the unique values to the output console:
```
private static void Unique(IEnumerable<int> nums)
    {
        var uniqueVals = new HashSet<int>();
        foreach(var num in nums)
        {
            if (!uniqueVals.Contains(num))
            {
                uniqueVals.Add(num);
                Console.WriteLine(num);
            }
        }
    }
```
This is a simple method, but you are unable to reuse any of the interesting parts. However, chances are that this search unique numbers would be useful in other places in your program. 
Suppose you write the routine this way:
```
private static IEnumerable<int> UniqueV2(IEnumerable<int> nums)
    {
        var uniqueVals = new HashSet<int>();
        foreach(var num in nums)
        {
            if(!uniqueVals.Contains(num))
            {
                uniqueVals.Add(num);
                yield return num;
            }
        }
    }
```
And you can use it this way: 
```
foreach(var num in UniqueV2(arr))
    {
        Console.WriteLine(num);
    }
```

The `yield` return statement plays an interesting trick: It returns a value and retains information about its current location and the current state of its internal iteration. This is a `continuable method`, which keeps track of their state and resumes execution at their current location when code enters them again. 

- The fact that `Unique()` is a continuation method provides two important benefits. First, that's what enables the deferred evaulation of each element. Second, and more important, the deferred execution provides a composability that would be difficult to achieve if each method had its own foreach loop 

## Item 32: Decouple Iterations from Actions, Predicates, and Functions

- A **predicate** is a Boolean method that determines whether an element in a sequence matches some condition. An **action delegate** performs some action on an element in the collection. 
```
namespace System
{
    public delegeate bool Predicate<T>(T obj);
    public delegate void Action<T>(T obj);
    public delegate TResult Func<T, TResult>(T arg);
}
```

- Filter methods use `Predicate` to perform their tests. Predicate defines which objects should be passed or blocked by the filter. 
You can build a generic filter that returns a sequence of all item that meet some criterion:
```
public static IEnumerable<T> Where<T>
    (IEnumerable<T> sequence, 
     Predicate<T> filterFunc)
{
    if (sequence == null)
        throw new ArguementNullException(nameof(sequence), "sequence must not be null")'
    if(filterFunc == null)
        throw new ArguementNullException("Predicate must not be null");
    foreach(T item in sequence)
        if(filterFunc(item))
            yield return item;
}
```
Each element in the input sequence is evaluated using the `Predicate` method.

## Item 33: Generate Sequence Items as Requested

- Iterator methods do not necessarily need to take a sequence as an input parameter. An iterator method that uses the `yield` return approach can create a new sequence, essntially becoming a factory for a sequence of elements. Instead of creating the entire collection before proceeding with any operations, you create the values only as requested.
```
static IList<int> CreateSequence(int numberOfElements, int startAt, int stepBy)
{
    var collection = 
        new List<int>(numberOfElements);
    for (int i = 0; i < numberOfElements; i++)
        collections.Add(startAt + i * stepBy);
    
    return collection;
}
```
This implementation has deficiencies compared with using `yield` return to create the sequence. First, this technique assumes that you're putting the results into a `List<double>`.

You can remove the limitations by making the generation function as an iterator method:
```
static IEnumerable<int> CreateSequence(int numberOfElements, int startAt, int stepBy)
{
    for (var i = 0; i < numberOfElements; i++)
    {
        yield return startAt + i * stepBy;
    }
}
```

## Item 34: Loosen Coupling by Using Function Parameters

Using **function parameters** means that your component is not responsible for creating the concrete type of the classes it needs.

The `List.RemoveAll()` method signature teakes a elegate of type `Predicate<T>`:
```
void List<T>.RemoveAll(Predicate<T> match);
```
The .NET Framework designers could have implemented this method by defining an interface:
```
// Improper extra coupling.
public interface IPredicate<T>
{
    bool Match(T soughtObject);
}
public class List<T>
{
    public void RemoveAll(IPredicate<T> match)
    {
        // elided
    }
    // Other apis elided
}
//The usage for this second version is quite a bit more work:
public class MyPredicate : IPredicate<int>
{
    public bool Match(int target) =>
        target < 100;
}
```
It is oftern much easier on developers who use your class when you define interfaces using delegates or other loose-coupling-methods.
- The reason for using delegates instead of an interface is that the delegate is not a fundamanetal attribute of the type.

## Item 35: Never Overload Extension Methods

The following is an example that misuses extension methods. Suppose you have a simple `Person` class that was 
generated by some other library:
```
public sealed class Person
{
    public string FirstName
    {
        get; 
        set;
    }
    public string LastName
    {
        get;
        set;
    }
}
```
You might consider writing an extension method to create a report of people's names to the console:
```
// Bad start.
// extending classes using extension methods
namespace ConsoleExtension
{
    public static class ConsoleReport
    {
        public static string Format(this Person taget) => $"{target.LastName, 20}, {target.FirstName, 15}";
    }
}
```

Generating the report:
```
static void Main(string[] args)
{
    List<Person> somePresidents =
        new List<Person>{
    new Person{
        FirstName = "George",
        LastName = "Washington" },
    new Person{
        FirstName = "Thomas",
        LastName = "Jefferson" },
    new Person{
        FirstName = "Abe",
        LastName = "Lincoln" }
        };
 foreach (Person p in somePresidents)
        Console.WriteLine(p.Format());
}
```
Imagine requirements change, and you need to create a report in XML format:
```
// Even Worse
// ambiguous extension methods
// in different namespaces
namespace XmlExtensions
{
    public static class XmlReport
    {
        public static string Format(this Person target) =>
            new XElement("Person",
            new XElement("LastName", target.LastName),
            new XElement("FirstName", target.FirstName)
            ).ToString();
    }
}
```
This functionality clearly needs a different implementation.

The following code create `Format` methods that use the `Person` type. 
```
public static class PersonReports
{
    public static string FormatAsText(Person target)=>
        $"{target.LastName,20},{target.FirstName,15}";
    public static string FormatAsXML(Person target) =>
        new XElement("Person",
            new XElement("LastName", target.LastName),
            new XElement("FirstName", target.FirstName)
            ).ToString();
}
```

## Item 36: Understand How Query Expression Map to Method Calls
- LINQ is built on two concepts: a query language, and a translation from that query language into a set of methods. Every query expression has a mapping to a method call or calls. 

- The .NET base library provides two general-purpose reference implementations. `System.Linq.Enumerable` provides extension methods on `IEnumerable<T>` that implement the query expressionm pattern. `System.Linq.Queryable` provides a similar set of extension methods on `IQueryable<T>` that support a query provider's ability to translate queries into another format for execution. 


## Item 37: Prefer Lazy Evaluation to Eager Evaluation in Queries
**Lazy evaluation** happens when each new enumeration produces new results.
**Eager evaluation** happens when you grab a set of variables and you want to retrieve them once and retrieve them now.

The following code generates a sequence and then iterates that sequence three times, with a pause between iterations:
```
private static IEnumerable<TResult> Generate<TResult>(int number, Func<TResult> generator)
{
    for (var i = 0; i < number; i++)
    {
        yield return generator();
    }
}
private static void LazyEvaluation()
{
    WriteLine($"Start time for Test One: {DateTime.Now:T}");
    var sequence = Generate(10, () => DateTime.Now);

    WriuteLine("Waiting...\tPress Return.");
    ReadLine();

    WriteLine("Iterating...");
    foreach(var value in sequence)
        WriteLine($"{value:T}");
    
    WriteLine("Waiting...\tPress Return");
    ReadLine();
    WriteLine("Iterating...");
    foreach(var value in sequence)
        WriteLine($"{value:T}");
}
```

In this example, notice that the sequence is generated each time it is iterated, as evident by the different time stamps. The sequence variable does not hold the elements create, but rather holds the expression tree that can create the sequence.

## Item 38: Prefer Lambda Expressions to Methods
- Some code will convert lambda expressions into a delegate to execute the code in your query expression. Other classes will create an expression tree from the lambda expression, parse that expression, and execute it in another environment. LINQ to Objects does the former and LINQ to SQL does the latter.

- LINQ to Objects performs queries on local data stores, usually store in a generic collection. 
- LINQ to SQL uses the expression tree contained in the query (the logical representation of the query). This parsing creates a T-SQL query, and the query string (as T-SQL) is sent to the database engine and is executed. 

- When creating a reusable library for which the data source could be anything, you must structure the code so that it will work correctly with any data source. This means keeping lambda expressions seperate, an as inline code, for the library to function correctly. 

```
private static IQueryable<Employee> LowPaidSalariedFilter
    (this IQueryable<Employee> sequence) =>
        from s in sequence
        where s.Classification == EmployeeType.Salary &&
        s.MonthlySalary < 4000
        select s;

var allEmployees = FindAllEmployees();

var salaried = allEmployees.LowPaidSalariedFilter();

var earlyFolks = salaried.Where(e => e.YearsOfService > 20);

var newest = salaried.Where(e => e.YearsOfService < 2);
```

- One of the most efficient ways to reuse lambda expressions in complicated queries is to create extension methods for those queries on closed gneric types. You should support both the LINQ-to_SQL and the LINQ-to-Objects implementations via an overload. 

## Item 39: Avoid Throwing Exceptions in Functions and Actions
- When you create code that executes over a sequence of values and the code throws an exception somewhere in that sequence processing, you'll have problems recovering state.

```
var allEmployees = FindAllEmployees();
allEmployees.ForEach(e => e.MonthlySalary *= 1.05M);
```
- If an exception is thrown somewhere in the sequence processing, some employees would receive raises and some would not thereby losing a consistant state. 
- This occurs because the code mofifies elements of a sequence in place and does not follow the strong exception gurantee.
- This can be fixed by guranteeing that whenever the method does not complete, the observable program state does not change.

- Not all methods exhibit this problem (only examine a sequence but do not modify it):
```
var total = allEmployees.Aggregate(0M, (sum, emp) => sum + emp.MonthlySalary);
``` 

- The method can be reworked to take into account the possibility of an exception by doing the work on a capy and then replacing the original sequence with the copy only if the operation completes successfully. 
```
var updates = (from e in allEmployees
    select new Employee
    {
        EmployeeID = e.EmployeeID,
        Classification = e.Classification,
        YearsOfService = e.YearsOfService,
        MonthlySalary = e.MontlySalary *= 1.05M
    }).ToList();
allEmployees = updates;
```

- This implementation also applies to all mutbale methods when exceptions may be thrown. 

## Item 40: Distinguish Early from Deferred Execution
**Declarative code** is expository: it defines what gets done.
**Implerative code** details step-by-step instructions that explain how something gets done. 

Imperative:
```
var answer = DoStuff(Method1(), Method2(), Method3());
```
In this code, the following steps are taken:
1. It calls Method1 to generate the first parameter to DoStuff()
2. It calls Method2 to generate the second parameter to DoStuff()
3. It calls Method3 to generate the third parameter to DoStuff()
4. It calls DoStuff with the three calculated parameters

**Deferred exeuction**, in which you use lambdas and query expressions, change the process.

```
var answer = DoStuff(() => Method1(),
    () => Method2(),
    () => Method3());
```
At runtime, it does the following:
1. It calls DoStuff(), passing the lambda expressions that could call Method1, Method2, and Method3
2. Inside DoStuff, if and only if the result of Method1 is needed, Method1 is called.
3. Inside DoStuff, if and only if the result of Method1 is needed, Method2 is called.
4. Inside DoStuff, if and only if the result of Method1 is needed, Method3 is called.
5. Method1, Method2, and Method3 may be called in any order, as many times (including zero) as needed.

- Since the imperative model always calls all three methods, any side effects from any of those methods always occur exactly once. In contrast, the declarative model may or may not execute all or any of those methods. 

- The important point in deciding between early and late evaluation is the semantics you want to achieve. If (and only if) the objects and methods are immutable, then the correctness of the program is the same when you replace a value with the function that calculates it, and vice versa. 

Sometimes, you may find that caching provides the most efficiency:
```
var cache = Method1();
var answer = DoStuff(() => cache,
    () => Method2(),
    () => Method3());
```

## Item 41: Avoid Capturing Expensive Resources

- When you capture a variable in a closure, the object referenced by that variable can have its lifetime extended. It is not garbage until the last delegate referncing that captured variable becomes garbage. The implication is that you don't know when local variables go out of scope if you return something that is represented by a delegate using a captured variable. 

The following code: 
```
var counter = 0;
var numbers = Extensions.Gnerate(30, () => counter++);
```
generates code that looks something like this:
```
private class Closure 
{
    public int generatedCounter;
    public int generatorFunc() => 
        generatedCounter++;
}
// usage
var c = new Closure();
c.generatedCounter = 0;
var sequence = Extensions.Generate(30, new Func<int>(
    c.generatorFunc));
```
The hidden nested class members have been bound to delegates used by `Extensions.Generate`. This can affect the lifetime of the hidden object, and therefore affect when any of the members are eligible for garbage collection. 

```
public IEnumerable<int> MakeSequence()
{
    var counter = 0;
    var numbers = Extensions.Generate(30, ()=> counter++);
    return numbers;
}
```
In this code, the returned object uses the delegate that is bound by the closure. Because the return value needs the delegate, the delegate's lifetime extends beyond the activation of the method. 

## Item 42: Distinguish between IEnumerable and IQueryable Data Sources

- `IQueryable<T>` and `<IEnumerable<T>` have very similar API signatures. `IQueryable<T>` derives from `IEnumerable<T>`. 

The following two query statements are quite different:
```
var q = 
    from c in dbContext.Customers
    where c.City == "London"
    select c;
var finalAnswer = from c in q
    orderby c.Name
    select c;

var q = 
    (from c in dbContext.Customers
    where c.City == "London"
    select c).asEnumerable();
var finalAnswer = from c in q
    orderby c.Name
    select c;
```

These queries return the same result but in different ways. The first query uses the normal LINQ to SQL version that is built on `IQueryable` functionality. The second version forces the database objects into `IEnumerable` sequences and does more of its work locally. 

- Most often, queries work more efficiently if you use `IQueryable` functionality instead of `IEnumerable` functionality. 
- `Enumerable<T>` extension methods use delegates for the lambda expressions as well as the function parameters whenever they appear in query expressions.
- `Queryable<T>` uses expression trees to process those function elements. Also, `IQueryable` providers understand a set of operators, and possibly a defined set of methods, that are implemented in the .NET Framework.

```
private bool isValidProduct(Product p) =>
    p.ProductName.LastIndexOf('C) == 0;

// This works
var q1 = 
    from p in dbContext.Products.AsEnumerable()
    where isValidProduct(p)
    select p;

// This throws an exception 
var q2 =    
    from p in dbContext.Products
    where isValidProduct(p)
    select p;
```

- You can use `AsQueryable()` to convert any `IEnumerable<T>` to an `IQueryable<T>`:
```
public static IEnumerable<Product>
ValidProducts(this IEnumerable<Product> products) =>
    from p in products.AsQueryable()
    where p.ProductName.LastIndexOf('C) == 0
    select p;
```

## Item 43: Use Single() and First() to Enforce Semantic Expectations on Queries

-LINQ can be used for both sequences and returning a single element
- `Single()` returns exactly one element. If no elements exist, or if multiple elements exist, `Single()` throws an exception. 

```
var somePeople = new List<Person> {
    new Person { FirstName = "Bill", LastName = "Gates"},
    new Person { FirstName = "Bill", LastName = "Wagner"}, 
    new Person { FirstName = "Bill", LastName = "Johnson"}
};

// Will throw an exception because more than one element is in the sequence
var answer = (from p in somePeople
    where p.FirstName == "Bill"
    select p).Single();
```
`Single()` immediately evaluates the query and returns the single element and will throw an exception even before examining the result.

The following code wll fail with the same exception (with a different message):

```
var answer = (from p in somePeople
    where p.FirstName == "Larry"
    select p).Single();
```

- If your query can return zero or one element, you can use `SingleOrDefault()`. This will still throw an exception when more than one value is returned.

```
var answer = (from p in somePeople
    where p.FirstName == "Larry"
    select p).SingleOrDefault();
```
This query will return null to indicate that there were no values that matched the query. 

- When you expect to get more than one value but want a specific one, you can use `First()` or `FirstOrDefault()`. Both methods return the first element in the returned sequence and if the sequence is empty, the default is returned. 

```
// Works. Returns null
var answer = (from p in Forwards
    where p.GoalsScored > 0
    orderby p.GoalsScored
    select p).FirstOrDefault();

// throws an exception if there are no values in the sequence: 
var answer2 = (from p in Forwards
    where p.GoalsScored > 0
    orderby p.GoalsScored
    select p).First();
```

- If you know exactly where in the sequence to look, you can use `Skip` and `First` to retrieve the one sought element.

This code finds the third-best goal-scoring forward:
```
var answer = (from p in Forwards
    where p.GoalsScored > 0
    orderby p.GoalsScored
    select p).Skip(2).First();
```

## Item 44: Avoid Modifying Bound Variables

The following code illustrates what happens when you capture variables in a closure and then modify those variables

```
var index = 0;
Func<IEnumerable<int>> sequence = () => Utilities.Generate(30, () => index++);

index = 20;
foreach (int n in sequence())
    WriteLine(n);
WriteLine("Done");
index = 100;
foreach (var n in sequence())
    WriteLine(n);
```
This code prints the numbers from 20 through 50, followed by the numbers 100 through 130. 

- Modifying the bound variables between queries can introduce errors caused by the interaction of deferred execution and the way the compiler implements closures. Therefore, you should avoid modifying bound variables that have been captured by a closure. 