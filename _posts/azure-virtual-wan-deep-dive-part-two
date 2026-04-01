---
layout: post
title: "Azure Virtual WAN Deep Dive Part Two"
date: 2026-04-01
categories: [Azure, Networking, Virtual WAN]
---

## WOW, it was September when I wrote part one!

My apologies for waiting until April, some months later to finish the part two. In truth I've needed the break from just about everything following a mental health breakdown. THE most import message I can pass in any blog is, if you feel under pressure, burn out or there is no way out - trust me there is, speak up and you will find there are more people there for you than you realise.

---

## Part Two:
## Azure vWAN – ExpressRoute, Security, and Real-World Scale

## 6. ExpressRoute in vWAN: Global Reach and the Hidden Gotchas

If you have ever designed a hybrid network with ExpressRoute, you know the drill. Plan the circuit, choose the peering type, match bandwidth to expected throughput, then connect it to your VNet via a gateway and build the routing rules around it. The whole process is designed for precision. The moment you introduce Virtual WAN into that architecture, the entire shape of the hybrid model changes. The ExpressRoute circuit no longer connects to a VNet gateway. It connects directly into the vWAN Hub. At first glance, that looks simpler. In reality, it introduces a new set of behaviours that you need to fully understand before declaring it ready for production. Because this is not VNet peering. This is backbone integration. And it behaves differently.
 
The ER Gateway Is Now a Hub Service
When you add an ExpressRoute gateway to a vWAN Hub, it becomes part of the hub’s managed services. You do not control the subnet. You do not manage the BGP sessions directly. You configure the connection via the portal or template, and Azure handles the provisioning behind the scenes. The routes received from your on-premises router are propagated into the vWAN hub routing tables. If you associate a VNet or a VPN site with a route table that includes those ER-learned routes, they become reachable. You can also propagate routes from Azure back into the ER circuit, depending on how you structure propagation settings.
 
This transitivity is powerful. It lets you route traffic between:
•	On-premises and VNets
•	On-premises and VPN-connected branch sites
•	Multiple ER circuits across regions
•	Different regions over Microsoft’s global backbone
But none of that happens automatically. You have to architect it deliberately.
 
Circuit Ownership and Connectivity Scope
An easy mistake is assuming that a single ExpressRoute circuit, once connected to vWAN, is available to all hubs globally. It is not.


Each ER circuit connects to a specific vWAN hub. If you want global connectivity, you must either:
1.	Peer multiple hubs together using vWAN’s hub-to-hub backbone
2.	Use Global Reach to link ExpressRoute circuits outside of Azure
3.	Duplicate connectivity across regional hubs using multiple circuits
This means you must carefully plan which circuits terminate where, and whether they need to see routes from other regions. It is tempting to treat vWAN as a global mesh, but ExpressRoute entry points still matter. Latency, compliance, and failover all depend on where the circuit lands.
 

Bandwidth and Cost Surprises
One thing that often gets missed is how the pricing model shifts. In a standard ER Gateway setup, ingress from on-prem to Azure is free. Egress is metered based on region and bandwidth.
In vWAN, egress between VNets and ExpressRoute-connected sites is also charged. But now the path might traverse multiple regions, or backhaul through a different vWAN hub, depending on your routing configuration. You might be pushing traffic into the ER circuit from regions you did not account for, which can introduce higher egress costs than expected. Traffic inspection also becomes more expensive if it routes through an Azure Firewall in another region before hitting ExpressRoute. The design may be clean, but the data charges can stack up quickly.
 
Route Control and Filtering
The vWAN hub routing tables behave the same way regardless of the connection source. Whether the route is learned from a VPN site, an ER circuit, or a VNet, you decide which tables it propagates into and which connections associate with those tables. But unlike standalone ER Gateways, vWAN does not give you granular route filters per peer. If you want tight control over what on-premises sees, you need to manage that on the physical router or use custom route tables within the hub to isolate paths. You can also inject static routes if needed, but again, this should be the exception. Dynamic propagation is the strength of the platform. Manual routes are fine for edge cases or isolation testing, but not as a control plane substitute.
 
