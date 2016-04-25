---
layout: post
title:  "理解 Restful API"
categories: Architecture
---

**R****E**presentational **S**tate **T**ransfer ([REST](https://en.wikipedia.org/wiki/Representational_state_transfer)) 于 2000 年由 Roy Fielding 在其博士论文 [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) 提出。


[](http://www.restapitutorial.com/lessons/whatisrest.html)
# Overview

REST = "REresentational State Transfer"

- Resource-based
- Representations
- Six Constraints
  - Uniform Interface
  - Stateless
  - Client-Server
  - Cacheable
  - Layered system
  - Code on demand

# Resource Based

- Things vs action
- Nouns vs verbs

Resource-based or nouns based instead of action or verb based

# Representation

- Representation is not the resource, it's the how resources get manipulated
- Part of the resource state
- Typically JSON or XML
- Transfor between client and server

# Uniform Interface

- Defines the interface between client and server
- Simplifies and decouples the architecture
- Fundamental to RESTful design
- It means
  - HTTP Verbs(GET, PUT, POST, DELETE)
  - URIs(resouce name)
  - HTTP response(status, body) 

# Stateless

- Server contains no client state
- Each request contains enough context to process the message
- Any session state is held on the client

# Cacheable


# Layered System

- Client can't assume direct connection to server
- Software or hardware intermediaries between client and server
- Improves scalability


# Code on Demand

optional


# Compliance with REST constraints allows

- Scalability
- Simplicity
- Modifiability
- Visibility
- Portablility
- Reliatblilty




![](http://7xp2eu.com1.z0.glb.clouddn.com/restresourceaction.png)

![](http://7xp2eu.com1.z0.glb.clouddn.com/resturimethod.png)
