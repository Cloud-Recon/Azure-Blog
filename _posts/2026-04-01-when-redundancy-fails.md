When-redundancy-fails!
---

Networking and DNS rarely get the glamour in an Azure design. They do not have the sparkle of shiny security tooling, clever automation, or a beautifully named new platform feature. But without them working properly, none of the fancy bells and whistles mean a thing. Your firewalls can be premium, your monitoring can be world-class, and your identity stack can look immaculate on paper, but if traffic cannot get where it needs to go, or names cannot resolve cleanly and consistently, the whole thing starts to wobble very quickly. This chapter is about that reality. Not the reassuring diagrams, not the tidy architecture slides, and not the kind of “high availability” language that sounds good in a meeting and falls apart at 02:13 on a wet Tuesday morning. This is about the parts that actually take you down from a networking and DNS point of view. The awkward failures, the misleading symptoms, and the components that quietly break the estate while something else gets the blame.


## When Redundancy Fails: The Things That Actually Take You Down

There is a moment in most outages where the conversation changes. The initial noise settles, people stop offering guesses, and attention narrows onto a few key screens that suddenly matter more than anything else. It is usually at that point where someone says, with a level of certainty that feels justified at the time, that it must be the network. When multiple systems begin to behave inconsistently, the instinct is to look at the thing that connects them all.

Before going too far down that path, there is a layer worth acknowledging, even if it sits slightly outside my core discipline. Identity has quietly become the real control boundary in Azure. When that layer fails, the impact is immediate and absolute. Token issuance stops, service principals cannot authenticate, managed identities cannot retrieve secrets, and policy enforcement cannot be evaluated. The infrastructure underneath may still be running exactly as designed, but it becomes unusable because nothing can prove what it is or what it is allowed to do. At that point, you are not troubleshooting an outage. You are locked out of the system you need to fix.

Identity failures are obvious.  
DNS failures are not.

## DNS: The Outage That Pretends to Be Something Else

DNS issues in Azure rarely present themselves as DNS issues. They surface as symptoms that appear to belong elsewhere in the stack. Storage latency increases, SQL connections begin to fail during connection establishment, APIs start retrying, and health probes begin to fluctuate between healthy and unhealthy. The behaviour is not binary. It is not up or down. It is inconsistent, and inconsistency is far harder to reason about.

From a networking perspective, everything appears correct. Routes resolve as expected, next hops are valid, and firewall rules permit the traffic that should be allowed. If you trace a successful flow, it completes exactly as designed. The difficulty lies in the fact that not every flow is successful, and the failures do not follow a predictable pattern.

The packet is correct.  
The path is correct.  
The destination is not.  
Or, more precisely, the system responsible for identifying that destination is no longer behaving deterministically.

## A Familiar Architecture and Where It Breaks

A typical enterprise Azure design follows a pattern that feels both logical and safe. A hub and spoke topology is established, Azure Firewall is centralised in the hub, and Private Endpoints are used to secure access to platform services such as storage accounts and SQL databases. DNS is handled by virtual machines in the hub, often acting as forwarders to on-premises or as intermediaries between Azure and external resolution paths.

The data plane works exactly as intended. Traffic leaves the spoke, follows a user-defined route to the firewall, is inspected, and is forwarded to the Private Endpoint. The routing is deterministic, and the firewall enforces policy consistently.

The control plane tells a different story.

A workload attempting to resolve a Private Endpoint name does not query Azure directly. It queries the DNS servers configured on the VNet, typically those hub-based virtual machines. Those machines then evaluate the request, forward it based on conditional rules, and eventually pass it towards the resolver that holds the answer.

What appears as a simple lookup becomes a multi-hop transaction.

And that transaction is fragile.

## The Hidden Cost of Crossing Boundaries

When DNS resolution crosses the hybrid boundary, it inherits every characteristic of that boundary. ExpressRoute introduces BGP convergence behaviour, route propagation delays, and transient packet loss during maintenance or topology changes. VPN adds tunnel renegotiation, rekey intervals, and sensitivity to latency.

At the network layer, these are expected behaviours.  
At the DNS layer, they are disruptive.

A DNS query operates within tight time constraints. If a response is delayed beyond the client’s threshold, the query is retried. Those retries may be sent to different resolvers, or issued concurrently, creating multiple competing resolution paths. If those resolvers are not perfectly aligned, they may return different answers, or fail independently.

Now resolution is not simply delayed.  
It becomes inconsistent.

