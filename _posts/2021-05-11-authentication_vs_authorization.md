---
layout: single
title:  "Authentication vs authorization"
date:   2021-05-11 21:15:00
categories: others security
author_profile: true
classes: wide
tagline: " "
header:
  overlay_image: /assets/images/faris-mohammed-d30sszrW7Vw-unsplash.jpg
  particles: true
---
Both terms are very familiar to me however I hesitate for a second every time when asked for the difference. The reason for the hesitation
is that I am not an English native speaker and both terms are looking very similar (at least it is my excuse for myself).
And yet the difference is so simple.

## Authentication
Authentication is a process of proving that somebody is who he says he is. Everybody can state he is Gordon Freeman but without proof, 
you can not trust him. Authentication emerges to help. The whole authentication process can be done as we know it well, by a username 
and a password. In a case of a state administration, we need an ID, passport birth certificate, or more of listed.

## Authorization
Authorization is a process of proving that somebody can do what he wants to do. This time we can compare it to a situation of buying
an alcoholic drink. There are rules that restrict buying alcohol under a certain age. If the person is older he is authorized to buy an
alcoholic drink. Usually, the claim is proved by a part of our identity (in this case birthdate from an ID).

### How it is connected
Let stick to the previous example of buying an alcoholic drink. I can give the ID of my older brother to the bartender. The ID authorizes 
buying the drink. Problem is that bartender does both authentication and authorization. If you look like your brother, you are probably 
OK because you borrow the identity of your brother, and the identity authorizes you to buy a drink. On the other hand the bartender
can notice the fraud, authentication fails and authorization is not even done.
From the previous information we can see that the authentication is first and very important step for authorization...