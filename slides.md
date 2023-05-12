---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
#background: https://source.unsplash.com/JBrfoV-BZts/1920x1080
# apply any windi css classes to the current slide
#class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Immutable Collections
  Learn about .NET Immutable Collections.

  Also includes read-only interfaces, and how they both differ.
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
#css: 'windicss'
layout: cover
---

# Immutable Collections

### *By: Mayoor Pipalia*

---
layout: center
---

# Immutable
## *What do we mean by Immutable?*

Namespace `System.Collections.Immutable`
 * .NET Core and 5+
 * Nuget Package: <a href="https://www.nuget.org/packages/System.Collections.Immutable">System.Collections.Immutable</a>

 * Example classes
   ```csharp
   ImmutableList<T>
   ImmutableDictionary<TKey,TValue>
   ImmutableHashSet<T>
   ```

<div v-click class="mt-8">

### Before Immutables, there was...

</div>

---
layout: center
---

# Read-Only
### Classes
```csharp
ReadOnlyCollection<T> // .NET 2.0+
ReadOnlyDictionary<TKey,TValue>
```

### Interfaces (.NET 4.5+)
```csharp
IEnumerable<T>
IReadOnlyCollection<T>
IReadOnlyDictionary<TKey,TValue>
IReadOnlyList<T>
IReadOnlySet<T> // .NET 5.0+
```

---
layout: default
---

# Read-Only Classes
## What were they thinking!?

```csharp
ReadOnlyCollection<T> : IList<T>
```

This compiles!
```csharp {all|3|5|7-10|4}
void Func1()
{
  var list = new ReadOnlyCollection<int>(new[] { 1, 2, 3 });
  //list.Add(4);
  Func2(list);
}
void Func2(IList<int> list)
{
  list.Add(4); // throws exception
}
```

<div v-after>

`IList<T>` is explicitly implemented, which is why you can't call `.Add()` directly.

</div>

---
layout: default
---

# Read-Only Interfaces
## Better!

```csharp
class List<T> : IReadOnlyList<T> { }

interface IReadOnlyList<T> {
  int Count { get; }
  T this[int index] { get; }
  IEnumerator<T> GetEnumerator(); // from IEnumerable<T>
}
```

No longer compiles:
```csharp
void Func1()
{
  IReadOnlyList<int> list = new List<int> { 1, 2, 3 };
  Func2(list); // compilation error
}
void Func2(IList<int> list)
{
  list.Add(4);
}
```

---
layout: default
---

# Problem 1 - Cast

Compiles and runs successfully!
```csharp {all|8-9}
void Func1()
{
  IReadOnlyList<int> list = new List<int> { 1, 2, 3 };
  Func2(list);
}
void Func2(IReadOnlyList<int> list)
{
  var editable = (List<int>)list;
  editable.Add(4);
}
```

<div v-after>

No warnings or suggestions if cast if valid.

</div>
<br />
<v-click>

Runtime exception (*Unable to cast object...*):
```csharp
IReadOnlyList<int> list = new[] { 1, 2, 3 };
```

</v-click>

---
layout: default
---

# Solution for #1 - Custom Class

```csharp
class MyReadOnlyList<T> : IReadOnlyList<T> {
  private List<T> TheList;
  public MyReadOnlyList(List<T> list) {
    TheList = list;
  }

  // IReadOnlyList implementation
  public int Count => TheList.Count;
  public T this[int i] => TheList[i];
  public IEnumerator<T> GetEnumerator() => TheList.GetEnumerator();
}
```

Pros:
 * Impossible to cast to an editable list

Cons:
 * You have to write these classes
 * Remember to use these classes

---
layout: default
clicks: 3
---

# Problem 2 - Caller Can Modify List

```csharp {1-9|3,6,8|8,11-14|all}
class Manager {
  private IReadOnlyList<string> TheList;
  private int LastIndex;
  public Manager(IReadOnlyList<string> list) {
    TheList = list;
    LastIndex = TheList.Count - 1;
  }
  public string Last() => TheList[LastIndex];
}

var list = new List<string> { "one", "two", "three" };
var mgr = new Manager(list);
list.RemoveAt(2);
mgr.Last(); // exception
```

<br />
<div v-click="1">

`Manager` keeps track of the last index.

</div>
<div v-click="2">

Caller modifies list, and then calls `Last()`. Exception thrown.

</div>
<div v-click="3">
Read-Only doesn't mean cannot be modified (aka immutable).
</div>

---
layout: default
clicks: 3
---

# Solution for #2 - Wrap + Copy

