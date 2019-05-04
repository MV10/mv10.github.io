---
title: Parsing for Encapsulated Enumeration Classes
tags: vs2017 .net .netcore reflection
header:
  image: "/assets/2019/05-04/header1280.jpg"
  teaser: "/assets/2019/04-04/header1280.jpg"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2019/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2019/mm-dd/pic.jpg)
---

Easier conversion of raw values to encapsulated enumeration objects.

<!--more-->

Last year I wrote a short article called [Serialization for Encapsulated Enumeration Classes]({{ site.baseurl }}{% post_url 2018-08-07-serialization-encapsulated-enumeration-classes %}) in which I described a technique I use to manipulate code/description pairs in ways that are similar to `enum` but without the many limitations of a real `enum`. Towards the end of the article I showed this bit of ugly code to demonstrate how to parse a raw code value:

```c#
string EliteDataValue = QueryEliteFlag(customerId); // returns Y or N
var EliteStatus = (EnumYesNo)EnumYesNo.Undefined.ReadJson(EliteDataValue);
``` 

I was never satisfied with that. `ReadJson` returns an `object` so the cast is unavoidable, and it's an instance method, so we need to choose one of the instances declared in the derived class (in this case I chose `Undefined`), and of course, we aren't actually reading JSON data. The example was unsatisfying, but more importantly, I strongly disliked leaving these oddities in production code. When I wrote the article, my real-world usage involved a lot of JSON as input data, but I later began using the encapsulation classes with raw code values. When you add additional real-world factors like casting `DataRow` lookups, the result is downright embarrassing:

```c#
var EliteStatus = (EnumYesNo)EnumYesNo.Undefined.ReadJson((string)row["EliteStatus"]);
``` 

Clearly a dedicated parser method was needed.

Code for this article has been added to the code for the earlier article, still available from GitHub at [MV10/Serializing.Encapsulated.Enumerators](https://github.com/MV10/Serializing.Encapsulated.Enumerators).


## Better, Not Great

My plan was to implement a static `Parse` method in the `Enumeration<T>` base class, which would be used like this:

```c#
string code = "Y";
var parsed = EnumYesNo.Parse(code);
```

Unfortunately that isn't possible. This is as close as I could get:

```c#
string code = "Y";
var parsed = EnumYesNo.Parse<EnumYesNo>(code);
```

I can live with "a little weird" versus "nearly incomprehensible".

## What Doesn't Work

I had thought I could use the `DeclaringType` reflection property, but static method calls are optimized in ways that prevent that from working. When a derived class calls a static method declared in the base class, the compiler output is a call made directly against the base class. There is no information at runtime which can be reflected in order to determine the derived class from that base class method. Consider this:

```c#
public class MyBaseClass
{
    public static string WhoAmI() 
        => MethodBase.GetCurrentMethod().DeclaringType.Name;
}

public class MyDerivedClass : MyBaseClass
{ }

MyBaseClass.WhoAmI();
MyDerivedClass.WhoAmI();
```

Both calls to `WhoAmI()` return `MyBaseClass` because the compiled code looks like this:

```
IL_0001:  call        MyBaseClass.WhoAmI
IL_0006:  nop         
IL_0007:  call        MyBaseClass.WhoAmI
```

Reading `DeclaringType` from the static `Parse` method I wanted to write only yielded `Enumeration<T>` rather than `EnumYesNo` or whatever other derived type that I actually needed. The solution was to pass the desired type to the `Parse` method.

## The Compromise

Now the extra type delcaration makes sense:

```c#
// as written in the source code:
parsed = EnumYesNo.Parse<EnumYesNo>(code);

// as compiled, calling the base class:
parsed = Enumeration<string>.Parse<EnumYesNo>(code);
```

The desired type is passed as a generic type parameter which allows us to declare the same return type, which at least avoids the need to cast from an `object`. The method itself then uses the `GetAll<>` method already in the base class to locate a match. I also added a parser for the `Description` property:

```c#
public static E Parse<E>(T code) where E : Enumeration<T>, new()
    => GetAll<E>()
        .Where(n => n.Code.Equals(code))
        .FirstOrDefault();

public static E ParseDescription<E>(string description) where E : Enumeration<T>, new()
    => GetAll<E>()
        .Where(n => n.Description.Equals(description, StringComparison.OrdinalIgnoreCase))
        .FirstOrDefault();
```

The `Description` property is declared as a string, so this allows us to specify a case-insensitive comparison. Unfortunately, the `Code` property is of generic type `T` so the `Equals` method has no case-sensitivity override. As written above, `Parse` requires an exact match: the sample `EnumYesNo` implementation can parse `Y` but not `y`.

The solution is to conditionally cast the `code` argument and `Code` properties when `T` is a string:

```c#
public static E Parse<E>(T code) where E : Enumeration<T>, new()
    => GetAll<E>()
        .Where(n => 
            (typeof(T) == typeof(string)) ? 
            (code as string).Equals(n.Code as string, StringComparison.OrdinalIgnoreCase) 
            : n.Code.Equals(code))
        .FirstOrDefault();
```

Incidentally, direct casting such as `(string)code` will not compile. You will get an error message stating that `T` can't be converted to a string, because the compiler can't guarantee the conversion will always work. However, using the `as` keyword allows for the possibility of a `null` result, which can be short-circuited, and so the compiler accepts this. At runtime, of course, the type-checking condition guarantees that the conversion will work, and we get the desired results:

```xml
Raw value:
  y
Parsed enum:
  Code: Y
  Description: Yes


Raw value:
  no
Parsed enum:
  Code: N
  Description: No
```

## Conclusion

Adding a simple value-parser required jumping through a lot more hoops than I initially expected, but the cleaner code was well worth the effort.

I hope you find it useful!