That inconsistency propagates upward. Applications retry connections. Connection pools begin to saturate. Load balancers interpret failed probes as backend failure. Autoscale reacts to perceived load that is actually the result of retries.

The system begins to behave as though it is under stress.

But the data plane is fine.

The instability exists entirely within the control plane.

## Where Azure Private DNS Changes the Model

Azure Private DNS does not simply improve DNS. It changes where resolution lives and what it depends on.

When a Private Endpoint is created, Azure provisions a network interface within your VNet and assigns it a private IP address. At the same time, when integrated with Private DNS, Azure creates the corresponding DNS record in the appropriate private zone. That record is not manually defined, not synchronised, and not duplicated across systems. It is created by the platform, based on the service itself, and maintained as part of the endpoint lifecycle.

This is where the shift begins.

Every VNet in Azure has access to the Azure-provided resolver at 168.63.129.16. This is not a single server. It is a virtual IP backed by a distributed, platform-managed DNS service that operates within the Azure fabric. It is zone resilient, regionally aware, and designed to resolve both public and private namespaces without requiring any infrastructure from the customer.

When you link a Private DNS Zone to a VNet, you are not just associating a set of records. You are telling the Azure resolver that, for this namespace, it is authoritative within that network boundary.

From that point on, resolution changes fundamentally.

A workload queries a Private Endpoint name. The request is handled locally by the Azure resolver. The resolver retrieves the record from the Private DNS Zone and returns the private IP directly.

No forwarding.  
No recursion across environments.  
No dependency on virtual machines.  
No reliance on hybrid connectivity.

The entire transaction stays inside Azure, inside the region, and inside the same failure domain as the workload.

This has several consequences that are not immediately obvious.

Latency becomes predictable, because the query does not leave the fabric. Resolution times are consistent, not influenced by hybrid links or external resolvers. Availability is tied to a platform service designed for resilience, rather than to virtual machines that must be made highly available through design and maintenance.

More importantly, it unlocks how Azure services are meant to be consumed.

Private Endpoints rely on DNS to function correctly. Without Private DNS, you are forced into manual record management or complex forwarding logic. With it, services such as Storage, SQL, Key Vault, and others integrate seamlessly into your network, resolving privately without exposing public endpoints or requiring translation.

It also enables patterns that are otherwise difficult to implement cleanly. Service isolation becomes straightforward, because resolution can be scoped to specific VNets. Multi-region designs become more predictable, because each region resolves services locally without depending on a centralised DNS infrastructure. Security models become stronger, because access to services is governed by both network path and name resolution within the same boundary.

Even inspection layers benefit.

When Azure Firewall processes traffic, it relies on consistent destination resolution. With Private DNS, the destination IP for a service is stable within the VNet, allowing firewall rules, FQDN tags, and application rules to behave predictably. Without it, the same service may resolve differently depending on where the query originated, introducing variability into security enforcement.

From an architectural perspective, this is the real value.

The control plane and the data plane are now aligned.

The workload resolves a name locally, receives a private IP that is valid within its routing context, and sends traffic along a path that remains entirely within Azure. There are no external dependencies in that transaction.

What was previously a distributed, multi-system process becomes a local, deterministic one.

And that is what removes instability.

## What This Really Comes Down To

DNS in Azure is not limited by capability. It is shaped by where resolution is anchored.

If resolution depends on virtual machines, forwarding chains, and hybrid links, it inherits their characteristics. Latency becomes variable. Availability depends on infrastructure you manage. Failure domains extend beyond the region.

If resolution is anchored within Azure, using Private DNS Zones and the platform resolver, it becomes local, consistent, and aligned with the architecture.

The shift is not about removing control.

It is about placing control in the right place.

Azure already knows where its services are. It already provides a distributed resolver. It already integrates DNS with Private Endpoints.

The more you align with that model, the fewer moving parts you introduce.

And in complex systems, fewer moving parts is often the difference between something that degrades under pressure and something that holds.

DNS does not fail loudly.

It delays.  
It drifts.  
It creates just enough inconsistency to send attention elsewhere.

And by the time it is found, the network has already taken the blame.

## ExpressRoute / VPN Gateway: The Cut That Is Immediate and Absolute

If DNS is the failure that drifts and misleads, hybrid connectivity is the one that does not hide.

When it goes, it goes clean.

