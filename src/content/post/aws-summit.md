---
title: "Using Containers without understanding Linux"
description: "After AWS Summit, I realized I could use containers without really understanding what made them work."
publishDate: "2026-06-10"
tags: ["linux", "containers"]
---

I thought I understood containers until someone asked me about cgroups.

I knew the Docker-level story: containers solved my issues with packaging app code and dependencies, too often I would have conflicting libraries in Python. By using containers it didn't just "work on my machine". I was able to do manual deployments with a simple `git pull` and `docker compose up -d`. They are a self-contained unit.

However, upon being asked if I knew about cgroups, it allowed me to realise I knew the tool, not the mechanism. Because of this I had to go back and learn the Linux layer underneath.

What I learned was simple: containers are not magic. They are Linux processes with isolation and resource limits applied around them.

Containers are built from Linux kernel features, and Docker just makes these features easy to use. **namespaces** make a process think it's alone on the system, giving isolation.
```
vscode /workspaces/first-devpod
➜  pstree
sh───sleep
```
**cgroups** or control groups limit what resources a container can use. Without cgroups, one container could starve others of resources.

**union FS** images are built in layers. Union file systems (like OverlayFS) stack these layers. This is great because the shared layers are cached, many containers share base layers and only differences are stored per container.

Containers, compared to VMs, have less overhead, start faster and use fewer resources as they share the host kernel. This is a trade-off with isolation.

By learning the underlying Linux fundamentals of Containers and VMs I've grown to see the reason why Docker has come to be as it makes using the Linux kernel features much easier.
