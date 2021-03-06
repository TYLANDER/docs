---
navhome: '/docs/'
next: False
sort: 20
title: Security drivers
---

# Security drivers

A security driver is a file in `/=home=/sec/<tld>/<domain>/hoon` that handles
the authentication for all HTTP requests to `https://<domain>.<tld>`. When
anything in Urbit makes an HTTP request through `%eyre` (our web server), it
checks to see if we have a security driver for the requested domain, and if so
filters the request through the driver. The security driver will usually either
decorate the request with the needed credentials if it has them, or else it will
guide the user through the process of authenticating Urbit with the service.

Each web service needs its own security driver, but most of them are pretty
standard. We recommend starting with an existing security driver based on your
needed authentication method (e.g. basic auth, OAuth1, OAuth2), and changing
whatever is specific to your service.

The best strategy for building security drivers is to copy a similar one and
tweak it until it works for you. The best representatives are Github for Basic
Authentication, Twitter for OAuth1, Slack for OAuth2 when access tokens don't
expire, and Google APIs for OAuth2 when access tokens do expire. In most cases
you'll need to make very few changes to one of these models.

Still, it's worth seeing one or two built from the ground up. Here, we'll build
a connector for the Github API v3. The simplest way to interact with the Github
API is to just fetch https://api.github.com from the dojo:

    ~your-urbit:dojo> +https://api.github.com

Github exposes a few endpoints to the general web, and the root endpoint is one
of them. This gives you a textual representation of a JSON object that contains
a bunch of urls to other parts of Github's API. This is exactly the same
response you would get from just running `curl https://api.github.com` on UNIX.

If we don't have a security driver for Github yet, many of the endpoints won't
be accessible, or they will only have publicly accessible information. Most of
what we care about requires us to be authenticated.

## Basic auth

Here's a simple security driver:

    ::  Test url +https://api.github.com/user
    ::
    ::::  /hoon/github/com/sec
      ::
    /+    basic-auth
    !:
    |_  {bal/(bale keys:basic-auth) $~}
    ++  aut  ~(standard basic-auth bal ~)
    ++  filter-request  out-adding-header:aut
    --

Github supports authentication through either Basic Authentication or OAuth2.
We'll show the basic auth example first, but in general we'd prefer OAuth2.

Since this driver is for Github, put it in `/=home=/sec/com/github/hoon`.

To try this out, we first have to initialize our credentials with
`|init-auth-basic`, which prompts for the url of the service (api.github.com),
username, and password. It stores this information in a manner that `%eyre`
knows how to decode and send put into your `bale`. Note that `|init-auth-basic`
is standard for basic auth, but other auth schemes (like OAuth) are initialized
in other ways.

After you've run `|init-auth-basic`, you should be able to run
`+https://api.github.com/user` and get a response indicating who you're logged
in as.

You can similarly POST:

    ~your-urbit:dojo> +https://api.github.com/gists &json _(cork poja need) '{"files":{"file1.txt":{"content":"can\'t stop the signal"}}}'

This creates a gist, go to `https://gist.github.com/<username>` to see it.

The `+url` syntax used in "source" position makes a GET request, but if you pass
it data it will make a POST request with that data as the body. This request is
authenticated seamlessly. You can send data of any mark as long as it has a
conversion path to `%mime`.

Let's take a look at the code and see if we can figure out what's happening
here:

    /+    basic-auth

First, we load a library called `basic-auth`. This library exposes two items.
`keys` is the type of the basic auth key, which is essentially a base64'd
username and password. `standard` accepts a `bale` and produces a core with
everything we'll need to implement basic auth.

Every security driver needs some amount of state, which is stored in `bale`.
`bale` contains:

-   `our`, our urbit name
-   `now`, the current time
-   `eny`, 256 bits of entropy
-   `byk`, the urbit, desk and case which the security driver is running from
-   `usr`, the particular identity we're logging in with
-   `dom`, the site we're accessing
-   `key`, the secrets required to authenticate requests. The type for this
    entry is supplied by the programmer as the argument to `++bale`. Thus, in
    our case, `key` in our bale is of type `keys:basic-auth`.

Additionally, a security driver contains at least one out of the five "special"
arms:

-   `++filter-request` is a function which takes a hiss (http request) and
    produces a `sec-move` (defined in zuse), which is one of:

-   `[%send hiss]`, which sends the new hiss. This is the case in, for example,
    basic auth, where all we need to do is add an extra header to the request.
-   `[%give httr]`, where httr is an http response. This immediately returns an
    http response to the sender.
-   `[%show purl]`, where a purl is a parsed url. This displays a message asking
    the user to visit the given url to continue the authentication process.
-   `[%redo ~]`, which redoes the request.

`%eyre` calls `++filter-request` just before sending an HTTP request to the
specified domain. This allows the security driver to filter the requests,
decorating them with authentication data.

-   `++filter-response` is a function which takes an httr and produces a
    `sec-move`. `%eyre` calls it after receiving an HTTP response from the
    specified domain. This allows the security driver to handle authentication
    errors, commonly caused by expired tokens, and retry the request.

-   `++receive-auth-query-string` is a function that takes a `quay` (list of
    query parameters) and produces a `sec-move`. `%eyre` calls it when it
    receives a request on the callback url. Generally, this happens in OAuth
    after the user has granted access to the urbit app, and the service makes a
    request to the callback url with the code. The security driver should then
    make a request to convert the code into an access token.