There is no slow degradation, no long tail of symptoms that can be misread or misdiagnosed. The impact is immediate and obvious, but what is often missed is that the behaviour you see at the surface is only the final stage of something that has already been unfolding underneath for a few seconds before you notice it.

From the outside, it starts simply.

An application attempts to reach an on-premises dependency and times out. A management session drops. Monitoring stops reporting. It feels like something has been “cut”.

And in a way, it has.

But not in the way most people imagine.

## The Design That Looks Resilient

Most hybrid designs start from a good place.

ExpressRoute is provisioned with provider-side redundancy. Two physical connections, two Microsoft edge locations, multiple BGP sessions. Inside Azure, a hub VNet hosts the ExpressRoute Gateway, with spoke VNets peered into it. Route propagation is enabled so that prefixes learned from on-premises are injected into those spokes.

On paper, it looks solid.

Multiple paths, dynamic routing, centralised control.

The assumption is that redundancy exists end to end.

But the reality is more precise.

All of that external redundancy collapses into a single point inside Azure.

The gateway.

Every BGP session terminates there. Every route learned from on-premises is received there. Every route advertised back is sent from there. Every packet that crosses the boundary between Azure and on-premises passes through that one resource.

It is not just part of the design.

It is the point where the design either holds or fails.

## What Actually Happens When It Drops

What actually happens when the gateway drops is rarely seen directly, but it can be felt almost immediately across the estate.

At first, nothing obvious breaks. There is no dramatic failure message, no clear alert that says the path has gone. Instead, things begin to stall. A connection that should establish in milliseconds hangs for a few seconds longer than expected. A retry succeeds, but the next one does not. It feels like hesitation rather than failure.

Underneath that, the real change has already started.

The BGP sessions between Azure and on-premises have dropped. That part is quiet. There is no visible event from the application’s perspective, but from a routing perspective, it is the moment everything begins to unwind. The routes that Azure learned from on-premises, all those prefixes that made the two environments behave as one network, are no longer valid. They are withdrawn, one after another, from the effective route tables.

At the same time, on-premises routers are doing exactly the same thing in reverse. The prefixes that represented Azure address space are no longer being advertised. They are removed from forwarding tables, leaving gaps where there used to be known paths.

For a brief moment, there is uncertainty.

Some flows continue, not because the path still exists, but because state has not yet expired. Cached routes, established sessions, and timing windows allow a small number of packets to complete their journey. It gives the impression that things are still partially working.

Then that disappears.

Azure workloads attempting to reach on-premises networks perform a route lookup and find nothing. There is no next hop, no valid path, just an absence where a route used to be. The packet is dropped, not because of a deny rule or a firewall decision, but because there is nowhere to send it.

From on-premises, the behaviour mirrors this. A request is made towards Azure, and the router simply has no knowledge of how to reach that destination anymore. Depending on configuration, it may attempt a fallback route, or it may discard the traffic immediately. Either way, the result is the same.

The path is gone.

This transition is not instant, but it is not slow either. There is a short window where routing tables are recalculating, where convergence is taking place, and during that time behaviour can feel unpredictable. One request succeeds, the next fails. A connection appears to establish, then drops without explanation.

It feels like instability.

But what is actually happening is the routing layer going through a very precise and deterministic process of convergence, and that process is far more structured than it appears from the outside.

When the BGP session drops, Azure immediately marks all routes learned through that session as invalid. These are not simply hidden or deprioritised, they are actively removed from the effective route tables associated with each subnet. The next hop that previously pointed to the gateway is no longer considered reachable, so any route depending on it is discarded.

At the same time, on-premises routers detect the loss of the BGP neighbour. Depending on their timers and configuration, they withdraw Azure prefixes and begin recalculating their own forwarding tables.

I saw this play out with a fictional customer I will call Northwind Legal. Their environment relied heavily on ExpressRoute, not just for application traffic, but for identity, document storage, and legacy system integration. When their gateway dropped, the first reports were not “we are down”, but “things feel slow”.

That was the convergence window.

Some requests still followed cached routes. Others failed immediately. Their environment was effectively operating in two states at once. One based on what the network used to know, and one based on what it had already recalculated.

That overlap is where confusion lives.

But it does not last.

As that state clears, the system settles into something far more predictable.

There is no longer a route.

## The Difference Zone Redundancy Makes

This is where the design either absorbs failure or amplifies it.

A zone-redundant gateway changes the failure model from something that triggers convergence into something that the platform can ride through without forcing the network to recalculate itself. The difference is not cosmetic. It is architectural.

