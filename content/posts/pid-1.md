+++
title = "PID 1"
date = 2024-01-28
+++
I was recently trying to sandbox systemd in a container and finally started reading up on PID 1, the first process in Linux user space.

`systemd` is PID 1 on most Linux distros. And systemd handles
booting, starting up services, cgroup management. All other processes fork off it.

PID 1 is also special (citation needed). If it does not register a signal handler,
Linux will do nothing when you send `SIGTERM` to PID 1.

## Enter containers...
When a container runtime needs to stop a container, it, by default, sends SIGTERM to PID 1.
The first process that runs in a container gets PID 1 in the container's
PID namespace.
Usually, it's the
application (`CMD` or `ENTRYPOINT` or the user-specified command).
If the application registers a SIGTERM handler
and forwards SIGTERM to its child processes, then all should be good.

[Bash](https://www.gnu.org/software/bash/manual/html_node/Signals.html)
and python by default do neither. So if they are PID 1, the container runtime will not send them a SIGTERM. So if I am starting my application with a bash script, SIGTERM will not reach my
application.

[Oracle JVM](https://docs.oracle.com/en/java/javase/17/troubleshoot/handle-signals-and-exceptions.html#GUID-57C048F6-0D4B-43BD-B27C-06A613435360) does register a handler unless the `-Xrs` option is used.

Chances are, when I do `podman stop` on a container, it sends a SIGTERM, waits (while the signal
gets ignored) for 10s, and then send a SIGKILL.

Figuring out what to do with children processes (and threads) when the parent
receives signals is in general a good idea.

## init for containers
podman and docker will start a simple `init` inside the container that handles signal forwarding and process reaping when they are run with
```
podman run --init ...
```
podman uses <https://github.com/openSUSE/catatonit>, and docker uses <https://github.com/krallin/tini> as their init.

See an example of a Dockerfile using init in the [Jupyter Docker stacks image](https://github.com/jupyter/docker-stacks/blob/3692e3a84cd629f47e0a370c99743ade22ed1995/images/docker-stacks-foundation/Dockerfile#L130)