```csharp {1-7|9-11|12-13|all}
class MyReadOnlyList<T> : IReadOnlyList<T> {
  private List<T> TheList;
  public MyReadOnlyList(IEnumerable<T> list) {
    TheList = list.ToList();
  }
  // IReadOnlyList implementation...
}

var list = new List<string> { "one", "two", "three" };
var mylist = new MyReadOnlyList<string>(list);
var proc = new Processor(mylist);
list.RemoveAt(2); // doesn't modify mylist
proc.Last(); // works
```

<br />
<div v-click="1">

Create `MyReadOnlyList` and pass it in.

</div>
<div v-click="2">

Caller can modify `list`, but doesn't affect `mylist`.

</div>
<div v-click="3">

Problems: copying entire list, two lists, remember to use `MyReadOnlyList`.
</div>

<!-- Not technically a problem, just a misunderstanding of what IReadOnly is. -->

---
layout: default
clicks: 4
---

# Problem 3 - Atomic Changes

We are doing patient upload to meters, using a list that contains up-to-date patients.
```csharp {1-4,15-16|2,4-6|7,15-16|9-10,12|10-12,15-16}
class Patient(string primaryId, string visitId);
static volatile IReadOnlyList<Patient> Patients;

void Thread1_ManagePatients() {
  List<Patient> patients = GetInitialPatients(); // includes patient "abc"
  Patients = patients;
  // Sync occurs

  // Some time later...
  patients.Remove(new Patient("abc", "123"));
  // Sync occurs
  patients.Add(new Patient("abc", "789"));
}

// Send patients to meter, using Patients property
void Thread2_SyncMeter() { }
```

<ul>
  <li v-click="2">First sync occurs. Meter gets patient <i>abc</i></li>
  <li v-click="3">Patient list is updated, where patient <i>abc</i> visit ID changes to <i>789</i></li>
  <li v-click="4">Meter doesn't have patient <i>abc</i> anymore because it read the list in-between the updates</li>
</ul>

---
layout: default
---

# Solution for #3 - Reader Writer Lock

```csharp {1,5,9,13-15|all}
static volatile ReaderWriterLockSlim RWLock = new ReaderWriterLockSlim();
void Thread1_ManagePatients() {
  // ...
  // Some time later...
  RWLock.EnterWriteLock();
  patients.Remove(new Patient("abc", "123"));
  // Sync occurs
  patients.Add(new Patient("abc", "789"));
  RWLock.ExitWriteLock();
}

void Thread2_SyncMeter() {
  RWLock.EnterReadLock();
  // Determine which patients to send to meter
  RWLock.ExitReadLock();
}
```

Cons
 * Writer blocked until all readers done, readers blocked until writer is done
 * Error prone. Forget to call the Exit method: deadlock.
 * May need to optimize code to reduce holding on to lock

---
layout: default
---

# Solution for #3 - Copy List

```csharp {4,8|all}
void Thread1_ManagePatients() {
  // ...
  // Some time later...
  patients = new List<Patient>(patients);
  patients.Remove(new Patient("abc", "123"));
  // Sync occurs
  patients.Add(new Patient("abc", "789"));
  Patients = patients;
}
```

Copy the list, modify it, replace the old list.

Pros
 * No locks needed
 * No lock optimizations needed

Cons
 * Lot more CPU and memory usage

---
layout: center
---

# Solution for all? Immutables!

---
layout: default
clicks: 3
---

# Create and Modify Immutable

```csharp {1-3|3-6|3,4,8-10|all}
using System.Collections.Immutable;

var list_a = ImmutableList.Create<int>(1, 2);
var list_b = list_a.Add(3);
Console.WriteLine(list_a.Count); // 2
Console.WriteLine(list_b.Count); // 3

var list_c = list_b.RemoveRange(new[] { 2, 3 });
Console.WriteLine(list_b.Count); // 3
Console.WriteLine(list_c.Count); // 1
```

<ul>
  <li v-click="0">Creating takes an initial list of values</li>
  <li v-click="1">Adding to the existing list creates a new list</li>
  <li v-click="2">Removing also creates a new list</li>
</ul>

---
layout: default
---

# Characteristics

 * Any modifications creates a new object
 * Original object remains unchanged (i.e. immutable)
 * Thread safe
   * Impossible to break thread safety
   * No locks used - in contrast to `System.Collections.Concurrent`
 * Atomic Changes

---
layout: default
---

# New Change == New Object
## It's not all rainbows and butterflies

<br />
Make some changes:
```csharp
var list_z = list_b.Remove(1).Remove(2).Add(4);
```

<br />
What's really happening:

```csharp
var list_x = list_b.Remove(1);
var list_y = list_x.Remove(2);
var list_z = list_y.Add(4);
```

Every little change we make creates a new list.

<div v-click class="mt-8">

### What can we do about this...

</div>

