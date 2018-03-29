---
layout: post
title: "Creating Type-Safe User Settings in .NET Core"
date: 2018-03-29
categories:
  - .NET
description:
image: https://picsum.photos/2000/1200?image=515
image-sm: https://picsum.photos/500/300?image=515
---

It's a pretty common, even required, feature of any app to have some settings about its operation that the user can control. C# implements a sort of 'settings' by configuration through the `IConfiguration` interface but there are several problems with this. A typical usage of this would look like this: 

~~~ csharp
private IConfiguration _config;
public void Foo()
{
  int option1 = (int)_config["option:foo_option"];
}
~~~

While this interface is read-only, we could implement a settings class that functions in a similar fashion such as:

~~~ csharp
public class Settings
{
  private Dictionary<string, object> _settingsDict = new Dictionary<string, object>();
  public Settings()
  {
    // Load from json or some other store
  }
  public object this[string key]
  {
    get
    {
      return _settingsDict[key];
    }
    set
    {
      _settingsDict[key] = value;
      // Save to file
    }
  }
}
~~~
This would work, and is very simple to use, but it is not very type, or typo, safe method. We have to a) remember the exact name of the setting, b) remember what type it is and c) type it in correctly. Worst of all, we don't get any compile-time warning of anything and doing something wrong will result in an ugly runtime exception.  

We can come up with something more robust, by eliminating the need to remember and use IntelliSense to remember it for us with the power of C# generics and reflection.

The core of this implementation is a class `SettingDefinition` which holds the name, type and a default for the setting.

~~~ csharp
public class SettingDefinition<T>
{
    public string Name { get; }
    public string TypeString => typeof(T).FullName;
    public T Default { get; }

    internal SettingDefinition(string name, T @default)
    {
        Name = name;
        Default = @default;
    }
}
~~~

These setting definitions are then defined in a static class `SettingList`. For example,

~~~ csharp
public static class SettingList
{
    public static SettingDefinition<bool> AskToOverwrite => 
        new SettingDefinition<bool>(nameof(AskToOverwrite), @default: true);
    
    public static SettingDefinition<int> SomeNumberSetting => 
        new SettingDefinition<int>(nameof(SomeNumberSetting), @default: 4);
}
~~~

Using the `nameof` operator allows us to be able to refactor easily and keeps everything consistent.  
To access these settings I define two interfaces `ISettingReader` and `ISettingWriter` which will expose methods to view and update the settings.  

~~~ csharp
public interface ISettingReader
{
    T GetValue<T> (SettingDefinition<T> setting);
    event EventHandler<SettingChangedEvent> SettingChanged;
}

public interface ISettingWriter
{
    void SetValue<T>(SettingDefinition<T> setting, T value);
}
~~~

In Transfer, we are using SQLite as a backend for persisting data to disk. This presents a challenge with our system. We will be representing the settings as a string and need some way to convert between an actual type in C# and a string in SQL. Luckily, C# has powerful reflection tools to allow this.

Our DB Table will look like this:

|---------------+-----------------+------------------|
| Name NVARCHAR | Value NVARCHAR  | Type NVARCHAR    |
|---------------|-----------------|------------------|
| "Setting1"    | "4"             | "System.Int32"   |
| "Setting2"    | "True"          | "System.Boolean" |
|---------------+-----------------+------------------|
{:.mbtablestyle}


When we store the object in the database, we can simply use `Object.ToString()` to get a string representation. To go the other way, we have to do some reflection trickery.

~~~ csharp
public T GetValue<T> (SettingDefinition<T> setting)
{
  string value = GetFromDb(setting.Name);
  var converter = TypeDescriptor.GetConverter(typeof(T));
  var result = (T) converter.ConvertFromString(value);
  return result;
}
~~~

Great! Now we can safely read from databse into a C# object. There is just one more thing before we're done, and that is initializing the database. We always want to have every setting in the database, so we will seed the database on startup, and add any settings that aren't there. For this, we will need more reflection.

~~~ csharp
var settings = typeof(SettingList).GetProperties();
foreach (var setting in settings)
{
    dynamic value = setting.GetValue(null, null);
    string defaultString = value.Default.ToString();
    InsertIntoDb(value.Name, defaultString, value.TypeString);
}
~~~

I'm usually wary of using Reflection as it usually has a huge performance overhead. However, in this case it's quite fast, which I will cover in my next post.