---
title: "State of SameSite cookies in Firefox and Chromium"
date: 2023-03-12T20:29:02+01:00
draft: false
tags: ["SameSite", "Firefox", "Chromium", "CSRF"]
---

SameSite cookies are commonly used to harden websites against CSRF attacks.
These attacks can be mitigated in certain scenarios with SameSite cookies, since a cookie with 
the SameSite attribute set to *strict* should not be send to the destination 
site, if the request passed a foreign site. However, handling of the SameSite 
attribute differs between Firefox and Chromium.


### What is the SameSite attribute in theory?

The SameSite attribute was specified in the RFC draft [rfc6265bis](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-11).
It can be used with three different enforcement modes [RFC6265bis#section-5.5.7.1](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis-11#name-strict-and-lax-enforcement): *none*, *strict* and *lax*.
- *none* disables the SameSite enforcement such that cookies are also sent when the request is invoked by a cross site.
- *strict* enforces the cookie only to be sent when it is a same site request. Therefore the cookie is only sent, when the site was requested by entering the url manually in the url bar of the browser or navigating on the same domain.
- *lax* is not as strict as the *strict* mode since it allows to sent cookies in a request that is invoked by a cross site when it uses a safe http method. Http methods are considered safe if their semantic is defined as read-only ([RFC7231#section-4.2.1](https://tools.ietf.org/html/rfc7231#section-4.2.1)).

In order to separate a same site request from a cross site request, the RFC introduces an algorithm in [section 5.2](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis-11#name-same-site-and-cross-site-re).
To conclude the algorithm, one can say, that every request, that is not initiated by hand(by entering the url manually) or by the exact match of the referring domain, is cross site.
Every other request is a same site request.


### Practical behavior in Firefox and Chromium

For testing the behavior of Firefox and Chromium I've implemented a website, that
sets three cookies, is able to display them and is able to redirect manually, as
well as automatically. Each of three cookies have the SameSite attribute set to
*none*, *lax* and *strict*. The website is able to perform the role of both sites,
since it is able to perform the redirect on both ways. Therefore the application
will be started twice in a docker container with a static ip address: 10.5.0.2 and
10.5.0.3. This is important since it enforces another target address for the
redirect.

After starting the docker container, we are able to open the first site in our
first browser we are testing - Firefox 110.0.1 on linux. Then we click on
"Set the cookies" in order to set all three cookies. As one can see in the
screenshot, they are set correctly. Next to Firefox you are also able to see,
that they are set in Chromium 111.0.5563.64 on linux.

![inital setup](/img/state-of-same-site-cookies-in-firefox-and-chromium/initial.png)

Next we click on "Invoke manual redirect for testing" in both browsers. This will
redirect the browser to 10.5.0.3. There we do not see any cookies, as we did not
set them. If we then click on "Invoke manual redirect for testing" again, we are
redirected back to 10.5.0.2 and are able to see only the cookies without the
SameSite-attribute set to *strict*, which behaves exactly like the RFC described
it before.

![manual redirect](/img/state-of-same-site-cookies-in-firefox-and-chromium/manual.png)

In our next test we will perform an automatic redirect. To reset the scenario,
we click on "Delete cookies" and "Set the cookies", which leads to the same page
as in the first screenshot. After resetting the scenario, we click on 
"Invoke the redirect for testing". This will automatically redirect to 10.5.0.3
and the back to 10.5.0.2. Hence the algorithm for determining a same site request
states out that every URL in the redirect must be the same as the origin for a
same site request, we expect to not see the cookie with the SameSite-attribute set 
to *strict* as well. However, Chromium is sending the cookie along with the request.
Firefox, in contrast, is behaving correctly.

![automatic redirect](/img/state-of-same-site-cookies-in-firefox-and-chromium/automatic.png)


### Potential attack surface

Some sites allow users to be redirected by a given URL as a query parameter, e.g.
when you enter the URL to a restricted area without authentication and the destination
URL is stored as a query parameter on the login page. If these redirects are used, even
if they are bad practice, and allow redirects to external sites, cookies with SameSite
set to *strict* will not mitigate CSRF attacks in Chromium.

In detail, an attacker would the an example URL to the victim like https://vuln.erable/login?to=https://att.acker. This would show the login site of
the vulnerable site to the victim, who then enters the credentials. The vulnerable
site then sends the cookie containing the session with SameSite set to *strict* back
to the browser and invokes then the redirect to the attacking site via HTTP- or JS-
redirect. The attacking site will then create some malicous redirect, which might
extract some information or controls some aspects in the vulnerable site. This redirect
is the performed by the browser. Firefox will drop the session cookie due to the
SameSite attribute and will therefore break the session, such that the exploit
would be mitigated in the first place. Chromium however will not drop the session
cookie as shown in the lab setup before. Therefore the exploit would be successful.


### Conclusion

Firefox and Chromium handle the SameSite attribute with enforcment mode *strict*
quite different in terms of redirects. On automatic redirects, Chromium will not
drop the cookie, in contrast to the specification. However, the potential attack surface
is quite limited and requires bad practices and other flaws in the web application.
Therefore one should be aware of the different behavior and not only rely on a single
countermeasure against certain attacks. The mitigation should always be validated
in certain scenarios.
