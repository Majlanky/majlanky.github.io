---
layout: full_single
title:  "Why not to use environment based spring profiles"
tagline: " "
header:
  overlay_image: /assets/images/zdenek-machacek-XFUrNtSWFCg-unsplash.jpg
  particles: true
date:   2022-07-28 22:15:00 TODO
words_per_minute: 70
categories: java spring configuration
mermaid:
  enabled: true
  theme: neutral
---
:warning: If you are using K8s, ArgoCD and similar it probably led you to proper way already. 
If you are just moving to K8s based environment, you can  find nifty information here. :warning:

Everybody knows Spring profiles and many things was said about it especially how to use it from the technical point of view. Similarly, as 
in other articles I will not be focusing on the technical point of view because I expect that you know or can find this information easily.
What I would like to focus on is what I see very often and which makes a lot of problems more or less, sooner or later. We can list issues 
first and then dive deeper in to the topics.
* Executable application first
* Environment based profiles

## Executable application first
Image or remember the situation when you clone a new project, or you are returning to the old one in new IDE, or similar situation. You clone the
repo (smooth), open IDE, and it prepares basic run configuration for you (still smooth), you try to run application after some basic 
research, it fails (not so smooth), you realize there must be some DB or something, run included docker-compose (in the best case it is there), 
or you run database by your self, start the application, and it fails again (dang it). What I am pointing at is that it would be nice to make 
the application executable by default. My approach is to provide `docker-compose.yml` in directory like `env/local` and 
prepare configuration of app for this (local )environment. just configure it like no other than the local one exists. So you will have only one
`application.yml` in resources and that's it.
There are some reasons why:
* The local is the first needed (it is used for development)
* I do not know about what environments there are or will be (you can name some but who really knows). To be more precise I do not want to know it...
* Even when I am able to name some environments, Am I able to say what is the exact configuration for it? Usually not.
* Only environment configuration I can influence and know well is the local one

All of listed are arguments for forgetting about any other environment during development. In the following chapter we will 
find what will happen when we use environment related profiles and what the approach brings.

## Environment based profiles
One of many solutions I saw is to use spring profile per environment. If the project use YAML configuration there are files like:
* application-local.yml
* application-devel.yml
* application-acc.yml
* application-prod.yml

or something similar depending on type and number of environments. This looks like awesome idea because you can configure each environment
independently on the others. Yes there are some margin cases like secrets and others, but it can be solved by some mock value with rewrite 
directly on machine or use old [new import feature](https://spring.io/blog/2020/08/14/config-file-processing-in-spring-boot-2-4).
What becomes troublesome is when you discover you can not find the truly used value. Let's assume that our production configuration in repository
looks like (`application-prod.yaml`)
```yaml
spring:
    datasource:
        url: jdbc:mariadb://weWillSee:3306/myCoolDb
        username: notRootForSure
        password: whoKnows
        hikari:
          connection-timeout: 10000
          minimum-idle: 10
          maximum-pool-size: 10
          idle-timeout: 10000
          max-lifetime: 1000
    security:
        oauth2:
            resourceserver:
                biz:
                    issuer-uri: http://somethingLikeFacebook/auth
                    jwk-set-uri: http://somethingLikeFacebook/auth/protocol/openid-connect/certs
```
And now how it looks on production machine (for example file with the same name in config directory which then overrides/extends the one in the jar):
```yaml
spring:
    datasource:
        url: jdbc:mariadb://myRealDbUrl:3306/myCoolDb
        username: myProdUser
        password: "**********"
        hikari:
          connection-timeout: 8000
```

Now some bug or weird behavior (let's say regarding authorization) will appear. What is the used IdP on production. You go to check the 
`application-prod.yml` in repo. Ok it looks like somethingLikeFacebook we are using. But wait, it can be overridden on machine. Ok lets check it 
there. Ok it is not overridden. Is it? It is in this case but the uncertainty is poisonous. 
Another issue can be when you override just part of complex configuration like the hikari in the example. Who do not know how tricky configuration
of Hikari is, let me tell you just simply all values are connected together. You change one and some others are ignored etc. Now if you basically
split configuration like this to two files, how you can easily check how it is configured.

Solution is simple, just do not use environment related profiles and override whole configuration. Then you can see whole configuration in one place
with high level of certainty, you will not get into any troubles when new environment appears.

## Postscript

By the way when I wrote the last sentence in the previous chapter I realized that I see very often that the environment related profiles are used
for logging too. And it is exactly the place when new environment start to be very inconvenient because it is not so easy override/extend-able hence
it will get cumbersome.


