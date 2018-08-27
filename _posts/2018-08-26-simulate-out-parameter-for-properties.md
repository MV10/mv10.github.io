---
title: Simulate "out" Parameters for Properties
tags: c# .net
header:
  image: "/assets/2018/08-26/header1280.jpg"
  teaser: "/assets/2018/08-26/header1280.jpg"
---

A short article showing `out`-like syntax for property setters.

<!--more-->

Anybody who has used c# long enough is familiar with the error message, `A property or indexer may not be passed as an out or ref parameter.` This is not valid in c#:

```csharp
class WillNotCompile
{
    string MyProperty { get; set; }

    void Main()
    {
        Concat("abc", "xyz", out MyProperty); // error
    }

    void Concat(string first, string second, out string data)
    {
        data = first + second;
    }
}
```

Every c# developer should understand the problem here: a property setter is really a method, not a variable. Even so, I run into this every few months. Not because I don't know the rule, and not because I don't understand the reason it isn't allowed, but because it _feels_ like syntax that ought to work. Typically you end up using a temporary variable. In fact, a temporary variable for `out` is such a common pattern that Microsoft added inline variable declaration for `out` in c# 7.0:

```csharp
class UglyButItWorks
{
    string MyProperty { get; set; }

    void Main()
    {
        Concat("abc", "xyz", out string temp); // inline temp var
        MyProperty = temp; // temp = "abcxyz"
    }

    void Concat(string first, string second, out string data)
    {
        data = first + second;
    }
}
```

In fact, VB.NET does allow `out` properties, and this is exactly what the VB compiler does under the hood. One of the guiding principles of c# language design is "no surprises," whereas the VB.NET philosophy is "just make it work"... sometimes with not-necessarily-clear tradeoffs (in this case, a bit of performance and memory overhead that was much more important in the early 2000s, and could still be an issue in high-frequency call scenarios).

While adding .NET Core 2.0 configuration support to the _Serilog.Sinks.MSSqlServer_ library, I found myself having to call a utility function to conditionally apply changes to more than 20 properties, and c# 7.0 was not an option. That meant three lines of code just to work around this little issue. There had to be a better way.

## A Better Way

The idea is to use a delegate to set the property. This is pretty similar to `out` in that you're passing a reference. (This could be simplified to eliminate the generic `<T>` and the conversion code if all of the properties in question are of the same type, but the conversion syntax is also useful to demonstrate.)

```csharp
class OutPropertyViaDelegate
{
    string MyProperty { get; set; }

    delegate void PropertySetter<T>(T value);

    void Main()
    {
        Concat("abc", "xyz", (outval) => { MyProperty = outval; }); // outval = "abcxyz";
    }

    static void Concat<T>(string first, string second, PropertySetter<T> setter)
    {
        var data = first + second;
        setter((T)Convert.ChangeType(data, typeof(T)));
    }
}
```

## Conclusion

This isn't a new idea. A quick search turns up discussions about similar approaches dating back at least 13 years. However, I see people asking about it even now, and the accidental use of `out` with a property is common enough in my own day-to-day routine that I thought it was worth writing down. Obviously for just one or two properties the extra code isn't worth it -- just use a temporary variable. But when the property count starts to grow, this is a handy technique to have in your toolbox.
