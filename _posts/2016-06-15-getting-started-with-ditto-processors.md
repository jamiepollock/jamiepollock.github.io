---
layout: post
title: "Getting Started with Processors in Umbraco Ditto v0.9.0"
date: 2016-06-15
tags: "umbraco, umbraco-ditto"
---
I’m a big fan of [Umbraco Ditto](https://github.com/leekelleher/umbraco-ditto), We’ve used it on all of our Umbraco projects at
[Connect](http://www.connect-group.com/) over the past couple of years, having become a bit of an evangelist, recommending it becomes a part of our Umbraco dev kit. It makes accessing data from the Content Management System (CMS) that much more friendly.

The the only thing that wasn’t always so clear was when to use TypeConvertors or ValueResolvers. Sometimes the lines blurred and it was fairly easy to abuse the best practices set out by the team.

## Enter processors

Fortunately there is now a solution as part of Ditto 0.9.0, the team have added in the rather fantastic Processors.

Processors takes the best of both worlds from TypeConvertors & ValueResolvers and then add a whole lot of awesome. They’re chain-able allowing for some really powerful functionality which the previous workflow couldn’t possibly achieve.

If you have an existing Ditto powered site and are looking to upgrade your TypeConvertors & ValueResolvers look no further than this [handy upgrade guide](http://umbraco-ditto.readthedocs.io/en/latest/upgrade-090/)
over on Ditto’s [readthedocs](http://umbraco-ditto.readthedocs.io/en/latest/).

## So let’s get started

I’m fairly new to tech blogging but I’ve covered each step of this blog entry in a [separate GitHub repository](https://github.com/jamiepollock/blog-demo-ditto-v090). Hopefully each commit will show the progress of what’s changed in Ditto. The quick start repo also comes with a mostly fresh Umbraco v7.4.3 instance and a local .sdf file so no document types are required to be setup. It’s all ready to go!

### Step 1: Adding our Controller, View & ViewModel

This step wires in the the standard workflow when using Ditto. The Controller binds the model to a Ditto populated Plain Old C# Object (POCO) and then passes that to the view.

If you’re familiar with Ditto then you’ll recognise the `UmbracoProperty` attribute that does the work typically of binding an Umbraco `IPublishedContent` property to the POCO property. The only difference in v0.9.0 is this is now a DittoProcessor rather than DittoValueResolver.

If you’re new to Ditto then a big change is the use of `@model DemoApp.ViewModels.HomePageViewModel` rather than `@inherits UmbracoTemplatePage`. This is so we can use our strongly typed POCO and nothing more. This means our page lacks the standard Umbraco helpers and `IPublishedContent` but I’m a firm believer in keeping this stuff outside of my views. :)

### Step 2: Let’s make it cleverererer

Let’s say we wanted to provide a fallback for our Title property. While not necessarily the first choice for a real world application I’ve chosen the SEO Meta Title (seo\_metaTitle). And then as a fallback for *that* fallback is the nodeName (Name). `AltUmbracoProperty` processor to the rescue! This is a new attribute to v0.9.0 which I feel makes alt property assignment far more readable than an optional overload of `UmbracoProperty` processor.

```csharp
...
[UmbracoProperty("content_title", Order = 0)] 
[AltUmbracoProperty("seo_metaTitle", Order = 1)]
[AltUmbracoProperty("Name", Order = 2)]
...
```

.So with these two additional lines we have a pretty bulletproof property. All `IPublishedContent` objects have a Name so we can ensure no matter what that our property will be populated with CMS goodness. Have a go at adding in content in the sample CMS, for each property to see the expected results. As you’ve probably already guessed the `Order` parameter is there to ensure the running order of processors.

### Step 3: Process More!

So now we have an example of the ordering and fallbacks of Processors, which is essentially an example of how they chain together. Let’s have a go at mutating the data we have into another type.

I’m going to add a custom `DittoProcessorAttribute`, this will take the current input value from our previous `UmbracoProperty` & `AltUmbracoProperty` processors and convert it into an integer.

*Note: While the return type is an integer and the property of Title is a string, Ditto manages the conversion from other types to a string automatically.*

So we now are going to add in a `StringLengthProcessor`.

```csharp
using Our.Umbraco.Ditto;

namespace DemoApp.ComponentModel.Processors 
{ 
    public class StringLengthProcessor : DittoProcessorAttribute 
    { 
        public override object ProcessValue() { 
            return Value == null ? 0 : Value.ToString().Length; 
        } 
    } 
}
```

This simply returns the length of the Value as a string or 0 if it’s null. Which then will be added to our `HomePageViewModel` by adding a new additional attribute.

```csharp
[UmbracoProperty("content_title", Order = 0)]
[AltUmbracoProperty("seo_metaTitle", Order = 1)] 
[AltUmbracoProperty("Name", Order = 2)]
[StringLengthProcessor(Order = 3)] 
public string Title { get; set; }
```
Because our new custom attribute extends `DittoProcessorAttribute`, it already knows all about the `Order` parameter.

Build & check your view now and see the text has been replaced with the length of the text entered in the CMS.

### Step 4: Bringing some Context

The final step is bringing in a custom `DittoProcessorContext`. This is what Ditto uses when it wants to bring in external data, perhaps from a the controller’s route data or a database. This is stuff that a Ditto processor really doesn’t need to have access to or understand to ensure our application has a separation of concerns. It’s a best practice which mean we don’t put `HttpContext.Current` or a PetaPoco database call in our `ProcessValue()` call.

In our case, we’re going to take a querystring parameter, offset, and add it to the string length generated by our previously chained custom Ditto processor.

So to do this, our Context looks like this:

```csharp
using Our.Umbraco.Ditto; 

namespace DemoApp.ComponentModel.Processors 
{ 
    public class AddNumberProcessorContext : DittoProcessorContext 
    {
        public int Number { get; set; }
    }
}
```

Simple enough, it has a single int property, Number, which we can bind
from our controller like so:

```csharp
using System.Web.Http;
using System.Web.Mvc;
using DemoApp.ComponentModel.Processors;
using DemoApp.ViewModels;
using Our.Umbraco.Ditto;
using Umbraco.Web.Models;
using Umbraco.Web.Mvc; 

namespace DemoApp.Controllers 
{
    public class HomePageController : RenderMvcController 
    {
        public ActionResult HomePage(RenderModel model, [FromUri] int offset = 0)
        {
            var contexts = new[]{ new AddNumberProcessorContext() { Number = offset } };

            return View(model.As<homepageviewmodel>(processorContexts: contexts));
        }
    } 
}
```

Note the parameter is decorated `FromUri` to ensure it only comes in from the Uri, in our case the QueryString. This is then added in as an array to the `processorContexts` parameter of `.As<HomePageViewModel>(…)`. I believe Ditto supports globally declared `DittoProcessorContexts` but that’s not something we’ll be covering in this blog article. For now we’re covering this logic as exclusively tied to the action route data for the HomePage.

Once Ditto has a bunch of Processor Contexts assigned to the Ditto `.As<HomePageViewModel>()` call we need our new Processor to recognise this.

```csharp
using Our.Umbraco.Ditto;

namespace DemoApp.ComponentModel.Processors 
{
    [DittoProcessorMetaData(ContextType = typeof(AddNumberProcessorContext))]
    public class AddNumberProcessor : DittoProcessorAttribute 
    {
        public override object ProcessValue() 
        {
            if (Value is int && Context is AddNumberProcessorContext) 
            { 
                var castContext = (AddNumberProcessorContext)Context; 
                
                if (castContext != null) { 
                    return ((int) Value) + castContext.Number; 
                } 
            }

            return Value;
        }
    }
}
```

It’s as simple as adding `[DittoProcessorMetaData(ContextType = typeof(AddNumberProcessorContext))]`, which sets the Processor to expect a Context of our new processor context. If a context is available we then use it in our code. I’ve added a check to ensure it is the right type and after that it’s a simple case of added the two numbers together.

### In conclusion (alternate title: A lot to Process\[or\]…)

We’ve covered:

1.  Using standard processors
2.  Chaining standard processors
3.  Creating a custom processor
4.  Creating a custom processor context for use with a custom processor

This is just the tip of the iceberg when it comes to the new additions added in Ditto v0.9.0.

In a separate blog I might look in to caching and perhaps some additional advanced uses.

Happy Dittoing!