## Exception Practices

## Item 45: Use Exceptions to Report Method Contract Failures

- Exceptions are costly at runtime, and writing exception-proof code is difficult. 
- Provide APIs for evleopers to test conditions without writing try/catch blocks everywhere
- Exceptions are the preferred failure-reporting mechanism because they have many advantages over return codes an an error-reporting mechanism. Because exceptions are class types and you can derive your own exception types, you can use exceptions to convey rich information aboout the failure. 
- Using exceptions to report contract failures does not mean that any method that cannot do what you want must exit by throwing an exception. This doesn't mean every failure is an exception. `File.Exists()` returns true if a file exists and false if it doesn't. `File.Open()` throws an exception if the file does not exist.`File.Exists()` satisfies its contract by telling whether or not a file exists. The method succeeds even when the file does not exist. 

Suppose you have a worker class that fails when certain widgets aren't in place. If your API includes only the worker methods but does not provide an alternative path through the code, you'll eoncourage developers to write code like this:
```
// Don't promote this:
DoesWorkThatMightFail worker = new DoesWorkThatMightFail();
try 
{
    worker.DoWork();
}
catch (WorkerException e)
{
    ReportErrorToUser("Test Conditions Failed. Please check widgets.");
}
```

Instead, you should add public methods that enable developers to explicitly check conditions before doing the work:
```
public class DoesWorkThatMightFail
{
    public bool TryDoWork()
    {
        if(!TestConditions())
            return false;
        Work(); // may throw on failures, but unlikely
        return true;
    }

    public void DoWork()
    {
        Work(); // will throw on failures
    }

    private bool TestConditions()
    {
        // body elided
        // Test conditions here
        return true;
    }

    private void Work()
    {
        // elided
        // Do the work here
    }
}
```

This requires you to write four methods: two public and two private. The `TryDoWork()` method valides all input parameters and any internal object state necessary to perform the work. `Work()` is called to perform the task. `DoWork()` simply calls the `Work()` method and lets any failures generate exceptions. This idiom is used in .NET because there are performance implications involved in throwing exceptions, and developers may wish to avoid those costs by testing conditions before allowing methods to fail. Now a developer can test conditions before performing the work:
```
if (!worker.TryDoWork())
{
    ReportErrorToUser("Test Conditions Failed. Please check widgets.");
}
```

## Item 46: Utilize using and try/finally for Resource Cleanup

- Types that use unmanaged system resources should be explicitly released using the `Dispose()` method of the `IDisposable` interface. The best way to ensure that `Dispose()` always gets called is to utilize the using statement or a try/finally block.
- Types that own unmanaged resources defensively create a finalizer for the times a developer forgets to dispose properly, and if forgotten about, the reosources will be freed later when finalizers get their chance to execute. 

Suppose you wrote this code:
```
public void ExecuteCommand(string connString, string commandString)
{
    SqlConnection myConnection = new SqlConnection(connString);
    var mySqlCommand = new SqlCommand(commandString, myConnection);

    myConnection.Open();
    mySqlCommand.ExecuteNonQuery();
}
```

Two disposable objects are not properly cleaned up in this example: `SqlConnection` and `SSqlCommand`. Both objects are in memory until their finalizers are called.
You can fix this problem by calling Dispose when you are finished with the command and the connection:
```
public void ExecuteCommand(string connString, string commandString)
{
    var myConnection = new SqlConnection(connString);
    var mySqlCommand = new SqlCommand(commandString, myConnection);

    myConnection.Open();
    mySqlCommand.ExecuteNonQuery();

    mySqlCommand.Dispose();
    myConnection.Dispose();
}
```

This is fine, unless any exceptions get thrown while the SQL command executes. In that case, the calls to `Dispose()` never happen. The `using` statement ensures that `Dispose()` is called. The C# compiler will generate a try/finall block around each object:

```
public void ExecuteCommand(string connString, string commandString)
{
    using (SqlConnection myConnection = new SqlConnection(connString))
    {
        using (SqlCommand mySqlCommand = new SqlCommand(commandString, myConnection))
        {
            myConnection.Open();
            mySqlCommand.ExecuteNonQuery();
        }
    }
}
```

Whenever you use one `Disposable` object in a function, the using clause is the simplest method to use to ensure that objects get disposed of properly. 

