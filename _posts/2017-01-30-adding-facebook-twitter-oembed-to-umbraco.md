---
layout: post
title: "Adding Facebook & Twitter OEmbed to Umbraco"
date: 2017-01-30
tags: "umbraco"
comments: true
---
Today, I'll be covering adding in Facebook & Twitter as OEmbed providers to Umbraco 7. Out of the box [Umbraco](https://umbraco.com/) has support for many different services, video services like YouTube & Vimeo being the most popular. However two popular services which aren't available for embed are Facebook & Twitter.

Fortunately in the true spirit of Umbraco, the core project allows anyone with the moxy to have a go the ability to add in their own providers to suit their client's needs. OEmbed is no different!

## Okay, sounds great but what's OEmbed & where can I use this?

OEmbed is a standard which third parties adhere to, allowing users the ability to provide a URL and know that that service will return some HTML which can, you guested it, be embeded in your web application.

You can read more on the topic at the [OEmbed website](http://oembed.com/).

The OEmbed dialog, is supported by Richtext Editor, the Umbraco Grid and there are also third party packages that use this dialog for making use of OEmbed markup in their own way.

Now we know why we might want to add in our own OEmbed providers, _let's have a go!_

### Twitter
So first we'll cover Twitter, because it's super easy to add with minor config changes and no code changes at all. Plus you'll feel like a rockstar after you've done it and will probably want to tackle the Facebook bit too.

Access _EmbeddedMedia.config_, located at `~/Config/EmbeddedMedia.config` and add the following lines of XML configuration.

``` XML
<!-- Twitter settings -->
<provider name="Twitter" type="Umbraco.Web.Media.EmbedProviders.OEmbedJson, umbraco">
    <urlShemeRegex><![CDATA[twitter\.com/]]></urlShemeRegex>
    <apiEndpoint><![CDATA[https://publish.twitter.com/oembed]]></apiEndpoint>
    <requestParams type="Umbraco.Web.Media.EmbedProviders.Settings.Dictionary, umbraco"></requestParams>
</provider>
```

For those of you unfamilar with the _EmbeddedMedia.config_ setup. We're basically:

* Adding in a provider called Twitter
* Expecting a JSON response hence the call to `Umbraco.Web.Media.EmbedProviders.OEmbedJson`
* We want to match any URL which matches the very loose regex pattern, "`twitter\.com/`".
* Once we find a URL that matches, send the URL to the provided API endpoint, in this case: [https://publish.twitter.com/oembed](https://publish.twitter.com/oembed)
* The data we're going to send is coming from the standard, `Umbraco.Web.Media.EmbedProviders.Settings.Dictionary`

So now that we have a Twitter OEmbed provider let's see it in action!

Using [this Tweet](https://twitter.com/umbracoproject/status/824159831498125314) from the [official Umbraco Project Twitter account](https://twitter.com/umbracoproject/), let's add it in to the Fanoe starter kit.

_**Note:** by default Embed is not a configured option in the Fanoe grid setup I've added it in to the row configuration for the purposes of this demo._

<iframe width="1280" height="760" frameborder="0" scrolling="no" src="//screencast-o-matic.com/embed?sc=cbVvYzQPEf&v=5&ff=1&title=0" allowfullscreen="true"></iframe>

Pretty nifty, eh? More information on Twitter OEmbed can be found on the [Twitter Developer documentation](https://dev.twitter.com/rest/reference/get/statuses/oembed).

### Facebook

Okay, deep breath. Facebook is a bit trickier to handle. Fortunately it's still entirely possible using the Umbraco framework but there are more hoops to jump through. 

According to the [Facebook Developer documentation on OEmbed](https://developers.facebook.com/docs/plugins/oembed-endpoints). Facebook currently has different endpoints for Videos and Posts.

To handle this we're going to need a fancy dancy regular expression and given that video has less potential outcomes we're going to target them with this regex and then leave the posts endpoint as a general catch all.

Also the documentation states it returns JSON so we'll need to use `Umbraco.Web.Media.EmbedProviders.OEmbedJson`. Our _EmbeddedMedia.config_ is going to look like this:

``` XML
<!-- Facebook Video settings -->
<provider name="FacebookVideo" type="Umbraco.Web.Media.EmbedProviders.OEmbedJson, umbraco">
    <urlShemeRegex><![CDATA[facebook\.com/((.*)/videos/|video.php)(\?(v|id)\=)?([0-9]+)/?]]></urlShemeRegex>
    <apiEndpoint><![CDATA[https://www.facebook.com/plugins/video/oembed.json/]]></apiEndpoint>
    <requestParams type="Umbraco.Web.Media.EmbedProviders.Settings.Dictionary, umbraco"></requestParams>
</provider>

<!-- Facebook Post settings -->
<provider name="FacebookPost" type="Umbraco.Web.Media.EmbedProviders.OEmbedJson, umbraco">
    <urlShemeRegex><![CDATA[facebook\.com/]]></urlShemeRegex>
    <apiEndpoint><![CDATA[https://www.facebook.com/plugins/post/oembed.json/]]></apiEndpoint>
    <requestParams type="Umbraco.Web.Media.EmbedProviders.Settings.Dictionary, umbraco"></requestParams>
</provider>
```

So that's that then, eh? We have our configuration everything should just work? Sadly no. 

If we add this in and fire it up, we'll get an error appear in the Umbraco app logs. The crux of it I've found is that the Facebook OEmbed endpoint doesn't play nicely with `System.Net.WebClient` which is used in [OEmbedJson's inherited method](https://github.com/umbraco/Umbraco-CMS/blob/release-7.5.8/src/Umbraco.Web/Media/EmbedProviders/AbstractOEmbedProvider.cs#L52-L58),`DownloadResponse(string url)`. Essentially calling the API Endpoint will return a HTML response saying the browser is not supported by Facebook.

Here's a screencast of the outcome in action, using a custom OEmbed provider to allow for easy debugging:

<iframe width="1280" height="818" frameborder="0" scrolling="no" src="//screencast-o-matic.com/embed?sc=cbVvrrQPHX&v=5&ff=1&title=0" allowfullscreen="true"></iframe>

The best solution I could find is providing a custom `OEmbedJson` provider which overrides the `DownloadResponse(string url)` method. I'd consider this to be an edge case as it's more of problem with integrating with Facebook rather than the core Umbraco.

That code looks like this:

``` csharp
using System.Net;
using Umbraco.Web.Media.EmbedProviders;

namespace YourWebsite.Web.Media
{
    public class MozillaOEmbedJson : OEmbedJson
    {
        public override string DownloadResponse(string url)
        {
            using (var webClient = new WebClient())
            {
                // Apply a user-agent Facebook can interact with.
                webClient.Headers["User-Agent"] = "Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)";
                return webClient.DownloadString(url);
            }
        }
    }
}
```

I'm all ears for a better solution than this as this definitely feels quite hacky.

So now all we need to do is update our _EmbeddedMedia.config_ file with our custom OEmbed provider. Which look like this:

``` XML
  <!-- Facebook Video settings -->
  <provider name="FacebookVideo" type="YourWebsite.Web.Media.MozillaOEmbedJson, YourWebsiteDllName">
    <urlShemeRegex><![CDATA[facebook\.com/((.*)/videos/|video.php)(\?(v|id)\=)?([0-9]+)/?]]></urlShemeRegex>
    <apiEndpoint><![CDATA[https://www.facebook.com/plugins/video/oembed.json/]]></apiEndpoint>
    <requestParams type="Umbraco.Web.Media.EmbedProviders.Settings.Dictionary, umbraco"></requestParams>
  </provider>

  <!-- Facebook Post settings -->
  <provider name="FacebookPost" type="YourWebsite.Web.Media.MozillaOEmbedJson, YourWebsiteDllName">
    <urlShemeRegex><![CDATA[facebook\.com/]]></urlShemeRegex>
    <apiEndpoint><![CDATA[https://www.facebook.com/plugins/post/oembed.json/]]></apiEndpoint>
    <requestParams type="Umbraco.Web.Media.EmbedProviders.Settings.Dictionary, umbraco"></requestParams>
  </provider>
```

_**Note:** the type must be fully qualified including DLL name. App\_Code users should add App\_Code instead of a DLL name if it's created within that folder instead of a separate DLL. Ommitting this will cause your provider not to fire._

Great! We've done the leg work, let's see this finally working!

<iframe width="1280" height="818" frameborder="0" scrolling="no" src="//screencast-o-matic.com/embed?sc=cbVvrvQPHJ&v=5&ff=1&title=0" allowfullscreen="true"></iframe>

_**Note:** If you missed this note from the Twitter section, by default Embed is not a configured option in the Fanoe grid setup I've added it in to the row configuration for the purposes of this demo._

Lovely!

## Summary

This is a great example of just how flexible Umbraco can be. We've covered not only a simple OEmbed configuration which comes with Umbraco but also an advanced case where we need to extend what Umbraco already has to get results.

For bonus points, with some CSS magic we could have responsive embedding. Please consult your local friendly CSS skilled person!