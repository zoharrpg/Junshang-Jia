+++
title="CMUD"
date=2023-12-19

[taxonomies]
categories = ["CMU-15640","Distributed System"]
[extra]
toc = true

+++

# Overview
CMU-15640 Distributed Systems Project 3<br>
In CMUD, players progress through the ranks of CMU computer science by mining coins,
improving their hardware, and solving hard problems. The game is a Multi-User Dungeon
(MUD), meaning that all players inhabit the same virtual world and interact with it through
a text terminal.


Unlike a traditional MUD with a single application server, CMUD uses the globally dis-
tributed architecture shown in Figure 1. Players run the CMUD client app on their own
machine, which connects to a nearby (low ping) backend storage server. The CMUD client
is stateless, converting the player’s inputs into queries against the backend storage server
and printing out the results. Different backend storage servers “sync” with each other (i.e.,
exchange data in the background) to maintain a single consistent global state, so that all
users see the same virtual world, including each others’ actions.

The backend storage system itself implements a simple key-value store for string keys and
values, similar to Project 0. Specifically, it supports operations to Get and Put key value
pairs, plus an operation to List all key-value pairs starting with a given prefix. Since this
API is very simple, all application logic happens on the client, unlike the classic three-tier
architecture with separate application and backend servers; see Figure 2.
The advantage of this architecture is that it simplifies app development and deployment:
a cloud provider can run the same generic backend storage system for many apps, while
individual app developers only need to code and deploy their app’s client, not their own
servers.

[Source Code Link](https://github.com/zoharrpg/CMUD)<br>
{{ render_link(path="15640-doc/p3_23.pdf",text = "Write-Up") }}