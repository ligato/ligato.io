+++
title = "CNFs with a Dose of Ligato and FD.io/VPP (Updated)"
author = "Chris Metz"
date = "02 May 2019"
layout = "blogdata"
+++ 

Cloud Native Network Functions (CNFs) are containerized instances of a classic physical or virtual network functions (VNF). A fast dataplane incorporating current and future VNFs functions is tablestakes. Exposing the CNF dataplane to applications and plugins requires a control plane agent with APIs.
 <!--more--> 

# Introduction 

Many Public, private and hybrid cloud operators have arrived at the on-ramp to “interstate cloud native”. Some have even blown through the metering lights and are accelerating, ready to leave those still picking up speed in the dust. Cloud native offers heaps of advantages: deployment velocity, horizontal scalability, resiliency, faster devops, optimized resource utilization and really cool project names with neat looking icons. 

Carving up application monoliths into smaller chunks of code called microservices; packaging up microservices into containers; housing said containers in PODs running on hosts (virtual or physical); using Kubernetes to orchestrate and schedule POD placements commanded by applications across standard Kubernetes (K8S) APIs - is THE way – the cloud native way.

Applications are king in cloud native land. But, you don’t hear a whole lot about networking in this space. Why is that so? A few thoughts. 
 
* __App developers don’t care about networking__ (sadly, treated as a gut course for many comp sci majors). They care about apps, how quickly they can be hacked, scrubbed of bugs and deployed to the cloud. 
  
* __Networking has its own language spoken only by fellow network geeks__ (full disclosure: I am a network geek). They get IPv4, ACLs, /32s, BGP, VXLAN tunnels, IPv6, Segment Routing, MPLS GRE, Service Chains and all that stuff (please consult your favorite network lexicon for more). What does all of that mean to the cloud native apps developer!? Nothing. Zero. Zip.

* __Speed mismatch between classic and cloud native network deployment and operations.__ The former is big monolithic “boxes” (physical or virtual), installed in one location in the network by smart network people, occasionally visited by smart network people for upgrades/troubleshooting (this is mutable infrastructure! See below) and generally left alone. Forever. The latter is bang-bang: Locate/execute configs. PODs up! Here are the PODs you can talk to! Here are the services. Speak. PODs Out! All automated. All game time decisions. This is an absurdly simple comparison but you get the picture.


To be fair, Kubernetes (k8s) does define the [Container Network Interface (CNI)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) – an API for network plugins that “bootstrap” and manage inter-POD communications. In other words, this provides network connectivity and makes the network transparent to PODs. That’s all good, but it also means PODs can’t really use other network services, such as security or QoS. Furthermore, k8s defines services and policies, which also have to be mapped into network configurations but there is no formal API to achieve this. Therefore it’s up to the network plugin implementers to figure out the mapping. 

Fortunately, the cloud native community is beginning to realize that networking is really important. Spurred on by innovations such as [FD](https://fd.io/).io (user space software data plane) and [BPF](https://opensource.com/article/17/9/intro-ebpf) (enhanced Berkeley Packet Filter), throughput is now seen as critical. The need for cloud native network functions (CNF) has been identified. CNF-enabling open source projects are active and code is up there in github repos. Vendors (legacy “big box” and new startups) are implementing and shipping CNFs. 

Cloud native seems ALL IN on networking.


# What is a CNF?

A virtual network function (VNF), is a software implementation of a network function traditionally performed on a physical device. Examples include IPv4/v6 routing, VPN gateway, L2 bridge/switch, firewall, NATs and tunnel encap/decap. 

Let’s lay out a few character traits of a VNF personality:

* __Deal mostly with L2 frames (Ethernet) and L3 (IP) packets.__ In some cases like a firewall or NAT, they could jump up to L4 to include TCP or UDP port numbers. Tuple bloat can occur if you drop down to L2 and include MAC, ethertype, VLAN, etc.). 