---
layout: default
clicks: 3
---

# Builder
## Do all changes in a batch (and be atomic too)

<br />
Back to our patient list sync
```csharp {1-3|5-10|8|all}
ImmutableList<Patient> patients = GetInitialPatients(); // includes patient "abc"
Patients = patients;
// Sync occurs

// Some time later...
var builder = Patients.ToBuilder();
builder.Remove(new Patient("abc", "123"));
// Sync occurs
builder.Add(new Patient("abc", "789"));
Patients = builder.ToImmutable();
```

<ul>
  <li v-click="0">Create the initial list, except Immutable this time</li>
  <li v-click="1">Create builder, modify the list, create immutable</li>
  <li v-click="2">Sync sees original list</li>
</ul>

---
layout: default
---

# Performance
## More Features means More Resources

<br />

Implemented as a **Balanced Binary Trees**

Performance tests from MSDN Magazine (10 million Int64s)

|                 | **List** | **ImmutableList** | **ImmutableList.Builder** |
|-----------------|----------|-------------------|---------------------------|
| **Append**      | 0.2s     | 12s               | 8s                        |
| **Prepend**     | Slow     | 13.3s             | 7.4s                      |
| **Memory Size** | 128MB    | 480MB             | 480MB                     |


---
layout: default
clicks: 2
---

# Immutable Object?
## What about the objects in the collection?

<br />

```csharp {1-10|12-13|all}
class Patient
{
  public string PrimaryId { get; set; }
  public string VisitId { get; set; }
  public Patient(string primaryId, string visitId)
  {
    PrimaryId = primaryId;
    VisitId = visitId;
  }
}

var patients = ImmutableList.Create(new Patient("abc", "123"));
patients[0].VisitId = "789";
```

<ul>
  <li v-click="0">Public getters and setters</li>
  <li v-click="1">Immutable Collections don't automatically make the items immutable</li>
</ul>

---
layout: default
---

# Manual Immutable
## Make all properties read-only

<br />

```csharp
class Patient
{
  public string PrimaryId { get; }
  public string VisitId { get; }
  public Patient(string primaryId, string visitId)
  {
    PrimaryId = primaryId;
    VisitId = visitId;
  }
}

var abc = new Patient("abc", "123");
var modified = new Patient(abc.PrimaryId, "789");
```

 * Constructor is only way to set the properties
 * Long constructor definition
 * Difficult to make modifications

---
layout: default
---

# Init Setter
## C# 9.0 (.NET 5) introduced init only setters

<br />

```csharp
class Patient
{
  public string PrimaryId { get; init; } = "";
  public string VisitId { get; init; } = "";
}
var abc = new Patient() {
    PrimaryId = "abc",
    VisitId = "123"
};
var modified = new Patient() {
    PrimaryId = abc.PrimaryId,
    VisitId = "789"
};
```

 * Initialization requires stating each property
 * Difficult to make modifications
 * All properties are optional
   * C# 11.0 adds a `required` modifier: `public required string PrimaryId {get;init;}`

---
layout: default
clicks: 3
---

# Record Type
## C# 9.0 also introduced record types

<br />

```csharp {1|1-6|7|all}
record Patient(string PrimaryId, string VisitId);

var abc = new Patient("abc", "123");
var modified = abc with {
    VisitId = "789"
};
var clone = abc with { };
```

<ul>
  <li v-click="0">Define a record type</li>
  <li v-click="1">Easy creation and modifications</li>
  <li v-click="2">Easy to clone</li>
</ul>

---
layout: default
clicks: 4
---

# More Record Features
## More Fun with Records

<br />

```csharp {1-4|6|8|10|all}
record Patient(string PrimaryId, string VisitId);

var abc = new Patient("abc", "123");
var clone = new Patient("abc", "123");

Console.WriteLine(abc == clone); // True

Console.WriteLine(abc); // Patient { PrimaryId = abc, VisitId = 123 }

(string primary, string visit) = clone;
```

<ul>
  <li v-click="1">Builtin Value-based Equality</li>
  <li v-click="2">Builtin ToString implementation</li>
  <li v-click="3">Builtin Deconstructor</li>
</ul>

---
layout: default
---

# When?
## When to use Immutable Collections?

<br />

 * Niche use-case
 * Large collection of data stored in memory
 * Modifying it regularly
 * Single writer, multiple readers

---
layout: default
---

# References

 * <a href="https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable">System.Collections.Immutable (.NET Reference)</a>
 * <a href="https://learn.microsoft.com/en-us/archive/msdn-magazine/2017/march/net-framework-immutable-collections">MSDN Magazine Article about Immutable Collections</a>
   * Goes over implementation detail
 * <a href="https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record">Records (C# Reference)</a>
