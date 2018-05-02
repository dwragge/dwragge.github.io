---
layout: page
---

# Personal Projects
<br/>

## [Dymajo Transfer](https://dymajo.com/transfer/)
[![Transfer Interface](https://s3-ap-southeast-2.amazonaws.com/dylanwragge-blob/dylanwragge-com/transfer.jpg)](https://dymajo.com/transfer/)
Frustrated with FileZilla and WinSCP, both being very ugly and their performance left a lot to be desired, we set out to create an FTP client that was both beautiful and performant. My friend [Jono](www.jono.nz) created the frontend, and I created the backend. The app is a UWP Windows 10 App, which uses [React](https://reactjs.org/) and [Flux](https://facebook.github.io/flux/) for the frontend, and [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) for the backend. Originally written in standard ASP.NET Web API, I was extremely unhappy with the spaghetti code and decided to rewrite in .NET Core, with the ability to go cross-platform in the future.

Interfacing between the two has proved several challenges, including being able to spawn a Win32 process from a sandboxed UWP app and marshalling data between the two processes. The architecture is relatively simple, using a REST API for commands and Websockets using [SignalR](https://github.com/aspnet/SignalR) for communicating from the server to the app. The project is close approaching an initial release. You can find some blog posts about the project on my [Blog](/blog/).
