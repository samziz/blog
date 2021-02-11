---
layout: post
title: "Representing graph structures in data"
date: 2021-02-08
categories: Foundations
---

I'm currently collaborating on a project to map out the entirety of social media - or at least common sites such as Twitter and Facebook - as a graph of interconnected human nodes. Or technically a set of graphs with a monomorphism mapping each to the other. But that's not important right now. What _is_ important is the question of how we store this graph.

The fact is that most modern databases are not well suited to storing graph-structured data. I would contend that modern databases are [not suited to a _lot_ of things], but that's an argument for another day. A graph data structure is _inimical to_ the modern database.

First, let's recap the situation we're in when it comes to databases.

## A potted history of the modern database

Once upon a time,

We alighted on SQL more or less as we alighted on the modern CRUD app. For SQL is extraordinarily well suited to the modern CRUD app, where the following invariants obtain:

- **There are a few 'domain objects'** such as User, Session, and finally the domain object which distinguishes your CRUD app from all the other CRUD apps: let's say PuppyPettingAppointment.
- **Most queries are get-object-by-ID lookups**. These are 
