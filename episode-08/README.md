# Episode 8 - Kube, Meet Pi

Date: January 27, 2021

Video URL: https://www.youtube.com/watch?v=_r1wN6cD32w

Outline:

  - Kubernetes on Bare Metal
  - Learning Kubernetes with the Raspberry Pi
  - Installing a Kubernetes Distribution
  - Setting up the Raspberry Pi Dramble cluster

## Heavy Metal Kubernetes

For the most part, the production Kubernetes clusters I've worked on were hosted in the cloud, where cloud vendors operate the servers, storage, load balancers, and in most cases, even the Kubernetes control plane itself, that schedules Pods onto Nodes and manages the cluster.

This is very convenient for many applications, but there are situations where Kubernetes is a good fit for your architecture, but you need more control over the cluster, more performance, or more security (e.g. an 'air-gapped' cluster).

And in these cases, you'll want to install Kubernetes on 'bare metal'.

Bare metal is a fancy way of saying 'a particular physical server dedicated to one task.' And in our case, that task is operating as part of a Kubernetes cluster.

There are some things that can be more difficult to manage in a bare-metal environment, including:

  - Cluster networking
  - High-availability for the Kubernetes control plane
  - Storage backends
  - External load balancing and network ingress
  - Certificate management (especially on non-public networks)

## Start with Training Wheels

And in my experience, the best way to learn how to do something well is to start with the basics: in this case, find the simplest, most inexpensive way to build a local bare metal Kubernetes cluster.

My weapon of choice? The humble little $35 Raspberry Pi!

Now, immediately I know you're thinking a Raspberry Pi is woefully underpowered with it's five year old mobile CPU, it's maximum 8 GB of RAM, and it's embarassingly slow microSD storage.

But what the Pi lacks in niceties it makes up for in size, energy consumption, and teaching ability.

### The Raspberry Pi makes for Compact Clusters

The most common refrain I hear from those skeptical of Pi-based clusters is the fact that you could network together a pile of old laptops or cheap corporate thin-client desktops and get more performance, more memory, and faster storage for the same price.

That's true, but once you start physically building a cluster with three or more nodes, you start to realize the efficiency of a single board computer, especially if you go with Power over Ethernet. Not having to take up a couple square feet of real estate on your desk or in a cabinet gives you the freedom to grab your entire cluster with one hand.

Not being tied to four separate power adapters plugged into four separate electrical outlets—plus an extra one for a network switch—means you can quickly move your cluster around as-needed.

In my experience, the simpler and smaller an educational tool is, the more likely it comes out of my closet for some quick testing or learning.

And this is to say nothing of the heat and noise that comes out of a typical Intel-based PC!

### The Raspberry Pi sips energy, and keeps its cool

And that leads me to another advantage of learning on a Pi. Unless you're running heavyweight apps full-tilt 24x7, the Raspberry Pi is pretty easy to keep cool with either heat sinks or a gentle slow fan blowing over the cluster.

Indeed, most of the cluster builds I've seen end up using either passive cooling with an open case, or they have one large-ish fan blowing over all the Pis quietly.

And the entire cluster—including a PoE switch—can operate on less than 50W of power.

At idle, it sips less than 10W of power.

When you're running a Kubernetes cluster, you're not going to take advantage of sleep or hibernation, so running on old laptops where those modes save a ton of power won't help you.

> While raw power usage is lower on a cluster of Pis, _performance per watt_ is likely going to be higher with even a few-generations-old Intel or AMD CPU. But unless you are running intense CPU-hungry applications, or recompiling software on the cluster constantly, that's not necessarily going to result in better power efficiency (over a longer term).

### The Raspberry Pi teaches lessons about scalability

Many developers today are running the latest processors, with dozens of gigabytes of RAM and the fastest NVMe storage. And often you get a budget to buy beefy cloud instances with similar specs and multi-gigabit networking.

But there's an old saying that reminds me of the importance of starting small:

> "Wax on. Wax off."

