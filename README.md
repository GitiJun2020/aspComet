# Background

The aim of this project is to provide a lightweight and extensible COMET implementation which does not require a custom server.
You can pick between 2 implementations that are demonstrated in their own sample web projects:

  1. **Samples/Chat** \
  Traditional ASP.NET Framework 4.7.1 app\
  (implements comet with a custom HttpHandler in the AspComet library

  2. **Samples/AspCometCoreApp**\
   ASP.NET Core 2 app\
   (implements comet with middleware from the AspCoreCometware library

Most COMET implementations require a custom server, due to the fact that ASP.NET's threading model (pooled threads) does not promote scalability for COMET applications. 

A strong motivation for this project is therefore being able to remove this requirement, and to be able to deploy COMET applications to any shared infrastructure or cloud based hosting. Either implementation can be packaged as a single .NET DLL coming in at under 40KB in size.

To find out more about COMET, it's worth reading [Neil Mosafi's blog post](http://neilmosafi.blogspot.com/2009/03/comet-pushing-to-web-browser.html) which describes some of the motivations for the original AspComet-vs2010 library. [bkwdesign]() 

# The Bayeux Protocol

[Bayeux protocol](http://svn.cometd.org/trunk/bayeux/bayeux.html) is a well established publish / subscribe protocol for building COMET servers.  By leveraging this protocol, applications which use AspComet will be compatible with a number of JavaScript client libraries, including JQuery.Comet and Dojo.Comet.

AspComet supports "long polling" under the [Bayeux protocol](http://svn.cometd.org/trunk/bayeux/bayeux.html), using Asynchronous Http Handlers in ASP.NET, which allows applications to scale to be able to service large number of client sessions.

Find out more background information at Neil Mosafi's blog post here: [http://neilmosafi.blogspot.com/2009/03/comet-bayeux-protocol-and-aspnet.html](http://neilmosafi.blogspot.com/2009/03/comet-bayeux-protocol-and-aspnet.html)

# Getting started

   1.  Build or reference the AspCoreCometware library in your new netcoreapp2.1
   2.  Configure your web app just like the sample that's provided:
```C#
    public void ConfigureServices(IServiceCollection services)
    {
        //Minimum req'd sevices
        services.ConfigureBasicCometServices();
        //your custom comet services
        services.AddSingleton<IClientFactory, AuthenticatedClientFactory>();
        services.AddSingleton<HandshakeAuthenticator>();
        services.AddSingleton<BadLanguageBlocker>();
        services.AddSingleton<SubscriptionChecker>();
        services.AddSingleton<Whisperer>();
    }
```
Then, configure the middleware to branch on your desired comet URL:
```C#
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        // Create branch to the CometMiddleware. 
        // All requests ending in comet/45.0 will follow this branch.
        app.MapWhen(
            context => context.Request.Path.ToString().Contains("/comet/45.0"),
            appBranch => {
                // ... optionally add more middleware to this branch
                appBranch.UseCometMiddleware();//formerly an 'HttpHandler'
            });

        EventHub.Subscribe<HandshakingEvent>(app.ApplicationServices.GetService<HandshakeAuthenticator>().CheckHandshake);
        EventHub.Subscribe<PublishingEvent>(app.ApplicationServices.GetService<BadLanguageBlocker>().CheckMessage);
        EventHub.Subscribe<SubscribingEvent>(app.ApplicationServices.GetService<SubscriptionChecker>().CheckSubscription);
        EventHub.Subscribe<PublishingEvent>("/service/whisper", app.ApplicationServices.GetService<Whisperer>().SendWhisper);

        app.UseFileServer();
    }
```
The project uses [Jetbrains TeamCity](http://jetbrains.com/teamcity) for continuous integration, hosted at [TeamCity.CodeBetter.Com](http://teamcity.codebetter.com/project.html?projectId=project59).  The easiest way to get up and running is to download the most recent artifact from there.  Otherwise you can build from source.

Once you have AspComet.dll, you'll need to set up the Http Handler for handling COMET requests.  Add a reference to the dll and add the following line to your Web.Config file:

	<httpHandlers>
	  <add verb="POST" path="comet.axd" validate="false" type="AspComet.CometHttpHandler, AspComet"/>
	</httpHandlers>

Then you need to add one line to your Global.asax.cs file:

	Setup.AspComet.WithTheDefaultServices();

And that's it!  You can now build client applications using a supported Bayeux Javascript client, subscribe to channels and publish messages to them.

# Some more advanced scenarios

### Responding to events

AspComet provides an event hub which allows the COMET Endpoint hosting application to be notified of certain events (such as new clients connecting).  By handling events you can trigger business logic, or send messages back to specific clients or channels.  Some events are cancellable meaning the application developer can add logic to their application to stop things from happening, such as a client subscribing to a channel.  See the chat sample for examples of this.

### Taking complete control

The configuration step described in the Getting Started section - Setup.AspComet.WithTheDefaultServices() - is just provided for the most basic scenarios and doesn't allow you to replace any components of the system.

However, AspComet is built with IoC in mind and is fully configurable via IoC, which lets applications completely replace any internal component of the framework.  It uses the [Common Service Locator Library](http://commonservicelocator.codeplex.com) to locate the services it needs, and provides metadata for applications to easily configure their container of choice with the default services required by the framework.

The best example if this is to look at the [Global.asax.cs file in the Chat sample](aspComet/blob/master/src/Samples/Chat/Global.asax.cs) which uses the Autofac container and replaces a few of the default objects with custom ones.

# License

AspComet is licensed under the terms of the MIT License, see the [included MIT-LICENSE file](aspComet/blob/master/MIT-LICENSE).