With a non-zone gateway, you are anchoring all BGP sessions, all route propagation, and all packet forwarding to a single failure domain. If that domain has a problem, the gateway disappears, and everything that depends on it begins the process you have just seen.

With a zone-redundant gateway, that anchor point no longer exists as a single entity.

The gateway is distributed across availability zones. Its control plane is replicated, its data plane is handled across multiple instances, and its presence from a routing perspective remains stable even if part of the underlying infrastructure is lost.

The important part here is not just availability, it is continuity of state.

BGP sessions do not drop because the endpoint they are peered with is still there. Routes are not withdrawn because the gateway is still advertising them. From the perspective of both Azure and on-premises routers, nothing has changed.

And that is the key difference.

The network does not need to converge.

There is no recalculation, no withdrawal, no temporary inconsistency.

Traffic continues to flow, and the event becomes something that is absorbed rather than exposed.

This is also where design decisions start to matter beyond just selecting a SKU.

If you are relying on ExpressRoute as your primary path, then pairing it with a zone-redundant gateway is not optional, it is fundamental. If you are introducing VPN as a failover path, it needs to be active, not passive, with BGP configured so that failover is automatic rather than manual.

Even then, care needs to be taken.

If both your ExpressRoute and VPN connections terminate on the same non-zone gateway, you have not created redundancy, you have created multiple paths to the same point of failure.

If your failover path exists but is not advertising routes correctly, or is filtered in a way that prevents full reachability, then when failover occurs, you may restore connectivity, but not functionality.

This is where testing becomes critical.

Not a design review, not a diagram validation, but an actual failover test where the primary path is removed and the behaviour is observed. Because the only way to know if your design holds is to see how it behaves when it is forced to.

## What This Really Comes Down To

Hybrid connectivity is often treated as complete once traffic flows and routes appear in tables.

But that is only the starting point.

The real question is not whether connectivity exists, but whether it survives failure without forcing the rest of the system to react.

A non-redundant gateway introduces a failure mode that is both simple and severe. It does not degrade performance, it removes connectivity entirely and forces the network to rebuild itself around that loss.

That rebuild process is where the instability appears.

The fix is not complicated, but it does require intent.

Anchor your design in zone redundancy so that the gateway itself is not tied to a single failure domain. Ensure that any secondary paths, whether ExpressRoute or VPN, are not just present but correctly integrated into the routing model. Use BGP properly, with clear understanding of how routes are preferred and how failover will occur.

Most importantly, reduce dependency on a single path wherever possible.

If identity, DNS, management, and application traffic all rely on the same hybrid link, then the failure of that link becomes a full platform event, not just a connectivity issue. Where possible, design so that Azure can operate independently for periods of time, even if hybrid connectivity is temporarily lost.

Because that is the real difference between resilience and recovery.

Resilience is not how quickly you can rebuild the path.

It is whether you needed to rebuild it at all.

And in hybrid designs, that decision is made long before anything fails.

## Where Traffic First Arrives, and Where It Can Quietly Die

Most Azure architectures are designed from the inside out. Workloads are placed into spokes, security is centralised in a hub, routing is carefully controlled, and everything begins to feel structured and deliberate. It is only later, often once everything is running, that attention shifts to a deceptively simple question. How does traffic actually get in, and what really happens to it once it does?

That entry point is where the outside world meets your platform, and it is far more than just a public IP or a DNS record. It is the beginning of a chain of decisions that must all align perfectly for a single request to succeed. A user types a URL, DNS resolves it to an endpoint, and traffic is directed towards an ingress service such as Azure Front Door or Application Gateway. From that moment, the request is no longer just data on the wire. It becomes something that is evaluated, inspected, redirected, and ultimately trusted or rejected based on a series of rules that span multiple layers.

When this works, it is invisible. Users see an application that responds quickly and consistently. There is no sense of the complexity behind it. But when it fails, the behaviour rarely points directly at the cause. A page loads once and fails the next time. An API responds intermittently. Health probes begin to fluctuate, marking backends as unavailable even though nothing has changed at the workload level. From the outside, it feels like instability. From the inside, it often looks like everything is configured correctly.

The truth usually sits somewhere in between.