Routing Intent and ER
If you enable Routing Intent in a hub that also has ExpressRoute, be aware that the behaviour changes. By default, Routing Intent can push traffic towards Azure Firewall for inspection. If you are not careful, this includes traffic heading towards your on-prem network. If that firewall lives in a different region, you now have a suboptimal path, longer latency, and a potential cost increase. My preference is to avoid enabling Routing Intent on ER-connected hubs unless you are absolutely certain about the direction of each flow and have confirmed the firewall is correctly placed. In highly sensitive hybrid designs, I often rely on custom route tables instead, with a very deliberate propagation model.
 
A Fictional Customer Example
I worked with a customer that had two ExpressRoute circuits: one in South Africa North and one in UK South. Their original plan was to centralise traffic inspection in UK South and allow South African traffic to route through that region before returning to on-prem. The problem was, their workloads in South Africa had latency-sensitive dependencies. By forcing traffic through a UK hub, performance dropped below acceptable thresholds. Moving the inspection point back into the South Africa hub, using Azure Firewall Premium with DNS proxy enabled locally, restored the performance they needed. The takeaway was simple. Just because vWAN allows global routing, it does not mean you should centralise inspection or connectivity.
 

Failover Strategy
You cannot stretch a single ExpressRoute circuit across multiple hubs. If you want failover, you need:
•	A second ER circuit in another region
•	A second hub with its own ER Gateway
•	Matching BGP configuration on both circuits
•	Propagation and association rules that allow for fallback routing
 
Failover testing must be planned. This is not automatic. If your on-prem routers do not advertise the same routes with the correct attributes to both hubs, failover will either be partial or silent.
ExpressRoute inside vWAN is extremely powerful, but only if you design with care. You are no longer connecting to a single VNet. You are entering a distributed backbone, where the shape of your routing and the location of your circuits determine everything. In the next chapter, we step into security. Now that we have traffic flowing from branches, VNets, and on-prem, how do we secure it? What role does Azure Firewall actually play in vWAN? And when should you break out and use something else? 

## 7. Security in vWAN: Firewalls, Policies, and Zero Trust Backbones

Security in a Azure hub was straightforward, if a little clunky. Drop in a third-party NVA, bolt on a route table, force traffic through it, hope nothing breaks. The firewall lived in its own subnet, traffic was steered using next-hop IPs, and you controlled everything until something stopped flowing, and then you controlled nothing until the packet capture finished. In Virtual WAN, the security model is different. You no longer control the plumbing. You define policy, and Azure takes care of the enforcement path. This has enormous benefits for simplicity and scale, but it also introduces architectural boundaries you need to understand clearly before you can trust it.
 
Built-in Firewall Integration
You can deploy Azure Firewall directly into a Virtual WAN Hub. This is not the same as deploying it into a VNet. In a vWAN Hub, the firewall becomes a service extension, not a routed destination. You do not manage subnets or next-hop IPs. You enable the firewall, and then configure the traffic flow using Routing Intent or custom route tables.
There are two types of traffic you can steer into Azure Firewall in vWAN:
1.	Internet-bound traffic
2.	Private traffic (spoke-to-spoke or site-to-site)
You do this by enabling Routing Intent in the hub and choosing the appropriate options. Once enabled, traffic is automatically redirected to the firewall for inspection. There are no UDRs involved. The platform inserts the inspection point for you.
Where This Goes Right
For most standard designs, this is exactly what you want. Branch users send traffic to Azure, it hits the hub, Routing Intent applies, the traffic passes through Azure Firewall, then continues to its destination. It’s fast, centrally logged, and the rules are defined in the Azure Firewall Policy engine.
•	You can define DNS rules, FQDN filtering, TLS inspection (with Premium tier)
•	You can integrate with Microsoft Defender for Threat Intelligence feeds
•	You can collect logs via Log Analytics and export them to your SIEM
•	You can define application rules per region, per workload type
 
And because the firewall is inside the hub, it scales with the rest of your vWAN topology. You don’t have to manage availability zones or HA pairs. It’s all baked in.
 