- The `using` statement works only if the compile-time type supports the `IDisposable` interface, so you cannot use it with arbitrary objects. A quick defeinsive as cluase is all you need to safely dispose of objects that might or might not implement `IDisposable`:
```
object obj = Factory.CreateResource();
using (obj as IDisposable)
{
    Consol.WriteLine(obj.ToString());
}
``` 

- Some types support both a Dispose method and a Close method to free resources. `SqlConnection` is one of those classes. You could close `SqlConnection` like this:
```
public void ExecuteCommand(string connString, string commandString)
{
    SqlConnection myConnection = null;

    try 
    {
        myConnection = new SqlConnection(connString);
        SqlCommands mySqlCommand = new SqlCommand(commandString, myConnection);

        myConnection.Open();
        mySqlCommand.ExecuteNonQuery();
    }
    finally
    {
        if (myConnection != null)
            myConnection.Close();
    }

}
```
This version does close the connection, but that's not exactly the same as disposing of it. The `Dispose` method does more than free resources: It also notfies the garbage collector that the object no longer needs to be finalized. `Dispose` calls `GC.SuppressFinalize()`. `Close` typically does not. If you have the choice, `Dispose()` is better than `Close()`.

- `Dispose()` does not remove objects from memory, it is a hook to let objects release unmanaged resources. 

## Item 47: Create Complete Application-Specific Exception Classes

- The first step is to understanding when and why to create new exception classes, and how to construct informative exception hierarchies. 

Each different exception class can have a different set of actions taken:
```
try
{
    Foo();
    Bar();
}
catch (MyFirstApplicationException e1)
{
    FixProblem(e1);
}
catch (AnotherApplicationException e2)
{
    ReportErrorAndContinue(e2);
}
catch (YetAnotherApplicationException e3)
{
    ReportErrorAndShutdown(e3);
}
catch (Exception e)
{
    ReportGenericError(e);
    throw;
}
finally 
{
    CleanupResources();
}
```

- Different catch clauses can exist for different runtime types of exceptions.
- You should consider creating different exception classes only when you believe developers will take different actions for the problems that cause the exception.  
- Exceptions are not for every error condition you encoutner. Throw exceptions for error conditions that cause long-lasting problems if they are not handled or reported immediately. For example, data integrity errors in a database should generate an exception. Writing a throw statement does not mean its time to create a new exception class. 

- When creating your own exception classes, always end the class in `Exception`, always derive the exception class from the `System.Exception` class or some other appropriate exception class. 

- When you create a new exception class, create all four of these constructors:

```
// Default constructor
public Exception();

// create with a message
public Exception(string); 

// Create with a message and an inner exception
public Exception(string, Exception);

// Create from an input stream
protected Exception(SerializationInfo, StreamingContext);
```

- The constructors that take an exception parameter are needed because one of the libraries you use generates an exception, and the code that called the library will get minmal information about the possible corrective actions when you simply pass on the exceptions from the utilities you use:
```
public double DoSomeWork()
{
    // This might throw an exception defined
    // in the third-party library:
    return ThirdPartyLibrary.ImportantRoutine();
}
```

can be fixed:
```
public double DoSomeWork()
{
    try 
    {
        // This might throw an exception defined
        // in the third-party library:
        return ThirdPartyLibrary.ImportantRoutine();  
    }
    catch (ThirdPartyException e)
    {
        var msg = $"Problem with {ToString()}using library";
        throw new DoingSomeWorkException(msg, e);
    }
}
```
This new version createsd more information at the point where the problem is generated. This is called **exception translation**, translating a low-level exception into a higher-level exception that provides more context about the error. 

## Item 48: Prefer the Strong Exception Gurantee

Three exception-safe gunratees:
    1. the basic gurantee
    2. the stronggurantee
    3. the nothrow gurantee

- The basic gurantee states that no resources are leaked and all objects are in a valid state after your exception leaves the function emitting it.
- The strong exception gurantee builds on the basic gurantee and adds that if an exception occurs, the program state did not change.
- The no-throw gurantee states that an operation never fails, from which it follows that a method does not ever throw exceptions
The strong exception gurantee provides the best tradeoff between recovering from exceptions and simplifying exception handling. 

- You get some help on the basic gurantee from the .NET CLR since the environment handles memory management. 

