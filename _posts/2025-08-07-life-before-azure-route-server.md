---
layout: default
title: "Azure Networking: Why We Still Rely on Route Tables"
date: 2025-08-07 10:00:00 +0100
categories: azure networking
permalink: /azure-networking-why-we-still-rely-on-route-tables/
---

![Matrix-style Microsoft wizard](/Azure-Blog/assets/images/Matrix.webp)

PoC v1.1  
Before we get into Azure Route Server, let’s talk about something that throws nearly every on-premises network engineer off balance — especially if, like me, you come from a Cisco background.

That moment you step into Azure and discover the **never-ending tangle of route tables** feels like a step backward. 

In the traditional world, routing was physical, direct, and often automatic. It just worked — until something broke. In Azure, you’ll quickly realise:

- Want traffic from a spoke to reach your firewall? = UDR  
- Want to steer traffic through a NVA? = UDR  
- Want to override a default route? = UDR  
- Peered VNets? VPNs? Private Link? = UDR  

Suddenly, it’s 2025 and we’re **hand-crafting route tables like it’s 2005**. But it’s not madness. There’s a method.

---

## 1. Microsoft’s Global Network Isn’t a Flat LAN

![Microsoft Global Network Map](/Azure-Blog/assets/images/MSFT%20GB%20Network.webp)

Microsoft owns one of the world’s most advanced private networks. Over 180 edge sites, spanning continents. 

That traffic from a spoke in UK West to UK South? It stays inside Microsoft’s backbone.  
It never touches the public internet.

But here’s the catch: **Azure assumes global reachability is a good thing.**

Which it is — until you need inspection, policy enforcement, or security boundaries.  
That’s where UDRs come in. They give you back **control of the route**.

---

## 2. Software-Defined Networking Doesn’t Care About Physical Rules

Your next hop could be in a different subnet. A different zone. A different region.

There’s no physical router reading the packet. It’s the **Azure fabric** that decides the path, based on:

- Metadata  
- NSGs  
- Route tables  
- BGP advertisements  
- Service tags  

You don’t plug in a cable to fix it. You write a UDR. That’s how you make logic apply to a network that doesn’t physically exist.

---

## 3. Azure’s Routing Stack is a Layered Cake

Azure doesn’t have just one routing table. It’s a blend of:

- **System routes** – Built-in defaults for every VNet  
- **User Defined Routes (UDRs)** – The ones you create per subnet  
- **BGP routes** – From VPN Gateway or ExpressRoute  
- **Peering routes** – For VNet-to-VNet connections  
- **Private Link routes** – For services pulled into your VNet  
- And others...

Only UDRs let you override the rest.  
**Ignore them, and you’re relying on assumptions** — which works… until it doesn’t.

---

## 4. This Isn’t Physical Kit — You’re Simulating a Network

You’re not racking switches anymore.

There’s no MAC address table to interrogate or cable to trace. You’re building a **logical** design inside a software platform.

UDRs are how you:

- Steer spoke-to-spoke traffic through an inspection zone  
- Restrict access to a storage subnet  
- Force internet egress via a proxy  

In other words, UDRs are the brake and steering wheel in this car.  
**And Azure won’t drive it for you.**

---

## The Bigger Picture

User Defined Routes aren’t elegant.  
They don’t scale easily.  
They’re not pretty.

But they are *necessary*. And once you understand why they exist, you can start designing networks that behave exactly how you want them to.

And that’s what makes Azure Route Server so interesting — because what comes next might just make all of this... easier.

Stay tuned.