Where It Starts to Struggle
There are two areas where the built-in firewall model hits friction. First is routing transparency. Once Routing Intent is enabled, you no longer see or control the exact route map behind the scenes. You can’t modify the system-generated route tables. You can’t inject granular exceptions without disabling Intent. If a specific flow is misrouted or blocked, troubleshooting becomes slower. Second is advanced inspection needs. Azure Firewall has improved massively in recent years, especially with the Premium tier, but there are still gaps:
•	If you need deep packet inspection beyond Layer 7
•	If your inspection rules depend on specific hardware integration
•	If you require intrusion prevention systems (IPS) with certified signatures
•	If you have a team already trained and invested in a third-party platform
 
In these cases, forcing traffic through Azure Firewall just because it's native might not be the right choice. You can insert third-party NVAs into a vWAN hub, but this is not as simple as a tick-box. It requires careful planning, supported appliances, and an understanding of how to route into and out of those NVAs while maintaining transitivity.
 
NVA Insertion in vWAN
Third-party firewall support is available through network virtual appliance partners, but not all vendors are supported, and not all integrations work the same way. Some vendors provide direct support for vWAN, allowing their appliances to be deployed and connected into the hub routing fabric. Others require workarounds or use of custom VNets.
If you do go this route, you will usually:
•	Deploy the NVA into a supported VNet
•	Connect that VNet into the hub
•	Use custom route tables to redirect traffic to the NVA
•	Ensure return traffic flows back through the hub for policy enforcement
 
This removes the simplicity of Routing Intent, but it gives you the inspection capability you need. You are now back in control, but you’ve traded automation for flexibility.
This is fine, as long as the team operating the platform understands the routing model and can troubleshoot symmetric flow issues without escalating everything to the vendor.
 
Designing for Segmentation
Virtual WAN is transitive by default. That means any connection into the hub can potentially talk to any other, unless you design for segmentation. This is a critical point. Security is not just about inspection. It’s also about containment. If a branch is compromised, it must not have unfiltered access to the rest of the network. This is where custom route tables become your primary tool.
For example:
•	Associate each spoke VNet or branch with its own route table
•	Only propagate shared services or required peers into that table
•	Use the firewall to inspect any private traffic that needs to cross boundaries
•	Apply NSGs at the VNet level to protect east-west traffic inside the subnet
 
Do not rely solely on Azure Firewall to be your perimeter. It’s powerful, but it is not a segmentation boundary on its own. Use layered controls. Treat every connection as potentially hostile.
DNS Considerations
DNS often gets overlooked in security design. With vWAN, especially when enabling secure internet breakout, you need to be clear on where DNS resolution occurs. If Azure Firewall is handling outbound flows, you may want to enable DNS Proxy in the Premium tier and forward requests to Azure DNS or your on-prem DNS servers. Otherwise, you risk DNS leakage, failed lookups, or routing loops. Always validate DNS resolution paths when Routing Intent is applied. Check the response source, not just the success code.
 
Virtual WAN gives you a scalable, integrated way to inspect traffic and enforce policy across regions and services. But it is not a catch-all solution. You still need to design for isolation. You still need to plan your inspection points. And you still need to match the tools to the threat model. In the next chapter, we will look at when Azure’s native controls are not enough. What if you need something deeper, or more vendor-specific? That’s where third-party NVAs come in and they deserve their own conversation.

## 8. Third-Party NVAs: When and Why to Break Out of Microsoft’s Backbone

For most customers adopting Virtual WAN, the idea of using only Microsoft-managed services is appealing. Routing is automatic. VPN and ExpressRoute scale without headaches. Azure Firewall slots into the hub. Everything is policy-driven and wrapped in a nice UI. You are no longer responsible for managing complex transit gateways or keeping third-party appliances patched and highly available. But then reality hits. You have an existing security investment in Palo Alto, Fortinet, or Check Point. Your enterprise security team already has predefined rule sets, detection pipelines, and compliance audits built around them. Or maybe Azure Firewall just doesn’t offer the feature depth you need. DNS proxy is not enough. You need TLS decryption with certificate-based re-encryption. You want IPS signatures that are tested against specific threat models. You want reporting that matches your SIEM workflows. 
In these situations, Azure-native controls will not cut it. You need to break out. And that means bringing NVAs back into the design but this time, inside the context of Virtual WAN.
 