-   `++receive-auth-response` is a function that takes an httr and produces a
    `sec-move`. `%eyre` calls it when it gets a response to the request made in
    `++receive-auth-query-string`.

-   `++update` is a function which converts old state to a new format when the
    security driver gets updated. If you want to just get rid of the old state,
    define `++discard-state` as `~`.

For basic auth, we only have to worry about `++filter-request`, since all we
need to do is add the correct header to each request. Our `++aut` core contains
a function `++out-addding-header`, which does exactly what we want.

Where do the keys come from, though? Remember we ran `|init-auth-basic` to input
them. This just prompts you for the service url, username, and password, and it
stores them next to the driver. In our case, that's
`/=home=/sec/com/gitub/api/atom`. These keys are stored encrypted, and you don't
want to edit them directly. `%eyre` loads them directly into your `bale`.

## OAuth2

Most services are better accessed through some form of OAuth. Github can be
accessed with OAuth2, which is a little more complicated than basic auth, but
not hugely so.

The basic steps for OAuth2 are these:

-   create an app and get its client id and client secret (on Github)
-   store these in Urbit (with `|init-oauth2`)
-   try to make a request that requires authentication. The security driver
    should prompt you to visit a particular url.
-   visit the url and click "authorize", which gives the app access to your
    account
-   the security driver will turn the code it receives into an access token
-   on every request, include that access token

Creating an app is easy and
[well-documented](https://github.com/settings/applications/new). If necessary,
set the callback url to `/~/ac/github.com/_state/in`, and note its client id and
client secret.

Run `|init-oauth2`, which will prompt for the hostname (api.github.com), client
id, and client secret. This will store the information, encrypted, in
`/=home=/sec/com/github.atom`, just as with Basic Authentication.

Now we need a security driver. Use this:

    ::  Test url +https://api.github.com/user
    ::
    ::::  /hoon/github/com/sec
      ::
    /+    oauth2
    !:
    ::::
      ::
    |_  {bal/(bale keys:oauth2) tok/token:oauth2}
    ++  scopes                          ::  comment out scopes to taste
      :~  'user'  'user:email'  'user:follow'  'public_repo'  'repo'
          'repo_deployment'  'repo:status'  'delete_repo'  'notifications'
          'gist'  'read:repo_hook'  'write:repo_hook'  'admin:repo_hook'
          'admin:org_hook'  'read:org'  'write:org'  'admin:org'
          'read:public_key'  'write:public_key'  'admin:public_key'
      ==
    ::  ++aut is a "standard oauth2" core, which implements the
    ::  most common handling of oauth2 semantics. see lib/oauth2 for more details,
    ::  and examples at the bottom of the file.
    ++  aut  (~(standard oauth2 bal tok) . |=(tok/token:oauth2 +>(tok tok)))
    ++  filter-request
      %^  out-add-query-param:aut  'access_token'
        scope=~[%client %admin]
      oauth-dialog='https://github.com/login/oauth/authorize'
    ::
    ++  receive-auth-query-string
      %-  in-code-to-token:aut
      url='https://github.com/login/oauth/api/access_token'
    ++  receive-auth-response  bak-save-token:aut
    --

The oauth2 library provides the main "engine" in `standard`, just like in basic
auth, except that we also have to specify a function to save the access token
when we get it. It's worth noting that this library is well documented in the
source, including examples: `/=home=/lib/oauth2/hoon`.

Running `+http://api.github.com/user` tries to make a request to Github with
authentication. This loads into the bale the keys, which are of type
`keys:oauth2`, defined in the oauth2 library. We don't have to worry about the
specifics of these keys -- the library handles them -- but they include the
client id and the client secret.

Before the request is sent, `%eyre` calls `++filter-request` to decorate it with
authentication headers. Since we don't yet have an access token, we need to
prompt the user to visit the "dialog" url. `++out-add-query-param` checks
whether we have a token, and if we don't it produces `%show` and a message
telling the user to visit the `oauth-dialog` url we provide, with the extra
query parameters added (client id, scopes, and state).

When the user visits that url, Github will ask them to log in and authorize the
application. When they do so, Github will post a request to
`/~/ac/github.com/_state/in` with a code in the query string. `%eyre` sends the
query string to `++receive-auth-query-string`, which is a function that takes a
quay and produces a `sec-move`. `++in-code-to-token` from our `aut` takes the
`code` parameter from the quay and uses it to create a request to the given url.
This request includes the client id, the client secret, and the code we just
got.

Github's response to that request includes an access token in its body. This is
handled in `++receive-auth-response`. `++bak-save-token` extracts that token and
gives it to the function that we passed in the definition of `++aut`. In our
case, we just save that token into our state for later usage.

Now we have the credentials we need, so `++filter-request` is called with the
original request, and `++out-add-query-param` passes the request through with
the addition of the query parameter `access_token`.

Just like with Basic Authentication, you should be able to run
`+https://api.github.com/user` and get a response indicating who you're logged
in as.

Some drivers a slightly more complicated. For example, Github's access token's
don't expire, but that's not the case for all service. The googleapis driver
shows the flow when the access token can expire. Twitter uses OAuth1, for which
we have another library.

Remember, of course, that the best strategy for building security drivers is to
copy a similar one and tweak it to taste.
