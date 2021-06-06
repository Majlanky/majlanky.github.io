---
layout: full_single
title:  "OAuth2 and OpenId Connect"
tagline: " "
header:
  overlay_image: /assets/images/hannah-mattern-AHd02RYlyic-unsplash.jpg
  particles: true
date:   2021-05-28 22:00:00
words_per_minute: 70
categories: others security
mermaid: 
  enabled: true
  theme: neutral
---
First, let me state that I am not an Oauth2, nor an OpenID Connect expert. I just did a lot of research about both to be sure I am using it
properly (that I am not using it improperly is more precise). That we will dive deeply into only a few parts is based on the fact. Maybe you read
some articles, and you are confused or looking for information that fills the gap. Hopefully this article is just for you.
**Purpose of the article is to provide first (or last) steps and "hard to find" information using a story-like approach.**

## What is OAuth2

OAuth2 is standardized protocol for authorization (if you are lacking in the terminology see my [post]({% post_url 2021-05-11-authentication_vs_authorization %})). 
What is important to say is that the scope of OAuth2 is **only** authorization. It means that OAuth2 provides ways how to authorize an entity (human, service, etc.)
to another entity.

## What is OpenID Connect

OpenID Connect is a top layer above OAuth2 solving the missing authentication feature. We will talk about how it is done later.

## What is the OAuth2 idea

I will not be using the proper terminology (I will come back to it later) because I found OAuth2 terminology a little confusing without knowledge of OAuth2
itself. In the following diagram, let's consider you have an account at Google and you are using its calendar. `App` stands for an application that optimizes your lifetime.

```mermaid
sequenceDiagram
    You->>App: I want you to optimalize my calendar
    App->>Google: Hey I need to access unknown users calendar
    Google->>App: Ok, let me talk with the user
    App->>You: Google wants to talk with you
    You->>Google: Ok Google, here I am speaking
    Google->>You: Authenticate yourself
    You->>Google: I am XXX
    Google->>You: Ok, do you want App can work with you calendar?
    You->>Google: Yes it is exactly what I want
    Google->>App: Ok, the user authorized you to work with his the calendar. Here is the key card  
```

You can see App does not know who the user is. App just received the key card to work with the calendar. In pure OAuth2 implementation, App does not receive any
information about the user (no identity), just some keys to access a non-contextual calendar. You can imagine the key card as a medium. The key card is
impossible to reprogram, so the holder of the card is able to do only permitted things with permitted resources.

### Let's add the proper terminology

In the context of the previous example:

* You = Resource owner (resource is the calendar)
* App = Client (Here we are talking about the application from OAuth2 point of view so App is OAuth2s client)
* Google = (Google stands for two parts here. It is able to identify user, and it holds calendar data)
  * Authorization server (The part that authorizes App to do things)
  * Resource server (The part where data of calendar are stored)
* The key card is implemented as something called *token*

## How OAuth2 works

We have an idea of how OAuth2 does things. Now let's dive deeper to understand how it is working. The first thing to examine is gaining permission meaning
token. Then we will make clear other aspects to reach the "aha moment".

### Types of clients
If we talk about clients here, we mean OAuth2 clients. There are two types of clients in OAuth2:
* Public
* Confidential

In OAuth2 every client has to authenticate to an authorization server. Even Google must know every client that wants to use it as an authority.
For this purpose, every client is identified at least with *client_id*, and some have *client_secret* too.

