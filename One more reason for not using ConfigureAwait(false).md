# One more reason for not using ConfigureAwait(false)

Applicable for .NET Framework based apps

***

One of my constant mistakes which always emerges on code review is forgetting to use ConfigureAwait(false) while working with asynchronous code. In this article I'm going to outline some basic moments why we should or should not resort to this statement. Additionaly, I'll present a case of misusing it from my own experience and what it cost me.

I won't stop to overview what exactly ConfigureAwait method does as I can't explain it better than [Stephen Toub](https://devblogs.microsoft.com/dotnet/configureawait-faq/#what-does-configureawaitfalse-do). 

## Use or not?

Generally speaking, ConfigureAwait(false) should be used every time the application logic doesn't depend on SynchronizationContext and TasksScheduler. The advantages of this are better performance and avoiding deadlocks. 

The simpliest and most common recommendation is:

- DO NOT USE ConfigureAwait(false) if developing app-level code
- DO USE it if working on general-purpose code such as portable libraries

## How I almost missed a bug?

Long story short: I used ConfigureAwait(false) in app-level code, particularly, a Web API controller method, which could've caused a bug in localization. 

First, let me describe the approach to localization used in that Web API. The language code came from the Accept-Language request HTTP header, and the CultureInfo.CurrentCulture object was based on it. Then, the localized strings are retrieved from a proper resource file responsible for the language provided by the culture info.

![Localization mechanism](/img/Localization.jpg)

Now, let's consider the following Web API method:

    public async Task<IActionResult> Method()
    {
        await _service.DoSomething().ConfigureAwait(false);

        await _emailSender.SendEmail().ConfigureAwait(false);

        return OK();
    }

After this code was deployed on a test environmeent, a bug was reported - email localization was broken. 

Here is why: according to the [official documentation](https://learn.microsoft.com/en-us/dotnet/api/system.globalization.cultureinfo?view=netframework-4.7#Async), starting from .NET Framework 4.6, culture is embedded into the operation's synchronization context. By adding ConfigureAwait(false), we allowed switching it. After _service.DoSomething() returned, the Web API method execution continued in a different synchronization context that had a default culture info.

## Summary

It isn't recommended to use ConfigureAwait(false) in app-level code. If you opt for using it though, remember bugs are likely to occur. In this case, it might be useful to pay additional attention to sychronization context dependent functionality, such as localization.