Ways to achieve the strong exception gurantee:
    1. Data elements that your program uses should be stored in immutable value types.
    2. You can use the functional programming style, such as with LINQ queries. That programming style automatically follows the strong exception gurantee.

The general guideline is to perform any data modifications in the following manner:
    1. Make dfensive copies of data that will be modified
    2. Perform any modification to these defensive copies of the data. This includes any operations that might throw an exception.
    3. Swap the temporary copies back to the original. This operation cannot throw an exception.

```
public void PhysicalMove(string title, decimal newPay)
{
    // Payroll data is a struct:
    // ctor will throw if fields aren't valid.
    PayrollData d = new PayrollData(title, newPay, this.payrollData.DateOfHire);

    // if d was constructed properly, swap:
    this.payrollData = d;
}
```

- The envelope-letter pattern hides the implementation (letter) inside a wrapper (envelope) that you share with public clients of your code. 
```
private Envelope data;
public IList<PayrollData> MyCollection
{
    get 
    {
        return data;
    }
}
public void UpdateData()
{
    data.SafeUpdate(UnreliableOperation());
}
```

The Envelope class implements the `IList` by forwarding every request to the contained `List<PayrollData>`:
```
public class Envelope: IList<PayrollData>
{
    private List<PayrollData> data = new List<PayrollData>();

    public void SafeUpdate(IEnumerable<PayrollData> sourceList)
    {
        // make the copy
        List<payrollDate> updates = new List<PayrollData>(sourceList.ToList());
        data = updates;
    }
    public PayrollData this[int index]
    {
        get { return data[index]; }
        set { data[index] = value; }
    }
    public int Count => data.Count;
    
    public bool IsReadOnly =>
        ((IList<PayrollData>)data).IsReadOnly;
    
    public void Add(PayrollData item) => data.Add(item);
    
    public void Clear() => data.Clear();
    
    public bool Contains(PayrollData item) =>
        data.Contains(item);
    
    public void CopyTo(PayrollData[] array, int arrayIndex) =>
        data.CopyTo(array, arrayIndex);
    
    public IEnumerator<PayrollData> GetEnumerator() =>
        data.GetEnumerator();

    public int IndexOf(PayrollData item) =>
        data.IndexOf(item);
    
    public void Insert(int index, PayrollData item) =>
        data.Insert(index, item);
    
    public bool Remove(PayrollData item)
    {
        return ((IList<PayrollData>)data).Remove(item);
    }
    
    public void RemoveAt(int index)
    {
        ((IList<PayrollData>)data).RemoveAt(index);
    }
    
    IEnumerator IEnumerable.GetEnumerator() =>
        data.GetEnumerator();
}
```

## Item 49: Prefer Exception Filters to catch and re-throw

- An exception filter is an expression on your catch clause, following the when keyword, that limits when the catch clause handles an exception of a given type.

```
var retryCount = 0;
var dataString = default(String);

while(dataString == null)
{
    try
    {
        dataString = MakeWebRequest();
    }
    catch(TimeoutException e)when(retryCount++ < 3)
    {
        WriteLine("Operation timed out. Trying again.");
        Task.Delay(1000 * retryCount);
    }
}
```

- Using exception filters can have a positive effect on program performance, as the .NET CLR has been optimized so that the presence of try/catch blocks where the catch cluase is not entered have mnimal effect on runtime performance. 

## Item 50: Leverage Side Effects in Exception Filters

```
public static bool ConsoleLogException(Exception e)
{
    var oldColor = Console.ForegroundColor;
    COnsole.ForegroundColor = ConsoleColor.Red;
    WriteLine("Error: {0}", e);
    Console.ForegroundColor = oldColor;
    return false;
}
```

This code writes out information about any exception to the console. This can be sprinkled anywhere in the code where you want to generate log information. Add a try/catch clause, and apply this false-returning exception filter:
```
try
{
    data = MakeWebRequest();
}
catch (Exception e) when(ConsoleLogException(e)) { }
catch (TimeoutException e) when (falures++ < 10)
{
    WriteLine("Timeout error: trying again");
}
```

- Exception filters with side effects are a great way to observe what's happening when exception are thrown somewhere in your code. In a large codebase, these tools can have a positive impact on finding the root cause of an exception being thrown by a misbehaving application. And once the source is found, fixing it is that much easier. 