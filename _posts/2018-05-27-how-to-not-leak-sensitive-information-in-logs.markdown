---
layout: post
title: "How to Not Leak Sensitive Information in Logs, Regex Performance and More"
date: 2018-05-26
categories:
  - .NET
description:
image: https://picsum.photos/g/2000/1200?image=1068
image-sm: https://picsum.photos/g/500/300?image=1068
---

As I get [Transfer]({{"/projects" | absolute_url}}) ready for shipping, I need to add the boring necessities such as metrics and logging. Thanks to .NET Core 2.0, there is already some great out of the box logging. However, after the recent blunders by [Github](https://www.zdnet.com/article/github-says-bug-exposed-account-passwords/) and [Twitter](https://www.theverge.com/2018/5/3/17316684/twitter-password-bug-security-flaw-exposed-change-now), I had to be pretty careful and think about what information what actually being logged. 

The main endpoint of the entire server was in `/files/{id}/{*path}`, where `id` is the guid of the connection and `path` was the path on the remote server to perform the action on. .NET Core was logging the server paths browsed to, a huge security flaw! 

Unfortunately for me, the logging code for this is buried in the interals of the framework and is difficult to customize. After lengthy googling, I decide to look for an alternative avenue to pursue. I investigated a way to somehow intercept the log before it got sent to the console. Luckily, with a simple override, it's possible to intercept calls to `Console.Write()` before they're written to the screen. My original code looked as follows:

~~~ csharp
  public class ConsoleLoggingSanitizer : TextWriter
  {
      public override Encoding Encoding => Encoding.UTF8;

      public override void Write(string value)
      {
          base.Write(value);
      }

      public override void WriteLine(string value)
      {
          base.WriteLine(value);
      }
  } 

  Console.SetOut(new ConsoleLoggingSanitizer())
~~~

After running, I promptly saw the program crash with a `StackOverflowException`, something was off. As it turns out, and what I didn't realise straight away, was that I had to proxy the call back to the console writer. So the class became:

~~~ csharp
public class ConsoleLoggingSanitizer : TextWriter
{
    public override Encoding Encoding => Encoding.UTF8;
    private TextWriter _passthroughWriter;

    public ConsoleLoggingSanitizer(TextWriter passthrough)
    {
        _passthroughWriter = passthrough;
    }

    public override void Write(string value)
    {
        _passthroughWriter.Write(value);
    }

    public override void WriteLine(string value)
    {
        _passthroughWriter.WriteLine(value);
    }
}
~~~

Which worked! And luckily, all the calls that the built-in logger was making, were all sent through the `Write(string)` method. As there were only a few logging calls to be made, I opted to use Regex's to parse and replace the logging calls. As Regex's are rather costly, I first ensured that the text even needed to be replaced. 

The two logging lines in question look like this:

~~~
info: Microsoft.AspNetCore.Hosting.Internal.WebHost[1]
      Request starting HTTP/1.1 GET http://localhost:5000/files/16134402-f69b-480a-a90d-0039bdcb28a6/path/to/confidential/file/ x-www-form-urlencoded

info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[1]
      Executing action method Transfer.Foo.Controllers.Files.FilesController.GetId (Transfer.ServerCore) with arguments (16134402-f69b-480a-a90d-0039bdcb28a6, path/to/confidential/file/, False, ) - ModelState is Valid
~~~

These can easily be parsed with a regex, and use a capturing to find and replace the sensitive string. The code originally was this:

~~~ csharp
public override void Write(string value)
{
    string[] candidates = { @"FooController.Get", "/files/" };
    if (candidates.Any(value.Contains))
    {
        string[] regexes = 
        {
            @"FilesController\.GetId \(Transfer\.ServerCore\) .+ \([0-9a-f\-]+, ([^,]+)",
            @"Request starting HTTP\/1\.1 GET http:\/\/localhost:5000\/files\/[0-9a-f\-]+\/(.+(?= application))"
        };

        var sanitized = value;
        foreach (var regex in regexes)
        {
            sanitized = Regex.Replace(sanitized, regex, m => m.Groups[0].Value.Replace(m.Groups[1].Value, "*****"));
        }
        value = sanitized;
    }

    _passthroughWriter.Write(value);
}
~~~

There are 4 different ways to handle the regex replacement, either using a compiled or uncompiled regex, and either using the `MatchGroup` delegate, or a replacement string. This corresponds to the following code:

~~~ csharp
var regex = new Regex(@"FilesController\.GetId \(Transfer\.ServerCore\) .+ \([0-9a-f\-]+, ([^,]+)", RegexOptions.Compiled);
var a = regex.Replace(input, m => m.Groups[0].Value.Replace(m.Groups[1].Value, "*****"));
var b = regex.Replace(input, "$`******$'");
var c = Regex.Replace(input, @"FilesController\.GetId \(Transfer\.ServerCore\) .+ \([0-9a-f\-]+, ([^,]+)", m => m.Groups[0].Value.Replace(m.Groups[1].Value, "*****"));
var d = Regex.Replace(input, @"FilesController\.GetId \(Transfer\.ServerCore\) .+ \([0-9a-f\-]+, ([^,]+)", "$`******$'");
~~~

By using the excellent library [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet), we can see exactly which one is faster! Unfortunately, the results are pretty uninteresting.

~~~
                              Method |     Mean |     Error |    StdDev | Rank |
------------------------------------ |---------:|----------:|----------:|-----:|
          ReplaceCompiledGroupAction | 7.271 us | 0.0824 us | 0.0771 us |    3 |
    ReplaceCompiledReplacementString | 6.584 us | 0.0948 us | 0.0887 us |    1 |
       ReplaceNonCompiledGroupAction | 7.551 us | 0.1045 us | 0.0927 us |    4 |
 ReplaceNonCompiledReplacementString | 6.699 us | 0.0813 us | 0.0760 us |    2 |
~~~

Another thing I was interested in was the actual performance overhead of the sanitizer, loading up the server into dotTrace, we can analyse it. This first code, over a 60 second sample, took up 40ms of time. Not bad, but I feel I can do better. 

After refactoring the code to use a Dictionary and breaking early to avoid unnecessarily parsing lines (it can be assumed only one regex will be replaced per line):

~~~ csharp
private static string BaseUrl => Regex.Escape(Program.BaseUrl);
private static Dictionary<string, string> candidates = new Dictionary<string, Regex>
{
    { "FilesController.GetId", @"FilesController\.GetId \(Transfer\.ServerCore\) .+ \([0-9a-f\-]+,([^,]+)" },
    { "/files/", $@"Request starting HTTP\/1\.1 GET {BaseUrl}\/files\/[0-9a-f\-]+\/(.+(?= application))" }
};

public override void Write(string value)
{
    var sanitized = value;
    foreach (var candidate in candidates)
    {
        if (value.Contains(candidate.Key))
        {
            sanitized = Regex.Replace(value, candidate.Value, m => m.Groups[0].Value.Replace(m.Groups[1].Value, "*****"));
            break;
        }
    }

    _passthroughWriter.Write(sanitized);
}
~~~

Over a 70s period, this resulted in 34ms of time, which is about a 28% improvement. Interestingly, the breakdown of time drew my attention.

<figure>
  <img src="https://s3-ap-southeast-2.amazonaws.com/dylanwragge-blob/dylanwragge-com/blog2-subsystem1.JPG" alt="Original Code Subsystem Breakdown"/>
  <figcaption>The breakdown of the original code, the two methods are Regex.LookupCachedAndUpdate and Regex.Replace</figcaption>
</figure>

<figure>
  <img src="https://s3-ap-southeast-2.amazonaws.com/dylanwragge-blob/dylanwragge-com/blog2-subsystem2.JPG" alt="Second Code Breakdown"/>
  <figcaption>The breakdown of the second code, note the different methods called, RegexParser.ScanRegex and Regex.Replace</figcaption>
</figure>


This makes it seem that a significant portion of the time is spend recompiling the regexes over and over. By changing the regex's to be compiled, we reduce the time to 20ms over 68s, a 40% improvement from the original! The breakdown is much better, too:

<figure>
  <img src="https://s3-ap-southeast-2.amazonaws.com/dylanwragge-blob/dylanwragge-com/blog2-subsystem3.JPG" alt="Final Code Breakdown"/>
  <figcaption>The final code breakdown, a huge improvement. Now, the regex parsing or replacing doesn't even register.</figcaption>
</figure>