The ingress layer is responsible for far more than simply accepting traffic. It terminates TLS, evaluates host headers, applies web application firewall policies, and decides which backend should receive the request. That decision is based not only on configuration, but also on health probes, routing rules, and sometimes even session persistence. A slight misalignment in any of those areas can create behaviour that appears random. A probe that is too aggressive may mark a healthy backend as failed. A routing rule that does not account for a specific path may send traffic somewhere unintended. A WAF rule may block traffic that looks legitimate but matches a pattern it has been told to deny.

None of these are dramatic failures.

But together, they create a system that feels unreliable.

Now place that ingress layer into a real design.

Traffic arrives at Azure Front Door, is distributed globally, and is then forwarded into a regional Application Gateway. That gateway applies additional inspection, evaluates backend health, and forwards traffic into a spoke network where the workload resides. On its own, each of these components is capable and resilient. But the path between them is what matters, and that path is only as strong as its weakest point.

This is where the next layer comes into play, and where the design begins to carry real weight.

Because once traffic enters the platform, it is rarely allowed to flow directly to the workload.

It is inspected.

## When Security Becomes the Thing That Breaks Everything

Modern Azure designs, particularly those aligned with Zero Trust principles, do not treat internal traffic as trusted simply because it is internal. Every flow is evaluated, inspected, and governed. That usually means routing traffic through a central inspection layer, whether that is Azure Firewall or a third-party network virtual appliance. At a design level, this makes complete sense. It provides visibility, control, and a consistent enforcement point for policy.

But the moment you make that decision, something fundamental changes.

You have introduced a dependency that every flow now relies on.

Traffic no longer moves directly from ingress to workload. It moves from ingress, through routing, into a firewall, and only then to its destination. Every request, every response, every connection is now dependent on that inspection layer behaving correctly.

When it does, the system feels controlled and secure.

When it does not, the entire platform can appear to stop.

I have seen this play out in environments that, on paper, looked exceptionally well designed. A global entry point through Front Door, regional load balancing through Application Gateway, and a central firewall controlling all east-west and north-south traffic. It ticked every box. It aligned with best practice. It looked resilient.

Until the firewall had a problem.

Nothing dramatic. No full crash, no obvious outage. Just enough of an issue that it could no longer process traffic at the rate required. From the outside, users could still reach the ingress layer. DNS resolved correctly. TLS sessions were established. Requests were accepted.

And then they disappeared.

From the application team’s perspective, the service was down. From the network team’s perspective, traffic was arriving. From the security logs, nothing was being denied.

The failure was not at the edge.

It was in the path.

This is where the choice of inspection platform begins to matter in a way that goes beyond feature comparison.

Traditional NVAs bring deep inspection capabilities and alignment with existing tooling, but they behave like appliances. They rely on you to build their world around them. You define the scale units, you handle the failover, you ensure symmetry in routing, and you introduce additional components such as load balancers or route manipulation to make sure traffic enters and exits the same way. The moment something changes in that flow, particularly under failure, the appliance can behave in ways that were never part of the original design. State is lost, sessions break, and routing becomes asymmetric in ways that are difficult to predict and even harder to diagnose.

That becomes even more pronounced when Azure Route Server enters the design.

With Route Server, you are no longer statically defining paths with UDRs. You are allowing routes to be learned dynamically through BGP from NVAs. On paper, this is powerful. It removes the operational burden of managing route tables and allows the network to adapt to changes in topology.

But it also means that your firewall or NVA is now participating directly in the control plane.

If that NVA drops a BGP session, advertises incorrect routes, or fails to propagate prefixes consistently, the impact is immediate. Routes change across the estate. Traffic is redirected, or worse, black-holed. The failure is no longer contained to a single device. It is propagated through the routing fabric itself.

This is where Azure Firewall approaches the problem differently.

Not just because I am a Microsoft employee, but because in practice it is one of the most effective firewalls you can place into an Azure architecture when you look at how it interacts with the platform itself.

It is not trying to behave like an appliance in a cloud that does not operate like a data centre.

Azure Firewall does not rely on BGP to insert itself into the path. It does not need to advertise routes, maintain adjacency, or influence the control plane in order to function. Instead, it sits within the routing model that Azure already provides. You steer traffic towards it using UDRs, Virtual WAN routing, or higher-level constructs like Routing Intent, and Azure ensures that traffic consistently flows through it.

When Azure Route Server is introduced, that distinction becomes critical.

With an NVA, Route Server effectively turns that appliance into a routing authority. It becomes responsible for advertising prefixes and influencing how traffic flows across the entire network. If it behaves incorrectly, the impact is widespread. Routes shift, paths change, and failures propagate.

