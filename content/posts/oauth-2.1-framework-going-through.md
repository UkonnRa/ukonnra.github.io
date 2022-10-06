---
title: "OAuth 2.1 Framework Going Through"
date: 2021-12-04
description: Go through the OAuth 2.1 framework specification
tags: ["oauth2", "oauth2.1", "authorization", "specification"]
categories: ["authorization"]
series: ["OAuth Going Through"]
draft: true
---

## Pre Introduction

Since [Spring Authentication Server](https://github.com/spring-projects/spring-authorization-server) is under active development, it's time to go through the authentication and authorization techniques behind. Maybe [its Wiki](https://github.com/spring-projects/spring-authorization-server/wiki/OAuth-2.0-Specifications) is a good start.

To be clear, I don't want this article (or this series, if having) to be a coarse imitation of the official specification, but interested more in the total framework and design, with general use cases.

So let's move on to the first piece, [OAuth 2.1 Specification](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-04)!

## Introduction

How to design a secure service to interact with the untrusted outer world? OAuth's specification gives us a great workflow.

Say a third-party client wants to access data from a resource server, it should ask the resource owner for granted and receive access tokens from the authorization server.

{{< mermaid >}}
sequenceDiagram
  autonumber
  participant C as Client
  actor RO as Resource Owner
  participant AS as Authorization Server
  participant RS as Resource Server

  C->>RO: Ask for grants to access resources
  RO->>AS: Client redirects
  AS-->>C: Auth process finished. Return Access Token
  C->>RS: Send requests with Access Token
  RS-->>C: Check the Access Token. If passed, returns the resources
{{< /mermaid >}}

### Authorization Code

In detail, Step 3, the auth process, can be split into several steps.

{{< mermaid >}}
sequenceDiagram
  autonumber
  actor RO as Resource Owner
  participant AS as Authorization Server
  participant C as Client

  RO->>AS: Finish the identity checking
  Note over RO,AS: Password or user managers (Google, QQ ...)
  AS-->>RO: Ask for permissions
  RO->>AS: Finish the consent step
  Note over RO,AS: Only permissions granted can be effected
  AS-->>C: Send Authorization Code (not Access Token)
  C->>AS: Request Access Token with Authorization Code
  AS-->>C: Return the Access Token
{{< /mermaid >}}

After the consent step, the Authorization Server returns a temporary Authorization Code. Then Client uses an extra one to exchange it for the usable Access Token. **Why do we need this extra exchange step?**

* The redirection to Authorization Server always uses user agents, which may not secure
* Client and Authorization Server uses a separate session for minimal exposing. The access token should not be exposed to others, even to Resource Owners using Client

### Refresh Token

For security reasons, **Access Token always expires fast**, say 1 hour, to prevent the damage when leaking. Secondly, every time the Client requests resources from Resource Server, Access Token is always needed. **Access Token leaking** does happen often.

To decrease the risk of leaking Access Token, let's introduce **Refresh Token**.

When Authorization Server returns Access Token, it sends Refresh Token as well. Refresh Token _has a longer expiration time_ (more than 14 days), and it will _not expose that much often_ than Access Token. We only use Refresh Token when asking for a new Access Token (say every 1 hour). So the probability and danger of leaking are far lower than Access Token.

## Clients

Clients should be registered into the Authorization Server before using it. There are 3 types of clients in OAuth 2.1:

* `confidential`: Have client credentials; a prior relationship with AS
* `credentialed`: Have client credentials; but not a prior relationship with AS
  * AS should not THAT trust this kind of clients
  * Leakage and abuse of credentials may happen as well
* `public`: Without credentials

### Client Authentication

As we described before, Clients usually ask resource owners for grants to access resources, and a malicious client may do as well. So for confidential and credentialed clients, it's Authorization Server's responsibility to check whether the clients are valid, registered clients. After all, these kinds of clients have more permissions than the public ones.

Client Authentication may happen when exchanging from the authorization code to access tokens. It's for:

* Create the binding between _refresh tokens & authorization code_ and clients they are issued to
* Handle the malicious clients by changing the client credentials. It's easier than revoke existing refresh tokens already issued
* A layer to do best practices like periodic credential rotations.

Like we do authentication for customers (IAM), we need clients to provide client IDs and secrets to do the identification. Instead of transmitting client id and secret directly using HTTP requests, some additional authentication methods like [mTLS](https://datatracker.ietf.org/doc/html/rfc8705) and [private_key_jwt](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-04#ref-OpenID) are more recommended.
