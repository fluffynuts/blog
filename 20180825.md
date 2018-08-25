As part of the series of posts illuminating [PeanutButter]https://github.com/fluffynuts/PeanutButter), one of the topics I wanted to cover was `PeanutButter.RandomValueGen`, a utility I find it difficult to start testing without (and it turns out that it's infected others too, hopefully making writing tests a little less effort).

`RandomValueGen` started life innocuously enough: I wanted random primitive values. We had an in-house nuget package for doing this, but it also included a bunch of other stuff I didn't want and depended on some other assemblies that I didn't need, so I figured it wouldn't be too hard to come up with my own. Personally, I prefer the use of `using static` for clearer (read: shorter) code,
so one might see me use `RandomValueGen` in tests like so:

```csharp
// ... other imports above
using static PeanutButter.RandomValueGen;

namespace Tests
{
  [TestFixture]
  public class TestSomething
  {
    [Test]
    public void ShouldGetSomeRandomValues()
    {
      var someInt = GetRandomInt();
      var someString = GetRandomString();
      var someDate = GetRandomDate();
      var flag = GetRandomBoolean();
      var money = GetRandomDecimal();
      var dbl = GetRandomDouble();
      // ... proceed with the rest of the test
    }
  }
}
```

Pretty-much all the primatives you'd expect are there, but there are some others which are more tailored to specific situations
```csharp
[Test]
public void MoreRandoms()
{
    var email = GetRandomEmail();
    var url = GetRandomHttpUrl();
    var alpha = GetRandomAlphaString();
    var alphaNumeric = GetRandomAlphaNumericString();
    var hostName = GetRandomHostname();
    var data = GetRandomBytes();
    var fileName = GetRandomFileName();
    var wordsInASentence = GetRandomWords();
    var versionObject = GetRandomVersion();
    var versionString = GetRandomVersionString();
    var enumValue = GetRandomEnum<SomeEnum>();
}
```

All of the above have optional parameters to influence data size (string length, how many bytes, min / max value, etc).

Naturally, the next step would be to be able to generate random _anythings_, eg it would be nice to be able to do:
```csharp
public class Person
{
  public int Id { get; set; }
  public string Name { get; set; }
  public DateTime DateOfBirth { get; set; }
  public bool IsActive { get; set; }
}

[Test]
public void ShouldGetSomeRandomValues()
{
  var person = GetRandom<Person>();
}
```

_and you can_, but let's take a quick diversion onto the other topic I brought up: builders.

The builder pattern is an expressive, fluent pattern for producing a complex object, and can often be seen as something like :

```csharp
[Test]
public void MakeAnAuthorWithABuilder()
{
  var person = PersonBuilder.Create()
    .WithId(42)
    .WithName("Douglas Adams")
    .WithDateOfBirth(new DateTime(1952, 3, 11))
    .WithIsActive(false) // sadly, not any more )':
    .Build();
}

```

And one way to implement that builder might be:

```csharp
public class PersonBuilder
{
  public static PersonBuilder Create()
  {
    return new PersonBuilder();
  }
  
  private readonly List<Action<Person>> _propSetters 
    = new List<Action<Person>>();

  public PersonBuilder WithProp(Action<Person> propSetter)
  {
    _propSetters.Add(propSetter);
    return this; 
  }

  public PersonBuilder WithId(int id)
  {
    return WithProp(o => o.Id = id);
  }

  public PersonBuilder WithName(string name)
  {
    return WithProp(o => o.Name = name);
  }

  public PersonBuilder WithDateOfBirth(DateTime dob)
  {
    return WithProp(o => o.DateOfBirth = dob)
  }

  public PersonBuilder WithIsActive(bool isActive)
  {
    return WithProp(o => o.IsActive = isActive);
  }

  public Person Build()
  {
    var result = new Person();
    foreach (var propSetter in _propSetters)
    {
      propSetter(result);
    }
    return result;
  }
}
```

This seems, of course, not that different from perhaps just using an object initializer:

```csharp
[Test]
public void MakeAnAuthorManually()
{
  var person = new Person()
  {
    Id = 42,
    Name = "Douglas Adams",
    DateOfBirth = new DateTime(1952, 3, 11),
    IsActive = false
  };
}
```

And someone had to write out the builder class. But this is mainly because of the nature of the 
object being built: it's a very simple POCO. The builder may seem like an over-engineered beast too,
but I have my reasons, which will become clearer a little later.

However, if there were lots of times you needed a Douglas Adams in your test code, you might
refactor out a method which incorporates the whole lot, so you could rather do:

```csharp
[Test]
public void MakeAnAuthorWithABuilder()
{
  var person = PersonBuilder.Create()
    .AsDouglasAdams()
    .Build();
}

public class PersonBuilder
{
  // .. assuming that the previous methods exist:
  public PersonBuilder AsDouglasAdams()
  {
    return WithId(42)
             .WithName("Douglas Adams")
             .WithDateOfBirth(new DateTime(1952, 3, 11))
             .WithIsActive(false);
  }
}

```