With Azure Firewall, the control plane remains anchored within Azure itself.

Route Server can still exist in the design, particularly where NVAs are required for specific workloads, but Azure Firewall does not depend on it. It continues to operate as a managed inspection layer, with routing decisions enforced by the platform rather than learned dynamically from a device.

That separation is what keeps failures contained.

Because the moment your inspection layer starts influencing your routing layer, the blast radius of any issue increases dramatically.

## What This Really Comes Down To

Ingress and inspection are not separate concerns.

They are part of the same journey that every packet takes from the outside world to your application and back again. Treating them independently creates gaps, and those gaps are where outages live.

A resilient ingress layer on its own does not make a platform resilient. A powerful firewall on its own does not make a platform secure. It is the alignment between them, and the way they interact with the control plane, that determines how the system behaves when something goes wrong.

The fix is not to add more components.

It is to design with awareness of where control actually sits.

If your inspection layer is also influencing routing through BGP, then you need to understand how that behaviour propagates under failure. If your ingress layer is resilient but forwards into a path that is not, then you have simply moved the failure point further inside the network.

The strongest designs are the ones where the control plane remains stable even when individual components fail.

That means using platform-native services where they make sense, reducing dependency on appliance-driven routing where possible, and ensuring that traffic paths remain predictable even when parts of the system are removed.

It also means making deliberate decisions about how traffic is forced through inspection. If everything depends on a single path, then that path must be able to survive failure without requiring the network to rebuild itself around it. That is where zone redundancy, distributed services, and platform integration stop being nice-to-haves and become fundamental design choices.

And it means testing those assumptions.

Not by reviewing diagrams, but by forcing the design into failure and watching how it behaves. What happens when an availability zone drops. What happens when a firewall instance becomes unavailable. What happens when routing changes under load.

Because that is where the real architecture reveals itself.

Not when everything is working.

But when something is not, and the system either carries on without you noticing, or stops in a place you never expected.

## Routing Control Plane: When Everything Is Working, but Nothing Lands Where It Should

There is a point in most designs where confidence quietly settles in.

Everything that should fail loudly has been addressed. Identity is stable and trusted, DNS is predictable, hybrid connectivity is resilient, ingress is doing exactly what it should, and inspection is in place and behaving. The diagram looks clean. The flows make sense. If you were to walk someone through it, there would be no obvious weak point to point at.

And yet, this is often where the most difficult outages begin.

Not because something has stopped working, but because the network is still working, just not in the way you believe it is.

What makes this particularly difficult to recognise is that nothing immediately looks wrong. Traffic leaves the source as expected. Routes resolve correctly. Firewalls allow the flows you would expect them to allow. If you trace a single successful request, it behaves exactly as designed, moving cleanly through the environment and returning without issue. There is no obvious break, no clean failure, nothing that gives you a clear place to start.

But the pattern is never consistent.

One request works, the next does not. A connection establishes, then stalls. A backend appears healthy, but probes begin to fail intermittently. It feels like something is blocking traffic, but nothing in the logs supports that. There are no denies, no drops, no clear indicators that anything has been stopped.

What you are seeing is not a failure of the data plane.

It is a shift in the control plane.

That shift only really appears once a network has moved beyond static routing. In simpler designs, routing is predictable because it is defined. User Defined Routes point traffic towards specific next hops, and behaviour remains stable until someone changes something. It may not scale particularly well, but it is understandable. You can look at a route table and know exactly what will happen to a packet.

The moment you introduce Azure Route Server, Virtual WAN, or BGP-enabled designs, that certainty changes. Routes are no longer just defined, they are learned. They are advertised, withdrawn, recalculated, and propagated across the network based on the state of the components participating in that process. The network stops being something you configure and becomes something that reacts.

That is where the power comes from.

And that is also where the risk begins.

I remember working through this with a fictional customer, a financial services organisation that had reached the point where static routing was no longer sustainable. Their estate had grown to the point where managing route tables had become an operational burden, so they did what many would do. They introduced Azure Route Server into the hub, integrated a pair of NVAs using BGP, and allowed the network to handle its own route propagation.

For a long time, it worked exactly as intended.

Routes appeared where they needed to be without intervention. New workloads were reachable without additional configuration. Failover between paths was handled through BGP behaviour rather than manual changes. The network felt adaptive, responsive, and far easier to manage than what had come before.

It is only later, usually during something routine, that the nature of that design becomes clear.

