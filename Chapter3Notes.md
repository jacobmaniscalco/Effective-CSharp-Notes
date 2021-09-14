# Working with Generics

- The C# compiler takes the C# code and create the Microsoft Intermdediate Languarge (MSIL, IL) definition for the generic type. In contrast, the JIT compiler combines a generic type definition with a set of type parameters to create a closed generic type. The CLR supports both concepts at runtime. 
- Generic class definitions are fully compiled MSIL types. The code they contain must be completely valid for any type parameters that may be used. The generic definition is called a **generic type definition**. A specific instance of a generic type is called a **closed generic type** (if only some of the parameters are specified, it's called an **open generic type**).
- With nongeneric types, there is a 1:1 correspondence between the IL for a class and the machine code created. 
- The JIT compiler creates one machine version of a generic class  for all reference types.

All these instantiations share the same code at runtime:
```
List<string> stringList = new List<string>();
List<Stream> OpenFiles = new List<Stream>();
List<MyClassType> anotherList = new List<MyClassType>();
```

- Different rules apply to closed generic types that have at least one value type usd as a type parameter. The JIT compiler creates a different set of machine instructions for different type parameters. Therefore, the following three closed generic types have different machine code pages:
```
List<double> doubleList = new List<double>();
List<int> markers = new List<int>();
List<MyStruct> values = new List<MyStruct>();
```
- Creating generic methods instead of generic classes can lmit the amount of extra IL code created for each seperate instantiation. Only those methods actually referenced will be instantiated. Generic methods defined in a nongeneric class are not JIT-compiled. 

## Item 18: Always Define Constraints That Are Minimal and Sufficient

**Constraints** enable the compiler to expect capabilities in a type parameter beyond those in the public interfance defined in System.Object. 
- You use constraints to communicate (to both the compiler and developers) any assumptions you've made about the generic types. Constraints communicate to the compiler that your generic type expects functionality beyond `System.Object`'s public interface. 
- This communication help the compler in two ways:
    1. It helps when you create your generic type: the compiler asserts that any generic type parameter contains the capabilities you specified in your contrainsts.
    2. The compiler ensures that nayone using your neric type defines type parameters that meet your specifications.

The following generic method does not declare any constraints on `T`, so therefore it must check for the presence of the `IComparable<T>` interface before using those methods.

```
public static bool AreEqual<T>(T left, T right)
{
    if (left == null)
        return right == null;
    if (left is IComparable<T>)
    {
        IComparable<T> lval = left as IComparable<T>;
        if (right is IComparable<T>)
            return lval.CompareTo(right) == 0;
        else
            throw new ArgumentException(
            "Type does not implement IComparable<T>",
            nameof(right));
    }
    else // failure
    {
        throw new ArgumentException(
        "Type does not implement IComparable<T>",
            nameof(left));
    }
}
```
The following code is much simpler if you specify that `T` must implement `IComparable<T>`:

```
public static bool AreEqual2<T>(T left, T right)
    where T: IComparable<T> =>
    left.CompareTo(right) ==0;
```
- This second version trades runtime errors for compile-time errors. 
    
## Item 19: Specialize Generic Algorithms Using Runtime Type Checking

The following code provides a reverse-roder enumeration on a sequence of items:
```
public sealed class ReverseEnumerable<T> : IEnumerable<T>
{
 private class ReverseEnumerator : IEnumerator<T>
    {
        int currentIndex;
        IList<T> collection;
        public ReverseEnumerator(IList<T> srcCollection)
        {
          collection = srcCollection;
          currentIndex = collection.Count;
        }
        //  IEnumerator<T> Members
        public T Current => collection[currentIndex];
        // IDisposable Members
        public void Dispose()
        {
            // no implementation but necessary
            // because IEnumerator<T> implements IDisposable
            // No protected Dispose() needed
            // because this class is sealed.
        }
        //   IEnumerator Members
        object System.Collections.IEnumerator.Current
            => this.Current;
        public bool MoveNext() => --currentIndex >= 0;
        public void Reset() => currentIndex = collection.Count;
    }
 IEnumerable<T> sourceSequence;
    IList<T> originalSequence;
 public ReverseEnumerable(IEnumerable<T> sequence)
    {
        sourceSequence = sequence;
    }
    // IEnumerable<T> Members
    public IEnumerator<T> GetEnumerator()
    {
        // Create a copy of the original sequence,
        // so it can be reversed.
        if (originalSequence == null)
        {
            originalSequence = new List<T>();
            foreach (T item in sourceSequence)
                originalSequence.Add(item);
        }
        return new ReverseEnumerator(originalSequence);
    }
    // IEnumerable Members
    System.Collections. IEnumerator
        System.Collections.IEnumerable.GetEnumerator() =>
            this.GetEnumerator();
}
```
This implementation assumes the least amout of information from its arguments. The constructor assumes that its input parameter supports `IEnumerable<T>` and that's it. 
- Since many of the types that implement `IEnumerable<T>` also implement `IList<T>`, you can improve the efficiency of the code.

The only change is in the constructor of the `ReverseEnumerable<T>` class:
```
public ReverseEnumerable(IEnumerable<T> sequence)
{
    sourceSequence = sequence;
    // If sequence doesn't implement IList<T>,
    // originalSequence is null, so this works
    // fine
    originalSequence = sequence as IList<T>;
}
```
To catch when the compile-time type of a parameter is `IEnumerable<T>` but the runtime implements `IList<T>` you can provide both the runtime check, and the compile-time method overload.
```
public ReverseEnumerable(IEnumerable<T> sequence)
{
    sourceSequence = sequence;
    // If sequence doesn't implement IList<T>,
    // originalSequence is null, so this works
    // fine
    originalSequence = sequence as IList<T>;
}
 public ReverseEnumerable(IList<T> sequence)
{
    sourceSequence = sequence;
    originalSequence = sequence;
}
```
`IList<T>` enables a more efficient algorithm than does `IEnumerable<T>`.

## Item 20: Implement Ordering Relations with `IComparable<T>` and `IComparer<T>`

- `IComparable<T>` defines the natural order for your types and contains one method, `CompareTo()`.
- `IComparer` describes alternative orderings. 

```
public struct Customer : IComparable<Customer>, IComparable
{
    private readonly string name;
    public Customer(string name)
    {
        this.name = name;
    }
    // IComparable<Customer> Members
    public int CompareTo(Customer other) =>
        name.CompareTo(other.name);
    // IComparable Members
    int IComparable.CompareTo(object obj)
    {
        if (!(obj is Customer))
            throw new ArgumentException(
                "Argument is not a Customer", "obj");
        Customer otherCustomer = (Customer)obj;
        return this.CompareTo(otherCustomer);
    }
}
```