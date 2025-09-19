# Exposing a Broker running on Kubernetes to a Linux machine using Skupper V2

This example demonstrates how to use Skupper V2 to expose an ActiveMQ Artemis broker instance
running on a Kubernetes cluster, which is not exposed outside of Kubernetes, into a bare metal Linux machine,
through a Virtual Application Network (VAN), using Skupper V2.

![Architecture-Overview](https://private-user-images.githubusercontent.com/7390706/491294367-1ac12995-5e03-46ac-9225-37440cae22b2.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NTgyMzA2NDYsIm5iZiI6MTc1ODIzMDM0NiwicGF0aCI6Ii83MzkwNzA2LzQ5MTI5NDM2Ny0xYWMxMjk5NS01ZTAzLTQ2YWMtOTIyNS0zNzQ0MGNhZTIyYjIucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI1MDkxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNTA5MThUMjExOTA2WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9M2FlZTA1MGNhZGUxOWE1YzQwNDg4NGI1Njg0NzAxYjBhMGZkZGE1OTFhN2M5NzQ5YzBlNmVmNzhlODVkMDg5OSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.eHsUsXbvvi6pM8mC9wFc2MM4WpEIGorNiGMB_Nt3SOY "Diagram")

To achieve that, we need to create a Skupper Link on the Skupper Site running on Kubernetes, under the namespace `broker`, to the Skupper site running on the bare metal Linux machine, through a Podman container.

> ___
> **_Note_**
>
> We will provide our own client and server Certificates, that will be used for linking the two Skupper sites.
> ___

Once the mTLS link between the two Skupper sites is established, the sites are connected and you can expose and consume your resources from any of the linked sites.

To expose the ActiveMQ Artemis instance, we need to create a `Connector` custom resource (CR) (connector.skupper.io/v2alpha1) on the `broker` namespace of the Kubernetes cluster.

The Connector needs to specify the target workload, which can be a Kubernetes `selector` or a static `host`, the respective `port` and a `routingKey` (an arbitrary string). There is a benefit of using a `selector`, as the Skupper V2 Controller will connect/disconnect to/from the respective pods, as they come and go. If you use a host, the Skupper Controller will keep a connector defined to your host, regardless of whether it is responding or not.

Then, on the Bare Metal site, you need to create a `Listener` CR (listener.skupper.io/v2alpha1). The Listener, on a System Site (sites that run outside of Kubernetes), needs a `host` (which is the IP or Hostname to be bound), a `port` and a `routingKey`.

The `routingKey` determines the matching path from a `Listener` to an active `Connector`. So make sure you use the same routingKey on both resources, as it is key to determine where the traffic will be sent to.

As soon as all your resources are defined, your AMQP clients that can reach the assigned host/port, on your bare-metal Linux machine, should work as if the ActiveMQ Artemis instance was running locally.

# Running this example

Here are two different ways you can set up this scenario.

The first one is a manual procedure where you can use the Skupper V2 CLI (`skupper`) and `kubectl` or `oc`.

The second approach uses the upstream `skupper.v2` _Ansible collection_ through a single playbook file, meant to be executed locally on the bare-metal Linux machine.

* [Manual procedure](MANUAL.md)
* [Ansible procedure](ANSIBLE.md)

# Validation your scenario

Once the example is running, you can download and run the following AMQP python clients, to validate everything is working as expected.

Download the following python clients, using:

```bash
wget -O scripts/simple_send.py https://qpid.apache.org/releases/qpid-proton-0.36.0/proton/python/examples/simple_send.py
wget -O scripts/simple_recv.py https://qpid.apache.org/releases/qpid-proton-0.36.0/proton/python/examples/simple_recv.py
```

Then run the sender script, to put some messages into `myqueue`:

```bash
python ./scripts/simple_send.py -a localhost:5672/myqueue
```

Validate that messages are stored in the queue, using:

```bash
./scripts/queue-stat.sh
```

Now you can consume messages, using:

```bash
python ./scripts/simple_recv.py -a localhost:5672/myqueue
```
