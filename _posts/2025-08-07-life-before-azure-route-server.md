---
layout: post
title: "Life Before Azure Route Server — Why We Still Rely on Route Tables"
date: 2025-08-07
author: Wayne Marks
categories: [Networking]
tags: [Azure, Networking, Route Server, UDR]
permalink: /life-before-azure-route-server/
---

Before we get into Azure Route Server, let’s talk about something that throws nearly every on-premises network engineer off balance — especially if, like me, you come from a Cisco background and the first time you step into Azure: **the never-ending tangle of route tables.**

---

In a traditional world, routes were simple. You had a router. It had interfaces. You knew exactly which side of the network a packet came from and where it needed to go. Routing was often automatic, mostly physical, and rarely touched unless something broke or a new site came online. With all its quirks, it **made sense**.

Then you land in Azure and discover you need to build **User Defined Routes (UDRs)** for what feels like… everything:

1. Want traffic from a spoke to reach your firewall? → UDR.
2. Want to steer traffic through an NVA? → UDR.
3. Want to override a default internet route? → UDR.
4. Want to peer a VNet and keep it secure? → UDR.
5. Using vWAN or ExpressRoute or Private Link? → You guessed it… UDR.

At first glance, it feels like we’ve gone backwards. All this software-defined magic, and we’re still **hand-crafting route tables like it’s 2005**. But there are good reasons for it, and understanding them is the key to designing Azure networks properly.

So why does Azure need UDRs? Why is this “manual routing” still a thing?

---

## 1. Microsoft’s Global Network Isn’t a Flat LAN

![Microsoft Global Network Map](/Azure-Blog/assets/global-network-map.png)

Azure isn’t a big flat layer 2 switch stretched across the world. It’s a **massive collection of peered edge routers**, backbone routers, and isolated data planes. Every region is essentially its own network island.

Azure deliberately limits automatic propagation of routes to avoid accidental routing loops, asymmetric flows, or bleeding traffic between security zones. UDRs give you control. That’s the point.

---

## 2. Azure Route Server Helps — But Only If You Use It Right

Azure Route Server lets your NVA or SD-WAN device participate in BGP without the pain of static routes. It brings **dynamic route injection** to the cloud edge — no more manual updates when you add a new spoke or branch.

But (and it’s a big but) — Route Server does **NOT** remove the need for UDRs. You still need:

- A UDR in each spoke subnet to point `0.0.0.0/0` to your NVA or firewall
- UDRs to route return traffic correctly (especially if asymmetric routing is a concern)

---

## 3. TL;DR — Don’t Expect Magic

Azure networking is **not broken**. It’s just not what you’re used to.

UDRs feel manual — because they are. But they’re also **deliberate**, **controllable**, and **predictable**.

Azure Route Server reduces the operational pain, especially in dynamic environments — but it doesn’t remove the need for thinking like a network engineer.

---

## Stay Tuned

In the next post, I’ll dive deeper into Azure Route Server — how it works, how to deploy it, and why ECMP still trips up even experienced architects.

[Back to blog homepage](/Azure-Blog)