Just like [Mr. Miyagi in The Karate Kid](https://www.imdb.com/title/tt0087538/characters/nm0001552), I like to teach the lesson that you build a foundation for advanced usage of complex tools like Kubernetes by starting small, focusing on a first step, perfecting your knowledge of that step, then moving on.

So start learning little things, with great constraints. In Kubernetes, you quickly learn lessons that will pay back in spades later:

  - Always add resource limits
  - Monitor CPU and memory usage
  - Optimize your applications so they don't end up thrashing—especially when you end up in situations where your storage, networking, or compute resources start going haywire.

And you might be skeptical: "I have a budget of $50,000/month for multiple 32 vCPU nodes with 64 GB of RAM and fast SSD-backed storage!" I hear you say.

But if you can build an efficient cluster using nothing but small Raspberry Pis, I guarantee your end users will appreciate the incredible speed and scalability you can get when you throw in a faster processor, faster networking, and faster storage, plus the resilience of configuration that deals with outages and slowdowns with aplomb.

### ARM is not all sunshine and roses

That's not to say everything is peachy when building on the Raspberry Pi. It uses a 64-bit ARM processor, and the default Pi OS is actually still _32-bit_, meaning you're starting out a bit crippled, at least early in the 2020s, where Intel and AMD processors still dominate the server landscape.

Through my four years building ARM-based Kubernetes clusters, I have grown to hate—but immediately recognize—the dreaded:

> "Exec format error."

This error would show up when the cluster would pull a container image, try running it, and realize the image doesn't support the ARM architecture of the Pi.

Luckily there are four forces that have come together to make this situation much more rare lately:

  - Raspberry Pi OS is now available as a 64-bit beta, and Canonical publishes a supported 64-bit build for Raspberry Pi as well.
  - Amazon's less-expensive Graviton processors have encouraged more developers to build ARM-compatible images.
  - Apple's M1 processor has proven the ARM architecture has great performance and energy efficiency potential.
  - Tools like Docker's Buildx make building multi-arch images (that work on ARM64, AMD64, and even PowerPC or other more exotic architectures) much easier.

So this problem has become much less prevalent, though there are still many popular and widely-used images that stubbornly refuse to support multiple architectures (*cough* MySQL *cough*).

When you run into these instances, you can either find an alternative image or build one on your own; I've done both of those things in support of my Rasbperry Pi Dramble cluster.

## Installing a Kubernetes Distribution

So now that we've chosen to build a cluster on Raspberry Pis, the next question is _what flavor of Kubernetes_?

In case you weren't aware, there are actually a number of different 'distributions' that maintain compatibility with the Kubernetes API and are managed in a similar way, including:

  - Kubernetes (K8s)
  - [MicroK8s](https://microk8s.io) (maintained by Canonical, the makers of Ubuntu)
  - [K3s](https://k3s.io) (maintained by Rancher)
  - [K0s](https://k0sproject.io) (maintained by Marantis)
  - [OpenShift](https://www.openshift.com) (maintained by Red Hat)

And these are not all, heck _Docker_ even ships their own Kubernetes-in-a-box with every download of Docker Desktop.

But not all of these distributions are suitable for running on the Raspberry Pi. For example, OpenShift requires at least 16 GB of RAM on master nodes, and from my own experience it really wants 32 or more GB per controller.

Other distributions are focused on the 'edge' use case and more minimalist requirements. For example, I ran K3s on a Turing Pi cluster earlier this year, and those nodes only had 1 GB of RAM and 100 megabit networking.

Kubernetes itself is not lightweight, but it runs surprisingly well if you have at least 2 GB of RAM on your Raspberry Pis. It's a little slower than MicroK8s and K3s, but it is nice to learn how to run the 'full' Kubernetes on a homelab built with Pis, so that's what we're going to do here.

### kubeadm

Now, we want to install Kubernetes—but how do we do it?

There are actually a number of ways to do _that_ too, including using pre-built automation like [Kubespray](https://kubespray.io/#/) or [Kops](https://github.com/kubernetes/kops), two officially-sanctioned installers.

But the most direct way to install Kubernetes is using [`kubeadm`](https://github.com/kubernetes/kubeadm), which is describe as:

> a tool built to provide best-practice "fast paths" for creating Kubernetes clusters. It performs the actions necessary to get a minimum viable, secure cluster up and running in a user friendly way.

There's also the option of building a Kubernetes cluster [_the hard way_](https://github.com/kelseyhightower/kubernetes-the-hard-way), but that is about a thousand times deeper than we can go in a 101-level series!

Anyways, I chose to build some light Ansible automation around `kubeadm` to set a Kubernetes controller on one Raspberry Pi, then configure the other three Raspberry Pis in my cluster as nodes that are joined to the controller.

## Setting up the Raspberry Pi Dramble

And if you're building a bare metal cluster, you'll need to put everything together yourself. Instead of going into a cloud dashboard and clicking a plus button a few times, then 'Go', you have to get your hands dirty and plug things together!

So first, let's take a short pause and listen to the [ASMR version of the Raspberry Pi Dramble cluster assembly](https://www.youtube.com/watch?v=M6zGntFBNw4).

Okay, now that that's out of the way, here's a [detailed build video](https://www.youtube.com/watch?v=C-vvccZhT_g) where I show how exactly I put together the Raspberry Pi Dramble, my four-node Kubernetes cluster that's set up with `kubeadm` through an Ansible playbook, with NFS for shared storage, and Traefik for ingress across all four nodes.

If you want to build a cluster just like this, I've documented everything in excruciating detail on the website [pidramble.com](http://www.pidramble.com), and the great thing is it's easy to add as many nodes as you want!

I even have some [Sourcekit PiTray mini's](https://sourcekit.cc/#/) on hand now, so I could add in nodes using the Compute Module in the same footprint as my normal Pi 4 model B's!

A lot of people ask me why I chose to build a cluster with four nodes. Well, back when I had the cluster running on Raspberry Pi 2's, I actually had a 6-node cluster. But back then I had a cheap 8-port gigabit switch. With my new cluster, I used power over ethernet, which means the switch is a lot more expensive! My little four port PoE switch cost more than $50, so I decided to trim the cluster size to 4 nodes instead of doubling my budget for the switch and Pi PoE HATs.

## Going Further

There are a few things I'm _not_ covering in today's episode that you'll probably want to look into more deeply.

First of all, the setup I use in my Dramble cluster uses an Ingress controller running on every node to sort out requests to the Drupal site.

If you're going to run more applications and you want true load balancing at a higher level, like what you'd get with a cloud service, you should look into [MetalLB](https://metallb.universe.tf).

There's a good article on setting it up on Raspberry Pi on Opensource.com: [Install a Kubernetes load balancer on your Raspberry Pi homelab with MetalLB](https://opensource.com/article/20/7/homelab-metallb).

Second, the current configuration doesn't include any monitoring system, even though I set up pretty thorough monitoring—including Pi CPU temperature monitoring—using Prometheus and Grafana. To see how I did that, check out my [Raspberry Pi Cluster Episode 4](https://www.youtube.com/watch?v=IafVCHkJbtI), starting around the 5 minute mark.

Finally, I can't wait for the [Turing Pi 2](https://turingpi.com/v2/) to be released. In my [review of the original Turing Pi](https://www.jeffgeerling.com/blog/2020/raspberry-pi-cluster-episode-6-turing-pi-review), I noted that it is much easier to set up and manage, hardware-wise, than a cluster of Pi 4 model B computers—but that the Compute Module 3+ is a far cry from the Pi 4 generation.

The Turing Pi 2 will support the usually-twice-as-fast Compute Module 4, which means you could have a single board with four nodes and one Ethernet connection for the entire setup, saving even more cabling and power hassle.

### Other Guides

It seems like building a Raspberry Pi Kubernetes cluster is a hot topic, since you'll find hundreds of guides to do it with any Kubernetes distribution if you search it on the web.

The Raspberry Pi Dramble's heritage dates back to 2014, when it was originally a discrete cluster of separate nodes running different applications, but it has evolved as cloud computing itself has evolved, and it'll be interesting to see how I can make it better as the clustering world moves on to newer and better things.