Why NVAs Still Matter
Naturally, as a Microsoft employee, you might expect me to be biased towards our native tools. And in most areas, I am. But I genuinely believe Azure Firewall is one of the strongest cloud-native firewalls available today. I would even go as far as to bet good money on it becoming the market leader in enterprise cloud inspection over the next few years. It is simple to deploy, scales well, integrates directly into vWAN, and in most cases, just works. But NVAs still have their place in the design. This isn’t about vendor loyalty. It’s about capability, legacy constraints, and trust models that have been built over years of real-world production.

Here are some situations where third-party NVAs still make sense:
•	You need outbound proxy chaining into legacy inspection stacks that cannot be rebuilt
•	You require deep content inspection based on file type or payload, not just FQDN or IP
•	Your threat detection pipeline only accepts logs in a format generated by a specific vendor
•	Your compliance framework mandates the use of a certified inspection engine that Azure Firewall does not yet support
•	You are mid-way through a data centre exit and cannot switch from your global firewall tooling just yet
 
This isn’t a weakness of Azure. It’s a reflection of how layered, complex, and non-negotiable enterprise security can be. The best design is not always the cleanest one, it’s the one that protects your environment and satisfies every regulatory box you’re expected to tick. This is not about replacing the backbone. It’s about inserting trust and control back into the flow on your terms.
Inserting NVAs into a vWAN Design
This is where many people get confused. You cannot drop an NVA directly into the vWAN Hub. The hub is a managed resource. You cannot deploy virtual machines into it. But there are two supported ways to integrate NVAs:
 
Option 1: Connected VNet Insertion
You deploy the NVA into its own VNet, then connect that VNet to the vWAN hub. You use custom route tables in the hub to steer traffic through the NVA. This is the most common method and gives you full control over routing, inspection, and failover logic.
Example flow:
•	A branch VPN site sends traffic to the vWAN hub
•	The route table for that VPN connection associates with a table that contains a static route
•	That route pushes traffic to the NVA VNet
•	The NVA processes the packet and forwards it back to the hub or to the destination VNet
This works for both inbound and outbound inspection, as long as return flows are symmetric.
You are responsible for:
•	Ensuring the NVA is scaled and highly available
•	Matching the routing logic to each connection type
•	Avoiding blackholing traffic due to asymmetric propagation
•	Maintaining configuration across regions if inspection is required in multiple locations
 
Option 2: NVA Partner Integration (Hub-as-a-Service)
A smaller number of vendors provide direct vWAN integration, sometimes marketed as “NVA-as-a-Service.” These solutions allow you to deploy NVAs into the hub routing fabric with less manual configuration. Azure handles some of the complexity behind the scenes.
In practice, this still relies on a VNet under the hood, but the provisioning is abstracted. Route propagation and high availability may be handled automatically depending on the vendor’s offer.
Do not assume this works out of the box. Check:
•	Whether the vendor supports dynamic route injection (BGP)
•	What regions the service is available in
•	Whether the appliance supports active-active failover
•	How logging and monitoring integrates into your existing tooling
 
Trade-Offs and Design Considerations
Adding third-party NVAs brings back some of the complexity that vWAN is meant to remove. You reintroduce stateful packet inspection outside of Microsoft’s backbone. You need to think about HA, patching, logging, and licence management. You are also responsible for testing failover paths and validating route propagation logic. But for many organisations, that’s a price worth paying. The trade-off is control. If your inspection requirements are non-negotiable, then NVAs become part of your core.
Some best practices from the field:
•	Place NVAs in regionally close VNets to reduce latency
•	Limit the number of flows you inspect to only those that require it
•	Avoid chaining NVAs across multiple hubs unless you have very strong route controls
•	Use route tables to isolate inspection from general transit
•	Test failover explicitly do not assume traffic will reroute cleanly just because the appliance has a second NIC
 
