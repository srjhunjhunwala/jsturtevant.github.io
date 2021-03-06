---
layout: fwtpost
title: "BigInteger Structure"
date: "2015-04-05"
categories:
- csharp
- .net framework tour
---

Where do you turn to if you have an integer to that is to large for the primitives supplied by the .NET Framework?  The ```BigInteger``` Structure!   The [BigInteger Structure](https://msdn.microsoft.com/en-us/library/system.numerics.biginteger(v=vs.111).aspx) in the ```System.Numerics``` namespace allows for any size signed integer to be represented.  

Unlike the built in primitives such as  ```int```, ```byte```, or ```long```, the ```BigInteger``` structure does not have a lower or upper limit as represented through the *MinValue* and *MaxValue* properties on the primitives.

Like primitives, ```BigInteger``` is immutable but this comes with a caveat... You can get an ```OutOfMemoryException``` if you perform operations on the value that causes it to grow to large.  

```BigInteger``` has [overloaded the standard mathematical operators](https://msdn.microsoft.com/en-us/library/aa288467(v=vs.71).aspx) so that you can do math just like any other primitive numeric type:

{% highlight csharp %}
var x = new BigInteger(1);
var y = new BigInteger(200);

var z = x + y;

Assert.AreEqual(201, z);
{% endhighlight %}  

Creating a number larger than any of the primitive types can be achieved in several ways:

{% highlight csharp %}
// Using a byte array
var bytes = new byte[] {1, 2, 34, 45, 12, 54, 23, 42, 88, 45, 45, 45, 12};
var x = new BigInteger(bytes);

// Math on  a large number
var x = BigInteger.Multiply(Int32.MaxValue, 99999);

// Parsing a string
string googol = "1".PadRight(100, '0');
var x = BigInteger.Parse(googol);
{% endhighlight %}  


## More Information
You find out more about the ```BigInteger``` structure on the [BigInteger MSDN page](https://msdn.microsoft.com/en-us/library/system.numerics.biginteger(v=vs.111).aspx).  Becuase the number is large and immutable, there is information on that page that tells you how to best handle performance problems you might run into.

This type will be available in the cross platform [.NET Core Library](https://github.com/dotnet/corefx).
