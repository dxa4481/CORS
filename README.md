JSON API's Are Automatically Protected Against CSRF, And Google Almost Took It Away.
===================

_Disclaimer: Let me start by saying I am not advocating the built in protections of JSON API's against CSRF as a sole line of defense. As I will outline here, it's not a reliable protection_


----------


## Background
-------------

To understand how JSON API's are protected against CSRF we need to first understand a few things

+ How Cross-Origin Resource Sharing (CORS) works
+ What is CSRF

If you feel like you already have a solid grasp on these topics and the nuances between `simple` and `non-simple` HTTP requests, skip to the `So Why Are JSON API's Protected?` Section

#### Cross-Origin Resource Sharing (CORS)

CORS outlines the policy which defines how a user visiting one origin can make requests to other origins and view the response to those requests. This policy is what defends against one origin from stealing another origin's data. It prevents Facebook from being able to steal your Gmail contents.

Simply put, while visiting one origin, you cannot make a request to a second origin and view the response without the _consent_ of the second origin. This consent takes the form of the `Access-Control-Allow-Origin` header in the response.

That said, there are cases where one origin can make requests to a second origin without consent, the first origin just won't be able to view the response. This nuance depends on whether the request is `simple` or `non-simple`.

When one origin attempts to make a `non-simple` HTTP request to a second origin, the browser first makes what is called a `pre-flighted` requests to check to see if the server gives consent to making the main request. See the image below to see in more detail how this works:

![words](https://mdn.mozillademos.org/files/14289/prelight.png)

`Simple requests` skip this preflight entirely. `Simple requests` can just be made directly to the server, and the only line of defense is once the response comes back to the user's browser, the browser checks the response headers to determine if the javascript on the page has consent to view the results. This becomes a problem for state changing requests, as I'll outline below in the CSRF section.

So what defines `simple` and `non-simple` requests? Well these rules can seem a little arbitrary, but I'll outline them below.

>What is a Simple HTTP Requests?
>__________________________________
>`Simple` HTTP requests are only the following methods:
>
>+ GET
>+ HEAD
>+ POST
>
>`Simple` HTTP requests can only have the following content-types:
>
>+ application/x-www-form-urlencoded
>+ multipart/form-data
>+ text/plain
>
>`Simple` HTTP requests cannot contain custom headers, and may only set the following headers
>
>+ Accept
>+ Accept-Language
>+ Content-Language
>+ Content-Type (but note the additional requirements above)
>+ DPR
>+ Downlink
>+ Save-Data
>+ Viewport-Width
>+ Width
>
>Anything else is a `non-simple` HTTP request.


So when do these rules become a problem?

#### Cross-Site Request Forgery (CSRF)

CSRF leverages the fact one origin can make credentialed `Simple` HTTP requests to a second origin without consent. The first origin cannot view the response, but they can make the request. This becomes a problem is the server is configured to perform a state changing actions for authenticated users for `Simple` HTTP Requests.

Credentials can take the form of any of the following

+ Cookies
+ Basic Authentication
+ Integrated Windows Authentication

If a user has these credentials saved, one origin can force said user to make `simple` requests with these credentials.

To solve this problem, most servers leverage the fact origin's can't view the response to requests, and they in affect, create their own preflight procedure to protect against `simple` http requests. This usually takes the form of one request being made which fetches a token that's required for subsequent requests. [More details on this can be seen here](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet)

## So Why Are JSON API's Protected?
_____________________________

If you haven't figured it out already, JSON API's should strictly enforce the `application/json` content-type. This makes the request `non-simple`, and forces the browser to pre-flight all cross-origin requests.

Countless web applications take no steps to protect against CSRF, so this added safety net for JSON API's has been comforting to me. There have been many cases when doing bug bounties or pen-tests where it was clear no CSRF protection was added by the developer. For modern RESTFUL applications that only communicate over JSON these developers have a nice safety net they don't even know about.

## Why Did Google Almost Take It Away?

What I'm referring to is [this thread](https://bugs.chromium.org/p/chromium/issues/detail?id=490015) 

Simply put, Google messed up when they implemented `navigator.sendBeacon` and accidentally allowed `non-simple` HTTP requests to be made cross-origin without preflight via the sendBeacon API.

This alone isn't a big deal, browsers have security problems all the time; they're usually fixed quickly.

What was concerning was this issue was open for 2 years, in part because of people in the thread who didn't understand the issue, and wanted to change the CORS rules instead of fix the issue. Here are a few comments from the thread:

> For what it's worth, the requirement that we do a preflight for certain Content-Type values is a very silly one.

and

> Since it's hard~impossible to predict the fallout from changing this behavior now, we're probably stuck with it. Well, at least for the features that have already shipped. Perhaps we could remove this requirement for new features moving forward?


To be clear, I'm not writing this to shame anyone, but simply to shed light on the issue, and illustrate how we almost lost what is in my view, an important built in browser protection against CSRF.