You should also document every NVA path, especially if mixing Routing Intent with custom static routes. It is easy to create loops or lose symmetry when trying to inspect specific flows but not others.
 The Balance of Trust
There is a broader point here. Using Microsoft’s backbone does not mean surrendering control. It means delegating the right parts of the network the scalable, boring parts to the platform. What you keep in your hands is trust. Inspection, logging, rule enforcement, data loss prevention. You decide what needs scrutiny, and where that scrutiny happens. For some, that means Azure Firewall is enough. For others, third-party NVAs are still critical. The goal is not to replace everything with Microsoft. The goal is to design with clarity, enforce with purpose, and know exactly what is flowing where.
In the next chapter, we will look at what happens when all of this goes global. How do you build for multiple regions? When should you deploy multiple hubs? What are the latency and compliance trade-offs? And how do you make sure the design does not collapse under its own complexity?

## 9. Cross-Region Architectures: When Global Hubs Actually Make Sense

There’s a pattern I see all the time with customers rolling out Virtual WAN. They start simple, a single hub, usually in their primary region, anchored around existing workloads or inspection points. For a while, that’s fine. All the spokes plug in. VPN and ExpressRoute land in one place. Internet breakout is centralised. The diagrams look clean, and the teams feel in control.
But then growth happens.
A new branch site opens in Dubai. Another team migrates workloads into Sweden. Someone spins up a dev environment in South Africa. Now traffic from Johannesburg is flowing through the UK hub just to access a local web service. DNS queries from Sweden are being resolved thousands of kilometres away. Users in the Middle East are complaining about login delays. Suddenly, that single hub design the one that worked perfectly for the first six months starts becoming a bottleneck.
This is when organisations realise that global reach doesn't mean global efficiency. Just because the Microsoft backbone can carry traffic from anywhere to anywhere doesn't mean it should.
One (again fictional) customer I worked with had centralised everything into a hub in UK South. That hub hosted their Azure Firewall Premium instance, their central DNS proxy, and a set of shared services including authentication, file storage, and their outbound internet path. Their South African branches connected to this hub over IPsec. Everything SaaS access, domain lookups, file transfers routed up to the UK and back down again. Technically, it worked. But user experience was terrible. DNS lookups were slow. TLS handshakes took too long. Some SaaS applications timed out completely under load. The IT team knew something was off, but assumed the backbone would mask it. It didn’t.
We deployed a second hub in South Africa North. We connected local VNets and branch VPN sites to it directly. DNS was resolved locally using Azure Firewall Premium with DNS proxy enabled. Routing intent was applied in-region. Inter-region routes between hubs were kept narrow, only allowing access to a few shared services in the UK. The impact was immediate. Authentication speed improved. File transfer latency dropped. Internet breakout no longer dragged traffic across the continent. But most importantly, the local IT team now had autonomy they could manage routing and security at the edge, without relying on the UK.
Another use case involved a financial services firm operating across Europe and Africa. They had compliance requirements that forced customer data for South African users to remain within South Africa. But their original hub was in West Europe, and without a second hub, all inspection was centralised. Traffic from customer endpoints in Cape Town was hitting firewalls in Amsterdam before looping back into Azure services hosted in Johannesburg.
The design was ticking all the boxes in the portal successful tunnels, good throughput, clean logs but it was failing at a legal and operational level.
We stood up a second vWAN hub in South Africa North. Inspection was localised. The compliance team signed off the design. We even implemented outbound internet breakout from that region using regionally scoped policies. The backbone was still available for internal DR failover, but by default, all operational flows stayed local. These aren't one-off fixes. This is how scalable vWAN should be used.
When customers rely too heavily on one hub, it usually starts as a shortcut one gateway to manage, one set of route tables, one place to inspect traffic. But the moment you introduce meaningful cross-region traffic, the cost and performance penalty begin to surface. Sometimes it shows up in latency. Other times it shows up in firewall logs or increased egress charges. And occasionally, it shows up in audit findings, where data residency was assumed but never actually enforced.
Deploying a second hub isn't admitting defeat. It's an indication that your architecture has grown and now needs to be regionalised. But it needs to be done correctly. Hub-to-hub connections via the Microsoft backbone don’t behave like standard VNet peerings. You have to design your route tables to control propagation. You need to define which flows should remain local and which should be transitive. And you must always place inspection points close to the user, not the administrator. What I always tell teams is this: start regional, not global. You can always expand. You can always peer later. But collapsing everything into one hub, just because the backbone makes it technically possible, is how you end up with a design that looks good in a demo and breaks the moment real users show up.
In the final chapter, we’ll look at what happens when those mistakes start stacking up what breaks, what slows you down, and what separates a secure, scalable vWAN deployment from one that ends up in permanent firefighting mode.

