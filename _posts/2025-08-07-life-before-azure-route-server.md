---
layout: default
title: "Life Before Azure Route Server"
date: 2025-08-07 10:00:00 +0100
categories: azure networking
permalink: /life-before-azure-route-server/
---

<p align="center">
  <img src="/Azure-Blog/assets/images/Matrix.webp" alt="Matrix-style Microsoft network wizard" width="500"/>
</p>

Before we get into Azure Route Server, let’s talk about something that throws nearly every on-premises network engineer off balance — especially if, like me, you come from a Cisco background.

The first time you step into Azure, you hit:  
**the never-ending tangle of route tables** — and what at first seems like a backward step in networking.

---

In a traditional world, routes were simple. You had a router. It had interfaces. You knew exactly which side of the network a packet came from and where it needed to go.

Routing was often automatic, mostly physical, and rarely touched unless something broke or a new site came online. For all its quirks, it *made sense*.

---

Then you land in Azure and discover you need to build **User Defined Routes (UDRs)** for what feels like… everything:

1. Want traffic from a spoke to reach your firewall? → UDR  
2. Want to steer traffic through an NVA? → UDR  
3. Want to override the default internet route? → UDR  
4. Want to keep peered VNets secure? → UDR  
5. Using VWAN, ExpressRoute or Private Link? → Yep… UDR again.

---

At first glance, it feels like we’ve gone backwards. All this software-defined magic — and we’re still  
**hand-crafting route tables like it’s 2005.**

But there are good reasons for it. And understanding them is key to building Azure networks that actually behave how you expect.

So why does Azure need UDRs?  
Why is this “manual routing” still a thing in 2025?

---

*Let’s get into it…*