#### Public
If the client is of *public* type, *client_id* is whole identification. Public clients are those who are not able to protect *client_secret* from reading,
hence there is no secret for this type of client. These types of clients must gain access more difficult way as compensation to identifying by public
*client_id* (anybody can easily get the *client_id* and use it during a malicious attack).Public clients are [SPAs](#spa) for example.

##### Confidential
In case of *confidential* client the *client_id* and *client_secret* pair is identification. Yes, you feel the analogy with username and password right here.
This type of client can communicate with an authorization server confidentially thanks to *client_secret*. *Client_id* and *client_secret* must be secured
and treated as username and password!
Confidential clients are server-side parts of any application (we can call the server, service, back-end, etc.).

***

### Communication

For a better understanding of why OAuth2 does things as it does, we must take a look at communication between resource owner, client, and authorization server.
The OAuth2 communication is based on HTTP redirects. Redirect is special HTTP status (303 or 302) that tells a browser to send the request to the URL given in
one of the given headers. Thanks to the redirect authorization server are able to *force* resource owner's browser to send requests to the client's URL (where parameters contain data from the authorization server). If you do not understand the information completely just remember there is this explanation and come back later.

All redirecting is done at so-called [*front* channel](#front-channel). The front channel is the observable part for example browser. When I am talking about
observable I mean by 3rd party library, malicious web-browser addon, etc. Sources flowing through the front channel should be heavily protected or moved
to [*back* channel](#back-channel). The back channel is the part of an application that is non-observable for example server.

### Gaining an access

To gaining access OAuth2 defines so-called *flows*. Flow differs by usage. We will discuss later when to use which flow. In this part, I will change to
OAuth2 terminology because I do not want to make it messy. In the parenthesis, I refer to the first diagram as a hint.  
The goal of a flow is to gain an access token. The token is the *key card* from the first diagram, and it is called access because of its purpose.

There are all available flows with the example of usage:

* [Code](#code)
* [Code with Proof Key for Code Exchange (PKCE)](#code-with-proof-key-for-code-exchange-pkce)
* [Implicit](#implicit)
* [Client credentials](#client-credentials)
* [Resource owner password credentials](#resource-owner-password-credentials)

I will start with the simplest (and less secure) flow and will be proceeding to the more difficult (and more secure) ones.

#### Resource owner password credentials

This flow is a kinda "make obsolete application works" solution. The main idea of OAuth2 is to keep resource owners (*You* in the first diagram)
credentials safe but this flow goes against the idea. In this flow client (*App* in the first diagram) gathers the credentials, sends them to the
The authorization server (*Google* in the first diagram). The authorization server responds with an access token.

```mermaid
sequenceDiagram
    Resource owner->>Client:Here are my credentials
    Client->>Authorization server:Here are credentias of resource owner
    Authorization server->>Client:Ok here is the access token
```

As you can see the flow is very simple and credentials very unprotected. This flow should be used as a margin case of backward compatibility. Ideally do not
use it at all.

#### Implicit

This flow is kinda deprecated and replaced by [PKCE](#code-with-proof-key-for-code-exchange-pkce) but you can meet it so let's take a look.
Usage of the flow is for [*SPA*](#spa) where a client is able to keep the gained access token more or less securely.
```mermaid
sequenceDiagram
    participant Resource owner
    Client->>Authorization server:A resource owner would like to give me access
    Authorization server->>Client:Redirect the user to the URL
    Client->>Resource owner:Go to the URL and confirm the access for me
    Resource owner->>Authorization server:Here are my credentials and allow the client access
    Authorization server->>Client: Here is the access token
```
One important thing to know that the "go-to URL" part is done by redirection (303 or 302 HTTP status). From a security point of view
everything is done on [*front channel*](#front-channel) hence access token is not secured as much as it can be in this flow
(see [PKCE](#code-with-proof-key-for-code-exchange-pkce) for more secured way to deal with SPA)

#### Code
This is the most common flow if the OAuth2 client is not [*SPA*](#spa) but a website with a backend part. The website with backend part means that backend provides
simple web UI (not [*SPA*](#spa)) but token authorized access to resource server is done from backend part hence backend is holding all access tokens
(for example in user sessions). Now let's focus on the flow.
```mermaid
sequenceDiagram
    participant Resource owner
    rect rgb(200, 50, 50, .3)
    Client->>Authorization server:A resource owner would like to give me access, let me know here (URL A)
    Authorization server->>Client:Redirect the user here (URL B)
    Client->>Resource owner:Go to the URL B and confirm the access for me
    Resource owner->>Authorization server:Here are my credentials and allow the client to access
    Authorization server->>Resource owner: Here is the code to get access token, redirect it to the URL A
    Resource owner->>Client: Here is the code to get an access token
    end
    rect rgb(30, 220, 60, .3)
    Client->>Authorization server: Here is the code and secret, gimme an access token
    Authorization server->>Client: Here is the access token
    end
```
What can be confusing is the last three messages. Why the heck the authorization server returns some code and making obstacles?
Well, everything is done in a matter of security. A clue can be the color differentiation and [communication](#communication) chapter.
The red part is done on [*front* channel](#front-channel) and the green part is done on [*back* channel](#back-channel). With flow like this,
the access token never touches the front channel, so there is no way for leaking. What more the code by itself is useless hence an access token is returned
only when the code is combined with the client secret (we talk about [here](#types-of-clients)). Because we need a client secret, client must be
[confidential](#confidential).

#### Code with Proof Key for Code Exchange (PKCE)
First ensure you read [Code](#code) flow first. The code with PKCE flow is very similar to the code flow with one major difference. The client, in this case, is [public](#public) so we do not have a client secret. OAuth2 must secure access tokens differently, and the PCKE is the solution. Let's
see the flow in a diagram:
```mermaid
sequenceDiagram
    participant Resource owner
    Client->>Authorization server:A resource owner would like to give me access, let me know here (URL A)
    Authorization server->>Client:Redirect the user here (URL B)
    Client->>Resource owner:Go to the URL B and confirm the access for me with this challenge
    Resource owner->>Authorization server:Here are my credentials and allow the client to access
    Authorization server->>Resource owner: Here is the code to get access token, redirect it to the URL A
    Resource owner->>Client: Here is the code to get an access token
    Client->>Authorization server: Here is the code and verifier, gimme an access token
    Authorization server->>Client: Here is the access token
```
There are no color boxes because everything is done on [*front* channel](#front-channel). In the code with PKCE flow client proofs that used code
was retrieved on the client's request (that it was no stolen). It is done with code challenge and code verifier. Both values are cryptographically
secured (description how is out of scope but there are many tutorials on the web) and the authorization server is checking that hashed verifier
matches the original challenge.

#### Client credentials
This flow is designed to get access tokens for accessing a client's own resources. The whole communication is done on [*back* channel](#back-channel) 
(OFC there are no protection against using the flow for example for SPA but it is a big no-no as clients credential and tokens are flowing on [*front* channel](#front-channel))
Flow is very straight:
```mermaid
sequenceDiagram
    Client->>Authorization server:Here is my client id and secret, gimmer access token
    Authorization server->>Client:Here is the access token
```

## Notes
### Why so many worries about securing an access token?

As I mention in the [post]({% post_url 2021-05-11-authentication_vs_authorization %}), authentication and authorization are usually connected.
The access token is the case when authorization is detached from authentication. Access token authorizes the holder to some resources but nobody cares about authenticating the holder. If an access token leaks from cookies or local storage or is captured during a MITM attack, the acquirer can use it
without any obstacle.

### Where the OpenId Connect is in the whole thing

I was talking about the OAuth2 the whole time. It is because OpenId Connect lies on OAuth2 so the process is the same. The difference is that instead of
gaining an access token, OpenId Connect adds an ID token in the game. The ID Token is added to the same response as the access token. The ID token
is immutable and contains information about the resource owner hence can be used for authentication (and must not be used for authorization).

## Other resources

Amazing resources you can check if you feel you need another angle of view to the topic:
* [Presentation](https://www.youtube.com/watch?v=996OiexHze0) about OAuth2 and OpenID Connect by OktaDev
* Already mentioned documentation by [OktaDev](https://developer.okta.com/docs/)

## Explanation of used terms:
### Authentication
See the [post]({% post_url 2021-05-11-authentication_vs_authorization %})
### Authorization
See the [post]({% post_url 2021-05-11-authentication_vs_authorization %})
### Front channel
The term describing an observable (from an attacker's perspective) part of an application. A browser for example is the front channel because cookies and communication can be easily hijacked by 3rd party library (and others). The non-observable part is called *back channel*
### Back channel
The term describing a non-observable (from an attacker's perspective) part of an application. A server for example is back channel because there is much less or no possibility of malicious addon or 3rd part library. The observable part is called *front channel*
### SPA
It is a shortage of a single-page application. Single-page applications are applications running in a browser that communicates without reloading
the browser page.