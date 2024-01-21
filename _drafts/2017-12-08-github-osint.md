---
layout: post
title:  "Finding Vulns with GitHub OSINT"
categories: security
---

One of my favorite things to see during a pentest is a version number on an open
source piece of software. When I was less experienced, I used to pass these by
as "best practices" vulnerabilities, something to be logged as informational.
However, I've learned that these best practices tend to be established for a
reason. Today, we'll walk through an example of using these version numbers to
translate into real, reportable vulnerabilities. More specifically, we'll look
into a very common tool in many corporate environments, Kibana.

## Aggregating Data on Data Aggregators

Kibana allows one to visualize data that's stored in Elastic Search, and
accordingly if one wishes to use certain versions of Elastic Search, they must
also use certain versions of Kibana. At the bottom of the Kibana pane is that
magic version number, and in this case, it was version 4.1.1, which was a few
years old at the time. I logged this information in the back of my mind, and
came back to it later after checking other parts of the system. I plugged the
version number into Kibana and got a hit from an Elastic (the group that makes
Kibana) [site][Elastic-CVEs]. 

Looks like we have hits for a quite a few CVE's on this version. CVE's can be a
very hit-or-miss discovery, depending a lot on what kind of coverage they've
gotten in the security community. Certainly, some CVE's, especially named ones,
have lots of literature about them, as well as reproduction steps. But a lot of
the time, you get ones like [CVE-2015-9056][XSS-CVE]. Absolutely no information
in the description, and further searching for that number leads to pretty much
no more details, let alone a portion of the code to check or reproduction steps.
So on first glance, this looks incredibly unhelpful, Kibana is a huge project
and trying to search the whole code base for this bug would be intractable.

## I Guess We Should Put This Code Somewhere

Fast forward another couple of days, and we're wrapping up this pentest, just
looking for loose ends to tie up. I still have the note of this CVE in my list,
but I really wanted to get some kind of PoC for this. I also had been having a
discussion earlier today about [Kerckhoff's Principle][Kerckhoff], specifically
regarding why it makes Open Source perfectly safe to use. But wait, if we have
everything about open source software being, well open, then we have access to
it's version control history. Thus, instead of trying to reduce the search space
for the CVE by location in the code, we can reduce it by time.

So here is the play. The CVE was fixed in version 4.1.3, which was tagged on
commit [f895f22][Kibana 4.1.3], so all we need to do is look for XSS
vulnerabilities that were patched around the time of that tag being released.
So we'll search GitHub issues for `is:pr is:closed xss` gives us only 12
results, one of which looks very interesting, [PR 5789][XSS PR]. The pull
requests looks like it lines up with when the CVE was reported, and the diff is
small and absolutely beautiful for this kind of recon:

```
diff --git a/src/kibana/plugins/settings/sections/indices/_scripted_fields.js
b/src/kibana/plugins/settings/sections/indices/_scripted_fields.js
index 2e2a0e5ab72..8cd4b1b1026 100644
--- a/src/kibana/plugins/settings/sections/indices/_scripted_fields.js
+++ b/src/kibana/plugins/settings/sections/indices/_scripted_fields.js
@@ -37,8 +37,8 @@ define(function (require) {
             rowScopes.push(rowScope);
 
             return [
-              field.name,
-              field.script,
+              _.escape(field.name),
+              _.escape(field.script),
               _.get($scope.indexPattern, ['fieldFormatMap', field.name, 'type',
'title']),
               {
                 markup: controlsHtml,
```

## POC || GTFO
Alright, confession time, I have no idea how to use Kibana (or at least I didn't
at the time of this engagement). Depending on the time boxed nature of your
pentest, this can be enough to go find the one person on your pentest team who
knows Kibana because they like data science and try to mine info from them.
However, this was the most interesting lead I had for the time being, so I
figured I would follow it and see what I could find.

So, we know that the bug exists in the `_scripted_fields.js` file, buried deep
in a path of source folders. The bug seems to be that the name of the scripted
field as well as the script itself were not properly JavaScript escaped. So, we
need to figure out where one goes to create a scripted field. Plugging "scripted
fields Kibana" leads to [a guide to Kibana scripted fields][Scripted Fields]. It
seems to be for a more recent version, but maybe the process is the same in the
older version. Also, the guide is riddled with security warnings, which is
again, music to my ears for this kind of recon.

Browsing the guide, it says we create Scripted Fields in the "Settings >
Indices" menu, which matches up with the path we saw in the diff! Sure enough,
we can go to that page, and a wonderful page pops up asking for a name and
script. A quick `<script>alert(1)</script>` later, and we have two new vectors
for XSS.

## So What?

When on an engagement, it can sometimes be easy to discredit findings like "an
old version" as just best practices, but one must stop to consider why these
best practices are in place. If a system is using unpatched software, especially
if it has open CVE's, it is a danger. Additionally, don't let a lack of
information about a CVE be discouraging. Having the source code and version
control history of a piece of code is a massive wealth of information, and while
difficult to parse is worth exploring. So on your next engagement, double check
those version numbers, and maybe dig around in GitHub for interesting patches.
Happy Hacking :)

[Elastic-CVEs]:     https://www.elastic.co/community/security
[XSS-CVE]:          http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9056
[Kerckhoff]:        https://en.wikipedia.org/wiki/Kerckhoffs%27s_principle
[Kibana 4.1.3]:     https://github.com/elastic/kibana/commit/f895f228de1c8c34b1a5e84c3336c19abc35bce3
[XSS PR]:           https://github.com/elastic/kibana/pull/5789
[Scripted Fields]:  https://www.elastic.co/guide/en/kibana/current/scripted-fields.html
