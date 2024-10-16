---
layout: post
title:  "Faster iOS Networking with Shared Dictionary Compression"
date:   2024-10-15 21:45:46 -0400
categories: jekyll update
---

## Introduction

Although iPhones are getting faster and faster with every release, networking delays remain a persistent thorn in the side of the user experience. Information remains limited by the speed of light in getting to its destination, and in many cases, there are additional slowdowns along the way (3G connections, subway tunnels, satellite internet, etc.). Driving down response sizes continues to yield benefits for users, and with that in mind, we're going to look at a relatively new technique for it, "shared dictionary compression". Although the technique has been used at places such as Google and Amazon for years, it has recently gained a lot of momentum in the larger developer community. That momentum is mostly on the browser side, but in this blog post, I'll show how we can easily leverage shared dictionary compression for iOS apps.

## The current state of (non-shared-dictionary) compression

To understand shared dictionary compression, let's first give a brief overview of its alternative. The iOS app (the "client") and server agree on some compression library they both support and the server sends the response back compressed with that library. The two most popular compression libraries for this are probably gzip, the oldest and most widely supported library, and Brotli, a relative newcomer that is generally faster than gzip. For iOS apps, both have been supported by `URLSession` transparently since iOS 11 (and even older in the case of gzip). It's important that developers get as much as they can out of this sort of compression before looking at shared dictionary compression, because it's simpler to support and works for a broader scope of responses. It's also important to understand the practical consequences of compression on responses. For example, using short keys in a JSON dictionary, like `"lc"` for `"like_count"`, won't necessarily reduce the compressed response size by much. The compression library can (ideally) see that `"like_count"` is repeated throughout the JSON payload, only store the actual string once, and just refer to that reference for all of its occurrences. Another thing is that alternative data formats to JSON, such as Protobuf, don't necessarily reduce response sizes very much once compression is factored in (and can even increase them!). This is because lots of wasteful looking stuff in JSON, like optional whitespace and quotes around strings, is repeated many times throughout responses, and repetitive data can be compressed really well. In short, compression allows developers to be more "lazy" and not worry as much about repetitive waste. For more information, and a case study of how Brotli can provide benefits over gzip, see [this article](https://tech.oyorooms.com/how-brotli-compression-gave-us-37-latency-improvement-14d41e50fee4).

And while we're at it, consider [switching to HTTP/3](https://developer.apple.com/videos/play/wwdc2021/10094/) if you haven't, and going through a checklist like [this one](https://www.getyourguide.careers/posts/part-four-improving-user-experience-by-boosting-runtime-performance) to make sure you're already leveraging easier networking speedups before using shared dictionary compression.

## What is shared dictionary compression?

Shared dictionary compression is where an iOS app and the server it's communicating with each have a copy of some piece of data (the "dictionary") that can be used to make a request smaller. For instance, if the client and server have both stored the previous day's response for some public endpoint, and today's response for that endpoint has only changed slightly, then the server could just return a diff between the two responses, and tell the client to use the diff in conjunction with the previous day's response (the dictionary). Let's suppose the endpoint is called `/latest_news`, and it returned the following response yesterday:

```
{
    "articles": [
        ...
        {
            "title": "Something new has happened",
            "description": "..."
        },
    ]
}
```

Now the client is asking for a new copy of `/latest_news` today, and the response is almost identical. Only one new article has been written since yesterday:

```
{
    "articles": [
        ...
        {
            "title": "Something new has happened",
            "description": "..."
        },
        {
            "title": "Something even newer has happened",
            "description": "..."
        }
    ]
}
```

With shared dictionary compression, the client and server could communicate that they each have a copy of yesterday's response, and the server could use that as the dictionary just send a diff like:

```
@@ -200,1 +200,8 @@
-        }
+        },
+        {
+            "title": "Something even newer has happened",
+            "description": "..."
+        }
```

This strategy leads to a bigger and bigger reduction in payload size given a larger and larger number of other articles in the response. Furthermore, in practice, the two responses don't need a simple diff like this for it to work. Modern compression libraries, like Brotli and Zstandard, can cleverly pick out snippets of the dictionary and basically say things like "at this point in the new response, use the characters from 435 to 526 in the previous response. Then add some new string 'foo'. Then use the characters from 826 to 879 in the previous response". The only thing that matters is that there's some amount of commonality between the two responses. In fact, the dictionary doesn't have to be a response at all. It could just be a collection of snippets that tend to show up in the `/latest_news` endpoint. For instance, it could look like `[" has happened", "\"\n        {\n            \"title\": "Something ", ...]` (see the `--train` flag for Zstandard's `zstd` tool for an example of how these can be created).

## How much impact can it have?

### Hint: it's more than you might think

Shared dictionary compression, being the more complicated compression alternative here, needs good results to justify itself. Measuring this with just back-of-the-napkin math is treacherous. It can be easy to look at a response payload that's already 20 kilobytes compressed with Brotli and say, "most people's WiFi and 4G devices have download speeds of at least 625 kilobytes/sec. 20 / 625 = 0.032, or 32ms, meaning shared dictionary compression has a ceiling of 32ms reduced, not much at all!". However, this would be ignoring a few important factors. One is that small delays can have a surprisingly high impact on user behavior (an [Amazon study](https://www.gigaspaces.com/blog/amazon-found-every-100ms-of-latency-cost-them-1-in-sales) found that every additional 100ms of latency cost them 1% in sales). Another is that bandwidth estimates largely concern the ideal case: a long-running download where the client and server have figured out how fast the connection is (and thus how fast to send data to the client), and have finished with initial formalities. A small 20Kb request on the other hand, can be dominated by costs like the server tentatively figuring out how fast to send bytes to the client, so it's really not hitting the theoretical max. Here, the larger issue is often the round-trip time (RTT), the time it takes for data to go from the client to server and back. The round-trip time is limited by the speed of light, and sadly can only be so fast, unlike bandwidth which has increased substantially over time. In general, the details of this involve things like TCP slow start and changes in HTTP2, and can be so complicated that even web experts [don't always agree on it](https://www.tunetheweb.com/blog/critical-resources-and-the-first-14kb/). Lastly, it's easy to overestimate user's speeds and forget how many exceptionally slow cases there are out there. For example: users in countries with slower internet speeds, users in a subway train where the connection is going in and out, and users in isolated areas where the internet is spotty.

I'll end this with a quote from an unnamed Chromium dev (note that "SDCH" here refers to a specific shared dictionary compression algorithm):
> Another change in the world is certainly that bandwidth has continued to rise", even though (as I've stressed with [one internet protocol]), the speed of light has remained constant, and hence RTTs have not fallen.  When we look at geographic areas such as India or Russia, bandwidth might not be growing so fast, RTT times routinely approach 400ms, and worse yet, packet loss can routinely approach 15-20% <gulp!!>.  With such long RTTs, and the significant loss rates, the compression rate has an even larger impact on latency than mere "serialization" costs.  The more packets you send, the more likely one is to get lost, and the more likely an additional RTT (or 2+!!) will be required for payload delivery.  As a result, it is very unsurprising that Amazon sees big savings in the tail of the distribution, when using SDCH in such areas.  I'd expect that most any sites that were willing to make the effort of supporting SDCH (and have repeat visitors!!!) have a fine chance of seeing similar significant latency gains (and sites can estimate the compression rate for a given dictionary before wasting the time to ship the dictionary!).

### Measuring the impact more accurately

Since we've seen how difficult it is to measure it with theoretical analysis, a more promising option is to measure things in production. One could do an A/B test where the experiment group has shared dictionary compression enabled. The developer can measure both the real-world change in response time as well as any changes in user engagement as a result.

### Results from real companies

It has been [shown](https://github.com/WICG/compression-dictionary-transport/blob/main/examples.md#static-resource-flow-results) that various network requests for popular websites would be substantially reduced in size by using shared dictionary compression. And size wins like that translate into real results: Amazon [reported](https://groups.google.com/a/chromium.org/g/blink-dev/c/nQl0ORHy7sw/m/LkXwXcUtAgAJ) a 10% reduction in page load times for browser loads in the US. On the mobile front, there's a top 10 App Store app that uses shared dictionary compression. The gains depend on the app, but there's a lot of potential for many apps.

## A concrete scheme

Shared dictionary compression can be implemented using any number of schemes. The developer can choose how the client tells the server that it has a shared dictionary, how the client tells the server which decompression algorithms it supports, etc. However, there's a [draft spec](https://datatracker.ietf.org/doc/draft-ietf-httpbis-compression-dictionary) for an end-to-end solution with a lot of momentum. Although that spec is largely geared towards browsers, and has more bells and whistles than one may need, it's a good starting point for mobile developers. Here's a summary of the spec as it could be applied to iOS apps (note that "URL match pattern" in this section refers to [this](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API)):
- How the server gets shared dictionaries to the client
  - The server can specify a shared dictionary to the client in one of two ways:
    - By adding a `Use-As-Dictionary: <options>` field in the response header (see chaper 2 of the draft spec). This header specifies that the response given should be used as a dictionary for future requests. It provides a URL match pattern as to which requests it's eligible for (e.g. `match="/product/*"`), and can also optionally allow for an ID (e.g., `match="/product/*", id=foo1234`). Note that these relative URLs are referring to the same base URL as of the request.
    - By specifying a `Link: <url>` header, which provides a URL for a dictionary that can be downloaded at the client's leisure (e.g., in a background URL session task). The response from this URL should also include a `Use-As-Dictionary` header in the same format as above.
  - If the dictionary comes with cache controls, such as `Cache-Control: max-age=3600`, then the client should only try to use this shared dictionary while it would otherwise be eligible to use for caching (e.g., while it's still "fresh")
- How the client sends a new request that works with shared dictionaries
  - For whichever compression schemes the client supports, the client adds those compression schemes to the `Accept-Encoding` header. For example, if a client supported Zstandard (regular non-shared-dictionary mode), Zstandard for shared dictionaries, and Brotli for shared dictionaries, then it would use `Accept-Encoding: zstd, dcz, dcb`. Note `dcz` is for Zstandard shared dictionary compression, and `dcb` is for Brotli shared dictionary compression. Technically, these are the "raw" variants of Zstandard and Brotli, but more info on that is outside the scope of this blog post, and can be found in 2.1.4 of the draft.
  -If the client has one or more matching dictionaries for the request, it can select one of those dictionaries and notify the server that it has it. It should select the dictionary with the longest match or, in the event of a tie, the most recent one. It uses the following header: `Available-Dictionary: <SHA-256 hash of dictionary>`
  - If the `Use-As-Dictionary` options for that dictionary included an ID then it should also be included, with a `Dictionary-ID: <id>` header
- How the server returns a response 
  - If the server has the dictionary with that SHA and (if applicable) with that ID, and it supports one of the shared dictionary encodings that the client does, then it can return the response compressed with that dictionary and that encoding. It should set the `Content-Encoding: ...` header appropriately.
- Security considerations
  - The dictionary must be treated as equally sensitive as the content, because a change to the dictionary can result in a change to the size and content of the decompressed response
  - To prevent third parties from injecting compression dictionaries for sensitive first-party content, dictionaries should only be used for requests to the same domain that the dictionary was downloaded from
  - See the security chapter of the draft for more details
- Other things
  - Any caches between the client and server should support `Vary` for both the `Accept-Encoding` and `Available-Dictionary` headers. Otherwise, it can deliver corrupt data if it returns a cached response that uses a dictionary for a client that doesn't have that dictionary.

Mobile developers aren't constrained to what browsers support, and can roll their own networking conventions in a situation like this. However, it's great to have the leading browser implementation in mind when doing it. If there's interest in it, I can release my own reference Swift implementation, and decompression-only 70kb version of the Zstandard library.

## Specific use cases

Although we've already shown an example for an articles endpoint that only changes slowly over time, here are a couple other examples:
- A calendar app syncs its state with the server by downloading the user's whole calendar whenever they open the app, including all of the events the user has scheduled, past, present, or future. This doesn't scale with a growing number of events, and so things are getting slow for long-time power users. They could use complicated diffing logic to only send a diff of what the user already has, where the server keeps track of exactly what events the user has/hasn't downloaded to each device. But what about using  "dumb" content-agnostic shared dictionary compression to get similar results with less effort and risk of bugs?
- A chat app wants to keep message payloads small for a user's chats. Every so often, it provides an out-of-band (i.e. `Link:` header) compression dictionary for each active 1-on-1 chat, with a bit of history from that chat.

## Conclusion

In summary, shared dictionary compression offers a powerful yet underutilized method for reducing payload sizes and improving network performance in iOS apps. By leveraging similarities between different responses, this technique can lead to significant improvements in load times, especially for users on slower or unreliable networks. While it may require some initial work, including measurement and server-side implementation, the benefits can be substantial. For more info, consider reading Chrome's [blog post](https://developer.chrome.com/blog/shared-dictionary-compression) on the subject.
