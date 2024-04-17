---
layout: page
title: PoC Release + Some Okta Research
date: 2024-01-27
tags: security
---

I've finally gotten a chance to catch up on my steadily increasing backlog of RSS articles, and one in particular from Luke Jennings at Push Security [caught my eye](https://pushsecurity.com/blog/okta-swa/). Luke took a look at the Okta functionality around SWA's and what kinds of security guarantees they provide. I strongly recommend you give the article a read through before continuing, as this post will be based on the information included there.

Luke did not include a PoC with their blog post, but I wanted to have this for future testing, so I set out to build one myself. The final product of this can be found [on my github](https://github.com/sig1nt/SWArevealer), which will be live by the time this blog post is published. This tool allows one to specify an Okta account and it's login credentials, and will enumerate and extract all the SWA passwords assigned to that account, even those hidden from the user normally. This can be very useful if you've already compromised an Okta account and now want to escalate privilege or expand your password collection.

About halfway into building this tool, I realized that Luke was researching the new Okta Identity Engine accounts, while my PoC was being built against Okta Classic. While OIE is the future of Okta, an informal survey of whoever would respond to my Discord questions determined that enough Okta Classic still exists that a documenting of my research on how to implement this tool would be helpful. **None of the research presented here exposes any previously undisclosed security issues in Okta.**

## Okta SWA Plugin

As per Luke's article, in order for Okta to perform SWA functionality, the user must install their custom browser extension, which effectively works the same as a password manager with autofill. Further research into how Okta has mitigated against the various footguns of password autofill should at some point be done, but will not be the focus of this post. Since this is a browser extension, we can proxy this through Burp and we find that it uses a set of bespoke endpoints for most of the work that it does:

```
/api/plugin/2/sites
```

This endpoint gives a list of all the different sites that will need SWA plugin help, as well as what endpoints need to be called in order to get the various passwords. A GET to this endpoint responds with the following snippet:

```json
{
  "site": [
    {
      "siteURL": "https://<my-org>.okta.com/app/<my app name>/<my app id>/",
      "scriptURI": "/api/plugin/2/app/<my app name>/<my app id>/flow",
      "callbackURI": "/api/plugin/2/app/<my app name>/<my app id>/script/flow/status",
      "progressMessage": "Signing in to Example..."
    },

...
```

Each SWA site will have an entry in the `site` array, which one can in turn use the `scriptURI` to make a request to get the username and password to be autofilled. Since the plugin needs to fill in the password regardless of secrecy, this attack bypasses the password transparency controls, just like Lukes. However, to call these API's, we need to be able to get both a session id and a bearer token from Okta, so we must now contend with Okta's login stack.

## Login to Okta

### Session Login

Logging into Okta starts with the process to get a session id. The easiest way to start this process, especially if you're automating it, is with a plain `GET` call to you Okta domain. This will accomplish two things:

1. Okta will send you all the cookies you will need in order to make the session id auth call later on
2. Okta will send you the "state token", which you must use for all of your login calls, in a massive blob of JSON

Once you've accomplished this, you can follow the official [Authentication API](https://developer.okta.com/docs/reference/api/authn). A call to `/api/v1/authn` with the appropriate parameters will start the authn flow. However, it's important to note that you can NOT use the `password` field for you password in this API call. Instead, you must send the call with only the username, and the API will send you the proper link to use to submit your password. _This is true even if you get a SUCCESS message from the API_.

The state token you get from that response will be used in the undocumented/very-poorly-documented `/login/token/redirect` API route. This was the most finicky part of this login process, as the route will fail with a generic 403 if the request is not crafted exactly correctly. Some of the things that caused this call to fail:

- Using a state token that was generated through the one-shot call to `/api/v1/authn` rather than submitting the password separately
- Not including all the cookies, even the seemingly meaningless ones, that you would have gotten from a `GET` request to your Okta domain (https://<something>.okta.com)
- Not including the state token in all steps of the login process
- Using HTTP 1.1 instead of HTTP 2 (this one got me for a while)

Assuming you've done all this correctly you should now see a 302, and more importantly you should receive a `sid` cookie. You in theory could be done at this point, because the actual password getting call is only authenticated with the `sid`. However, if you want that sweet `/api/plugin/2/sites` call which enumerates all the SWAs in an account for you, you'll need a bearer token, so that means...

### Time For Some OAuth

Fortunately, the folks over at Okta know a thing or two about OAuth (I say staring at [@aaronpk](https://twitter.com/aaronpk)'s book on my desk), so the Oauth flow used here is pretty bog standard; to the point that I was able to use an oauth library for most of this part. Since this is technically a front-end flow, Okta has opted to use [Authorization Code with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce).

The only part that's slightly odd is the consent screen, which is to say the lack of one. In a normal Oauth flow, we would be given a URL where the user would need to present credentials and consent to the specific scopes requested. However, instead, the consent URL Okta provides simply checks that the `sid` from the prior section is present, and goes forward with issuing the token. I actually like this as a way to specifically login in your first party clients that have already established a web session with you, without having confusing dialogue about why Okta is authenticating with itself.

After this, it goes back to being standard Oauth, and in the end you are presented with your bearer token and can call any API you so please with in Okta or the plugin.

## Conclusion

The Okta SWA feature provides a clear value add to the product, but it cannot expect to be able to obscure passwords from the people using it. If Okta wishes to continue this feature, adding proper warnings about the accessibility of these passwords would go a long way to making sure users are properly informed.

If you want to dig deeper, feel free to inspect [my PoC](https://github.com/sig1nt/SWArevealer) and feel free to reach out with any questions you might have (I'll do my best to respond).