* __Typically come with more than one interface.__ Packet processing is sandwiched between input and output interfaces. Proof point is the extinction of one-armed routers.

* __Fast data plane.__ No dawdling in packet processing. Keep those packets moving, else users and apps will not be happy.

You get the idea. These are network functions, virtualized, with features and functions inherited from their physical antecedents. VNFs can be connected or combined together as building blocks to offer a full-scale networking communication service. 

VNFs are not nor can they be classified as web server entities with a full-blow TCP/UDP network stack. At this time. A VNF network stack is composed of network functions with the aforementioned traits. This is an important distinction when discussing a home for VNF stacks below. Please do not answer the door when “Techno Network Terminology Parsing Police” (TNTPP) come knocking. Let's try to cover all the bases with this: 

__VNF Stack is defined as the layer-specific packet processing functions (i.e. forwarding, policies, stats, etc.) required to realize the desired network function in virtual environments.__  

So, what exactly is a cloud native network function (CNF)? We could go with “ __[.. a VNF built and deployed using cloud native technologies ..]( https://github.com/State-of-the-Edge/glossary/blob/master/edge-glossary.md#virtualized-network-function-vnf)__”. Not bad but need a bit more to get the feel.  The first portion in this definition, “a VNF”, translates to a containerized instance of a VNF.



## User-in-Space

Let’s proceed to the “built” in this definition by cracking open a physical or virtual host with its own host operating system (i.e. Linux) allegedly supporting one or more CNFs. Inside you might find one or more of these CNFs (and/or application containers too) living inside Kubernetes pods. To simplify the discussion, let’s stay with containers and CNFs and assume they exist in pods. A VNF stack is now a CNF stack.

The CNFs could run a portion of their functions in what is referred to as user space. You will also find the host OS kernel providing various utilities including a TCP/IP network stack. All of the containers on this host rely on this single network stack. All packets in and out of user space must pass through the kernel. In this scenario, a CNF consists of one piece of function in user space and the other piece, the network stack, residing in the kernel. We can capture this notion with a figure below.
<br />

{{< figure src="/images/ligato/user-space-kernel.png" class="image-center figcaption" caption="User Space and Kernel Networking" >}}

 <br />
  
  

Does the CNF stack belong in the kernel? Must new CNFs rely on the kernel network functions? 

Maybe not. We could decide to park the entire CNF in user space. Now you can have each and every CNF run its own network stack including the data plane, each packaged with its own control and management plane instances. There is no need for the CNF to rely on the kernel network stack. 

How does a CNF residing in user space help?

* __Elevated performance throughput__ as now the CNF can talk directly to the physical network interface card (NIC) with the goal of keeping pace with the speed developments happening in this space.

* __Accelerated network innovation__ development and roll-out. CNF developers can go to town and paint their innovations on a large user space canvas. It is THE opportunity to mandate all CNFs run in user space. It just make sense.

* __Fast recovery__. If anything happens to the user space CNF stack (e.g. upgrade, crash, restart, etc.), it DOES NOT bring whole node down. You just start it up quickly and continue on with your work. More on the quickly part below. Hint: it is pretty cool.

* __[12-Factor App Methodology](https://12factor.net/) applied to CNFs.__The nexus of innovation lives in the cloud. Adopting all or some of the principles in the design, development and deployment of CNFs is a no brainer. There is no downside. Just do it. 

<br />
  
{{< figure src="/images/ligato/user-space-CNF.png" class="image-center figcaption" caption="CNFs in User Space" >}}


Take a look at these two figures and see if they resonate. The CNFs avoid the kernel altogether. Note that other pods (not shown) using the kernel can continue to operate business as usual. Peaceful coexistence. And being honest - reality into perpetuity because there will always be pods that use the host's kernel networking stack.


## Don't Touch That! I'll Make You a New One.

The “deployed” can be realized through the use of the __[immutable infrastructure](https://www.digitalocean.com/community/tutorials/what-is-immutable-infrastructure)__ pattern. This means _“thou shall NOT TOUCH a software image in production”_. EVER. If changes are needed (e.g. bug fix, config mods, new feature upgrades), the image is destroyed, a new image created, and then cycled into production. Create – Deploy – Destroy, Rinse and Repeat. Clean, consistent, predictable and simple. 

Another take on the notion of immutable infrastructure says we can put place workloads (i.e. CNFs) anywhere in the network with confidence because all CNFs reside in user space. The kernel infrastructure never changes and thus is immutable. 

Think about this in the context of a mission-critical IT CNF, perhaps an advanced firewall. And suppose a new patch needs to be applied to this CNF image, STAT! If the zip code of the new CNF image is “userland”, then we can feel confident nothing will break. The kernel infrastructure is untouched – Immutable. __Don’t stress – immutable is best.__


## What else does a CNF need?

There are several not-to-be-left-unsaid properties we would like to see in CNFs moving forward.

* __Interface support__. CNFs must be interface-flexible meaning support for all of the different flavors of cloud native virtualized network interfaces (e.g. veth, memif). They should also be able to handle multiple interfaces concurrently, instead of just one as has been the case to date.

* __Observability__. CNFs must/must/must generate a robust suite of telemetry, logs, tracing, stats as a data feed into cloud native management solutions such as [Prometheus](https://prometheus.io/). This cannot be overlooked.

* __Programmability__. Cloud native deployment velocity dictates rapid CNF placement and turn up with the desired functions. Recall the need to map k8s services and policies to the network components. Or program a common software data plane to provide a new CNF function. Agents and APIs can make this happen. 

There are a few others such as CNF meshes and service chains but these are topics for future installments.

So there you have it. CNFs are containerized VNFs built and deployed with cloud native technologies. CNFs run in user space for maximum performance, isolation and “innovation-abiliy”. CNFs accommodate cloud native virtual network interfaces, incorporate observability functions and and interact with agents and APIs to manage CNF functions. CNFs should exploit immutable infrastructure deployment best practices with impunity. And goes without saying, all happening with the expressed written consent of the major league cloud native network community.

On to the discussion of the CNF data plane and programmability.


# Slow Down! You are going too fast!

[FD.io]( https://fd.io/) is touted as the “Universal Dataplane”. Cloud native networks will need every ounce of capacity, scale, functions and services that a software data plane can offer. Even then that might not be enough but fear not, the “innovationists” never stop working on something bigger/faster/better.  If this plot sounds familiar, please reference the history of the Internet.  

FD.io/VPP is an open source project sponsored by the Linux Foundation. It enjoys broad industry support and VPP itself has been implemented and deployed in products for years. For expediency purposes lets quickly list some its features that make FD.io attractive as a software data plane for CNFs:

* High Performance on the order of 1Tb throughput with a million routes on industry standard Intel and ARM processors. 

* Resource efficient.

* Developer-friendly and written in C

* Runs in user space 

* [Tons of L2, L3 and L4 features]( https://wiki.fd.io/view/VPP/Features) with too many to enumerate.

* Tons of exportable telemetry, logs, stats, etc.

* Plugin-ability. Customized functions to be easily hacked up and implemented into the VPP packet processing system

* High-speed shared memory interface [Memif]( https://www.youtube.com/watch?v=6aVr32WgY0Q) for inter-VPP communications

* Operates in bare metal, VM and container platforms.

* Programmability. This is necessary. Can't live without it. Tablestakes. (see below).

We can be pretty confident that with the FD.io/VPP dataplane, CNF performance will be there as needed. Feature extensibility and plugin-ability offers future protection, developers will be down with it and operations folks will love it. 

For more information, you can tune into the [FD.io YouTube channel]( https://www.youtube.com/channel/UCIJ2OP6_i1npoHM39kxvwyg), pull down a very good whitepaper from [here]( https://fd.io/wp-content/uploads/sites/34/2017/07/FDioVPPwhitepaperJuly2017.pdf), puruse a [deck presented at ONS Europe 2018](https://events.linuxfoundation.org/wp-content/uploads/2017/12/High-Performance-Cloud-Native-Networking-K8s-Unleashing-FD.io-Maciek-Konstantynowicz-Giles-Heron-Cisco.pdf)) or open up your browser and exercise Google. Lots out there. 


# Someone Tell It What to Do

We have our CNF data plane. But a data plane alone, no matter how fast, doth not a network make. We need a programmable data plane. We need the ability for CNFs to communicate with other control and management microservices in the cloud native network. How can we do that?

[Ligato]( https://ligato.io/) is the answer.  The bumper sticker version of Ligato follows:

_Ligato is an open source framework for building applications to control and manage Cloud Native Network Functions (CNF). It comes with a VPP Agent serving as the control plane for FD.io/VPP-enabled CNFs. It also provides a whole raft of different plugin building blocks developers can use to create their own cloud native containerized masterpieces._
<br />

{{< figure src="/images/ligato/ligato-framework-arch.svg" class="image-center figcaption"caption="Ligato Framework" >}}
<br />
For those looking into Ligato for the first time, it might not be 100% clear about what it is. How does it relate to CNFs? What are plugins and how are they used? Do I need VPP? Let’s clear that up right now using the picture above as a reference.

For starters it is laid out as a classic stack; apps on top and packet forwarding ala VPP or the linux kernel on the bottom. The Ligato stuff is inside the green box in the middle and that is pretty much all you need to know. But if you are looking for more, here you go:

* Developers use Ligato to _build CNF and non-CNF applications_. It is largely comprised of a set of plugins (<sigh> … an overloaded term for sure; analogy is React components or Go packages but I digress) where each plugin performs a specific function. They come with lifecycle management (i.e. initialization and graceful shutdown). It is the combination of plugins and lifecycle management enabled by the Infra functions that define roughly what the CNF can do. Note from the figure above that the VPP Agent includes plugins so the previous statement remains true.

* Comes OOTB (out-of-the-box) with plugins galore to choose from. [__Here__](https://docs.ligato.io/en/latest/plugins/plugin-overview/) you will find an abundance of plugins supporting different functions including those for APIs, datastores, messaging and logging. One more point about plugins and that is the developer is not required to use all plugins – merely those specific to the microservice(s) they wish to build.

* Comes with tutorials, docs and examples to get you going. Here is the requisite [__"Hello World"__](https://docs.ligato.io/en/latest/tutorials/01_hello-world/) example. Makes it easier for newcomers and veterans alike.

* Developers can create their own application-specific plugins. For example, if a new microservice requires kafka messages to be consumed and written to a database, the developer could use the [__Kafka plugin__](https://docs.ligato.io/en/latest/plugins/infra-plugins/#kafka-plugin) to ingest the messages and an application plug-in to write the messages to a database. 

Ligato is plugin-rich so developers are free to use whatever combination they like. What's next?


## VPP Agent 

Again referencing the figure above, it is positioned in the Ligato framework as a set of [VPP-specific plugins](https://docs.ligato.io/en/latest/plugins/vpp-plugins/) inheriting the infra lifecycle management. The use of one or more of these plugins (along with any other application plugins) exposes FD.io/VPP functionality (e.g. forwarding, packet service treatment, stats generation, etc.) to other Ligato plugins and/or external control/management applications.
	
<br />
<br />

{{< figure src="/images/ligato/ligato-vpp-agent.png" class="image-center figcaption"caption="Ligato VPP Agent" >}}
<br />
Let’s examine how the VPP agent fits into a CNF architecture with an FD.io/VPP dataplane. In the figure directly above we show a container (in user space of course) with an FD.io/VPP dataplane and the VPP agent. The container packaging is lightweight and eliminates version mismatches. We described earlier the notion of user space CNFs so DPDK drivers are used here to speak directly to the NIC and bypass the kernel.  

The VPP agent plugins provide northbound APIs for configuring and managing default VPP functions such interface configuration, L2 bridge domains, L3 IP routing/VRFs, L4 namespaces, ACLs and segment routing and so on. Other plugins extending FD.io/VPP control and management API access can be incorporated into the agent. 

The [GoVPP](https://wiki.fd.io/view/GoVPP) plugin is mandatory for the VPP agent. This is essentially a Go-based toolset allowing plugin(s) to communicate with the VPP dataplane process(es). Note the plural in both plugins and VPP processes.

At the risk of "stack layer jumping", brief mention should be made here of the KVScheduler (KV stands for Key Value) plugin. Technically it is not a VPP agent plugin but communicates directly with VPP agent plugins to ensure that consistent and correct configuration is installed in the VPP dataplane. Its importance cannot be overstated in the architecture, development, management and operation of CNF (and non-CNF) solutions. 

A problem arose when the number of configuration items (i.e. VPP plugins) scaled upwards due to the necessary turn up of features in the VPP dataplane. The configurator plugins built each configuration item (e.g. interface, route, bridge domain, etc.) from scratch leading to lots of code duplication. Configurators used notifications to talk with each other and react to changes asynchronously. At some point it became untenable for the configurators to sort through and sequence in correct order all of the different configuration items. It was like a bunch of people standing around, shouting different directions to a restaurant. Hunger pains continue. Chaos ensues. Not happy.     
 
KVScheduler to the rescue! It uplevels configuration item dependencies and ordering into a graph abstraction. The configuration items are nodes, the relationship between them, links. The graph is used to build configuration state to be programmed into the VPP dataplane of the CNF.

Much detail omitted but you can find out more about the __[KVScheduler](https://docs.ligato.io/en/latest/developer-guide/kvscheduler/) here__.  

We are fast approaching the well-known attention span threshold and I need to go to the store.

  
# Almost Done

What have we got so far?


* __Decent definition of a CNF. VNF workloads living in containers.__ Maybe not the most complete but good enough for now.

* __Cloud Native-ized, open source software-based data plane bulging with features.__ FD.io cuts it.

* __Compendium of functional plugins backed up by open source libraries, documentation, examples, lifecycle management and a credo of “only use what is needed”.__ Environment is friendly for creating and applying customized plugins. Ligato!

* __Agent for control and management of VPP-based data plane.__ The VPP agent offered up with Ligato.


Where do we go from here? The answer is …

#### The Programmable VPP vSwitch!

_By bolting the control plane VPP agent and VPP data plane together, a programmable VPP vSwitch is created – a CNF really - a basic CNF building block - the foundation for high-performance container network solutions._

<br />
<br />

{{< figure src="/images/ligato/ligato-cnf-evolve.svg" class="image-center figcaption"caption="Programmable VPP vSwitch" >}}

<br />
Absent the requisite cloud native management and control plane pieces, it is perfectly legal to picture this as a CNF. Modern networks (physical or virtual) require that switches are outfitted with a variety of interfaces accommodating different devices, hosts, capacities, services and network configurations. Same applies here but now we are dealing with the new (memifs) and the legacy (e.g.veth). 

Switches provide high-throughput L2/L3 forwarding. And not just for slinging packets - also present are policies, QoS, stats export and so on. These cannot, I repeat cannot impact performance. This is the case here with the VPP vSwitch. And it is programmable. And fast. Very.

We have the programmable VPP vSwitch. Throw in a little user space data plane functionality, immutability deployment practices, Kubernetes lifecycle management, and the tools to build customized microservice and we have the pieces in place to craft up cool CNF-flavored solutions.

Where do we go from here? I would hazard a guess – more cool CNF solutions. 

Stay tuned.