## 10. Common Pitfalls and Field Lessons: Where vWAN Projects Go Wrong and Why You Should Still Use It
 
Over the years, I’ve been pulled into more broken hub-and-spoke deployments than I care to count. Some of them looked great on paper. Diagrams were clean, every route table was colour-coded, and the inspection flow made sense if you didn’t question the physics of where the users actually sat. Others were clearly patched together in a rush: nested UDRs, asymmetric flows, custom scripts to “fix” BGP, inspection VNets chained into inspection VNets. The result was the same every time. A traditional network design force-fit into a platform that had already moved on.
The problem wasn’t the effort. It was the mindset.
Too many Azure networks still start with on-premises thinking. Layer 3 first. Static segmentation. Hand-built transit paths. Routing controlled hop by hop. That worked in 2005. It even held up into the early cloud migrations. But now we’re in a software-defined era and your network needs to behave like software. This is where Azure Virtual WAN fits in. And in my professional opinion, there is now far more reason to adopt vWAN than to avoid it. I’ve seen the difference it makes. I’ve seen customers waste months trying to scale legacy hub-and-spoke designs across four regions and 50 VNets then switch to vWAN and have it running in days. I’ve seen teams stuck in permanent UDR maintenance mode, only to shift to routing intent and finally get their weekends back. I’ve seen security engineers struggle to trace flows between NVAs and ExpressRoute, then move to a properly segmented vWAN hub and finally understand what’s happening without needing packet captures at every hop.
Are there pitfalls? Absolutely. vWAN is not a free pass. You still need to plan your route tables. You still need to think about IP design. You still need to test failover and make sure your firewalls are in the right place. And if you skip those steps, the same problems you had in classic designs will follow you here.
The most common mistakes I see?
Relying on the Default route table without understanding what it propagates. Assuming Routing Intent will magically inspect everything without checking where your flows are coming from. Stretching a single hub across global users and then wondering why latency spikes every morning. Inserting third-party NVAs without route symmetry. Overlapping address spaces across branches. Misaligned BGP between VPN sites and ER gateways. Each one of these issues is avoidable if you treat vWAN as a new model, not a new wrapper around the old one.
What vWAN gives you if you design for it, is structure. It gives you a backbone that works at global scale, without needing to maintain the scaffolding yourself. It gives you a routing fabric that propagates dynamically, without endless route table copy-paste. It gives you a centralised policy engine for security and segmentation. It gives you hub-to-hub failover, regionally scoped control, and a consistent way to bring together VPN, ER, inspection, and shared services in a single, coherent design.



And most importantly, it forces your network to grow up.
 
You stop thinking about subnets and start thinking about flows. You stop scripting your way out of bad peering logic and start declaring intent. You stop owning every edge device and start focusing on policy, reachability, and trust boundaries. That is how you get scale. That is how you get clarity. That is how you move from firefighting to forward planning.
So here’s where I land after years of fixing broken designs and building new ones:
If you’re starting a new network estate in Azure today, there is no reason not to seriously consider Virtual WAN. And if you're already in Azure and your network is becoming a source of operational drag, vWAN may be your reset button not just in tooling, but in thinking. This series started with a rant about route tables and ended with a blueprint for how to rethink your cloud network. If that’s where you are right now stuck between what used to work and what should work next then vWAN isn’t just a service. It’s your next architecture.
