---
title: Output caching middleware in ASP.NET Core
author: tdykstra
description: Learn how to configure and use output caching middleware in ASP.NET Core.
monikerRange: '>= aspnetcore-7.0'
ms.author: riande
ms.custom: mvc
ms.date: 10/26/2022
uid: performance/caching/output
---
# Output caching middleware in ASP.NET Core

By [Tom Dykstra](https://github.com/tdykstra)

:::moniker range=">= aspnetcore-7.0"

This article explains how to configure output caching middleware in an ASP.NET Core app. For an introduction to output caching, see [Output caching](xref:performance/caching/overview#output-caching).

The output caching middleware can be used in all types of ASP.NET Core apps: Minimal API, Web API with controllers, MVC, and Razor Pages. The sample app is a Minimal API, but every caching feature it illustrates is also supported in the other app types.

## Add the middleware to the app

Add the output caching middleware to the service collection by calling <xref:Microsoft.Extensions.DependencyInjection.OutputCacheServiceCollectionExtensions.AddOutputCache%2A>.

Add the middleware to the request processing pipeline by calling <xref:Microsoft.AspNetCore.Builder.OutputCacheApplicationBuilderExtensions.UseOutputCache%2A>.

> [!NOTE]
> * In apps that use [CORS middleware](xref:security/cors), `UseOutputCache` must be called after <xref:Microsoft.AspNetCore.Builder.CorsMiddlewareExtensions.UseCors%2A>.
> * In Razor Pages apps and apps with controllers, `UseOutputCache` must be called after `UseRouting`.
> * Calling `AddOutputCache`and `UseOutputCache` doesn't start caching behavior, it makes caching available. Caching response data must be configured as shown in the following sections.

## Configure one endpoint or page

For minimal API apps, configure an endpoint to do caching by calling [`CacheOutput`](xref:Microsoft.Extensions.DependencyInjection.OutputCacheConventionBuilderExtensions.CacheOutput%2A), or by applying the [`[OutputCache]`](xref:Microsoft.AspNetCore.OutputCaching.OutputCacheAttribute) attribute, as shown in the following examples:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="oneendpoint":::

For apps with controllers, apply the `[OutputCache]` attribute to the action method. For Razor Pages apps, apply the attribute to the Razor page class.

## Configure multiple endpoints or pages

Create *policies* when calling `AddOutputCaching` to specify caching configuration that applies to multiple endpoints. A policy can be selected for specific endpoints, while a base policy provides default caching configuration for a collection of endpoints.

The following highlighted code configures caching for all of the app's endpoints, with expiration time of 10 seconds. If an expiration time isn't specified,  it defaults to one minute.

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies1" highlight="3":::

The following highlighted code creates two policies, each specifying a different expiration time. Selected endpoints can use the 20 second expiration, and others can use the 30 second expiration.

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies1" highlight="4-5":::

You can select a policy for an endpoint when calling the `CacheOutput` method or using the `[OutputCache]` attribute:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="selectpolicy":::

For apps with controllers, apply the `[OutputCache]` attribute to the action method. For Razor Pages apps, apply the attribute to the Razor page class.

## Default output caching policy

By default, output caching follows these rules:

* Only HTTP 200 responses are cached.
* Only HTTP GET or HEAD requests are cached.
* Responses that set cookies aren't cached.
* Responses to authenticated requests aren't cached.

The following code applies all of the default caching rules to all of an app's endpoints:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies3":::

The following code removes these defaults while applying caching to all of an app's endpoints:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies4":::


You can override these defaults.

## Specify the cache key

By default, every part of the URL is included as the key to a cache entry, that is, the scheme, host, port, path, and query string. However, you might want to explicitly control the cache key. For example, suppose you have an endpoint that returns a unique response only for each unique value of the `culture` query string. Variation in other parts of the URL, such as other query strings, shouldn't result in different cache entries. You can specify such rules in a policy, as shown in the following highlighted code:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies2" highlight="7":::

You can then select the `VaryByQuery` policy for an endpoint:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="selectquery":::

Here are some of the options for controlling the cache key:

* <xref:Microsoft.AspNetCore.OutputCaching.OutputCachePolicyBuilder.SetVaryByQuery%2A> - Specify one or more query string names to add to the cache key.
* <xref:Microsoft.AspNetCore.OutputCaching.OutputCachePolicyBuilder.SetVaryByHeader%2A> - Specify one or more HTTP headers to add to the cache key.
* <xref:Microsoft.AspNetCore.OutputCaching.OutputCachePolicyBuilder.VaryByValue%2A>- Specify a value to add to the cache key. The following example uses a value that indicates whether the current server time in seconds is odd or even. A new response is generated only when the number of seconds goes from odd to even or even to odd.

  :::code language="csharp" source="output/samples/7.x/Program.cs" id="varybyvalue":::

For more options, see the <xref:Microsoft.AspNetCore.OutputCaching.OutputCachePolicyBuilder> class.

## Cache revalidation

Cache revalidation means the server can return a `304 Not Modified` HTTP status code instead of the full response body. This status code informs the client that the response to the request is unchanged from what the client previously received.

The following code illustrates the use of an [`Etag`](https://developer.mozilla.org/docs/Web/HTTP/Headers/ETag) header to enable cache revalidation. If the client sends an [`If-None-Match`](https://developer.mozilla.org/docs/Web/HTTP/Headers/If-None-Match) header with the etag value of an earlier response, and the cache entry is fresh, the server returns [304 Not Modified](https://developer.mozilla.org/docs/Web/HTTP/Status/304) instead of the full response:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="etag":::

Another way to do cache revalidation is to check the date of the cache entry creation compared to the date requested by the client. When the request header `If-Modified-Since` is provided, output caching returns 304 if the cached entry is older and isn't expired.

Cache revalidation is automatic in response to these headers sent from the client. No special configuration is required on the server to enable this behavior, aside from enabling output caching.

## Use tags to evict cache entries

You can use tags to identify a group of endpoints and evict all cache entries for the group. For example, the following code creates a pair of endpoints whose URLs begin with "blog", and tags them "tag-blog":

:::code language="csharp" source="output/samples/7.x/Program.cs" id="tagendpoint":::

An alternative way to assign tags for the same pair of endpoints is to define a base policy that applies to endpoints that begin with `blog`:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies2" highlight="3-5":::

Another alternative is to call `MapGroup`:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="taggroup":::

In the preceding tag assignment examples, both endpoints are identified by the `tag-blog` tag. You can then evict the cache entries for those endpoints with a single statement that references that tag:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="evictbytag":::

With this code, an HTTP POST request sent to `https://localhost:<port>/purge/tag-blog` will evict cache entries for these endpoints.

You might want a way to evict all cache entries for all endpoints. To do that, create a base policy for all endpoints as the following code does:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies2" highlight="6":::

This base policy enables you to use the "tag-all" tag to evict everything in cache.

## Disable resource locking

By default, resource locking is enabled to mitigate the risk of [cache stampede and thundering herd](https://en.wikipedia.org/wiki/Thundering_herd_problem). For more information, see [Output Caching](xref:performance/caching/overview#output-caching).

To disable resource locking, call [SetLocking(false)](xref:Microsoft.AspNetCore.OutputCaching.OutputCachePolicyBuilder.SetLocking%2A) while creating a policy, as shown in the following example:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="policies2" highlight="9":::

The following example selects the no-locking policy for an endpoint:

:::code language="csharp" source="output/samples/7.x/Program.cs" id="selectnolock":::

## See also

* <xref:performance/caching/overview>
* <xref:fundamentals/startup>
* <xref:fundamentals/middleware/index>
* <xref:fundamentals/change-tokens>

:::moniker-end