In this case, it was a change window. Nothing unusual, nothing high risk. One of the NVAs experienced a brief instability during the change. Not a full failure, not even something that would normally cause concern, just enough for its BGP session with Route Server to drop and then come back.

That moment was all it took.

From the perspective of Azure Route Server, that change in state was legitimate. The prefixes that had been advertised by that NVA were no longer present, so they were withdrawn. The routing tables across the environment were updated to reflect that. Paths that had existed seconds earlier were removed because, from the control plane’s point of view, they were no longer valid.

Then the session came back.

The prefixes were re-advertised. The network recalculated again. New paths were introduced, and traffic began to follow them.

All of this happened exactly as it should have.

And that was the problem.

Because while the control plane was behaving perfectly, the data plane was caught in the middle of that change.

Traffic that had been flowing along a known path was now being redirected while sessions were still active. Some requests completed because the original path still existed in memory long enough for them to finish. Others failed because the route had already been removed. Return traffic began to follow different paths than the forward traffic, and that is where the behaviour started to break in ways that were not immediately obvious.

The inspection layer was no longer seeing full conversations.

It was seeing fragments.

Packets arriving without their corresponding return path. Responses taking a different route back than the one they came in on. From the firewall’s perspective, these were not valid sessions. They were incomplete, and incomplete sessions are dropped.

From the outside, none of this made sense.

Applications were not fully down, but they were not working either. Some users could connect without issue, others could not connect at all. Requests would succeed and fail in a pattern that felt random. There was no single point you could isolate, no component that had clearly stopped responding.

Because nothing had stopped.

The network had simply changed, and in doing so it had broken the alignment between the forward path and the return path that everything else depended on.

This is the point where the difference between Azure Firewall and traditional NVAs becomes more than just a technical preference.

In a design where NVAs are integrated with Azure Route Server, they are not just inspecting traffic. They are influencing routing decisions. They are part of the control plane, advertising prefixes and shaping how traffic flows through the network. That gives a great deal of flexibility, but it also means that any instability in those devices directly affects the routing behaviour of the entire estate.

They are no longer just in the path.

They are defining it.

Azure Firewall does not operate in that way.

Not just because of where it comes from, but because it is designed to sit within Azure rather than reshape it. It does not participate in BGP, it does not advertise routes, and it does not influence how the control plane recalculates paths. Instead, it relies on Azure’s routing constructs to bring traffic to it, and it processes that traffic consistently regardless of how those routes were learned.

When Azure Route Server is present, that distinction becomes critical.

The control plane may still change, routes may still be learned and withdrawn, but Azure Firewall itself is not introducing those changes. It is not injecting instability into the routing fabric. It remains a consistent point of inspection within a system that may otherwise be adapting in real time.

That separation is what limits the blast radius.

Because when the control plane shifts, you want the components that depend on it to remain stable, not amplify the change.

The real issue in these scenarios is not that routing is dynamic.

It is that routing becomes unpredictable when too many components are allowed to influence it without clear boundaries.

Dynamic routing is powerful, and in many environments it is exactly what is needed. But it has to be introduced with an understanding that every participant in that control plane carries weight. Every advertised route, every withdrawn prefix, every recalculation has an immediate effect on how traffic moves.

And once that movement changes, everything built on top of it must be able to cope.

If it cannot, then a momentary shift in the control plane becomes a visible failure in the application.

The strongest designs are the ones where that does not happen. Where the control plane can change without breaking the data plane. Where routing decisions are anchored in a way that prevents drift from becoming instability. Where inspection remains consistent even when paths are not.

Because when the control plane begins to drift, the network does not stop.

It continues.

But it no longer behaves in a way you can predict.

And that is when the system feels most broken, not because it has failed, but because it is still running, just not in the way you designed it to.

If there is a common thread running through all of this, it is that failure rarely arrives wearing a name badge. It does not walk in and politely announce that DNS is drifting, that the gateway has gone, or that the routing control plane has started making creative decisions on your behalf. It usually arrives as confusion, partial symptoms, and that sinking feeling that something is off even though all the dashboards are trying their best to look calm. The lesson is not to fear complexity for the sake of it, but to be honest about where it lives and what happens when it moves. Good design is not about making a diagram look clever. It is about making sure that when something breaks at 02:13 on a wet Tuesday morning, the estate bends instead of folds, and you are not left staring at a screen muttering words that would get you barred from your own change advisory board.
```
