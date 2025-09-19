# Manual procedure

## Install the Skupper V2 controller on your Kubernetes cluster

To install the Skupper V2 controller (upstream), you just need to apply a YAML to your Kubernetes cluster.

To do that, run:

```bash
kubectl apply -f https://github.com/skupperproject/skupper/releases/download/2.1.1/skupper-cluster-scope.yaml
```

The controller will be installed into the `skupper` namespace.

## Install the Skupper V2 CLI

Run the following script:

```bash
curl https://skupper.io/install.sh | sh
```

## Installing the ActiveMQ Artemis Operator

This example, assumes you have the OLM (Operator Lifecycle Manager) running in your cluster. If you don't, you can run the following command against your local Kubernetes cluster. If you have OLM running, skip the following step.

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.34.0/install.sh | bash -s v0.34.0
```

Once OLM is running, you need to install the operator, using:

```bash
kubectl create -f https://operatorhub.io/install/activemq-artemis-operator.yaml
```

## Creating an ActiveMQ Artemis Instance

This example, expects the ActiveMQ Artemis instance as well as the Skupper site on Kubernetes to run
in the `broker` namespace. So first, create the `broker` namespace, then create the ActiveMQ Artemis instance.

```bash
kubectl create namespace broker
```

Now create the ActiveMQ Artemis instance, using a pre-defined YAML:

```bash
kubectl -n broker apply -f ./ansible/ocp/activemq-artemis.yaml
```

This instance is configured to expose the AMQP port `5672` and to define a queue named `myqueue`.

Wait for the Artemis pod to be ready, running:

```bash
kubectl -n broker wait --for=condition=Ready=true pod -l ActiveMQArtemis=artemis-broker
```

After the pod's condition is met, you can run the following command, to make sure that the queue `myqueue` has been defined:

```bash
./scripts/queue-stat.sh
```

## Running your Podman site

### Creating the resources on your Podman site

In V2, Skupper allows you to have multiple namespaces in a System site (outside of Kubernetes). The `default` namespace is used, if a namespace name is not provided.

You can write your resources to one or multiple yaml files, and apply them to your local namespace using the CLI (or even manually, if needed).

Here are the resources we need:

* **`Site`**

A `Site` is a mandatory resource. You must have a `Site` defined in order to have a Skupper instance on your namespace.

```yaml
apiVersion: skupper.io/v2alpha1
kind: Site
metadata:
  name: fedora
```

* **`RouterAccess`**

The `RouterAccess` is created automatically when you create a Site using the CLI, passing the `--enable-link-access` flag.

This resource, specially on System sites, tells Skupper that your site will accept incoming links from other Skupper sites.

On System sites, it is also used to determine the `host` and `ports` that will be bound. And it also specifies the `subjectAlternativeNames` which determines for which hostnames and IP addresses the self-signed certificate will be valid for.

In this example, we are using `my.podman.host` as the hostname that will be defined in the Skupper `Link` which will be applied to the Skupper `Site` running on Kubernetes.

> ___
> **_IMPORTANT_**
>
> Make sure that the `my.podman.host` hostname resolves to a valid IP that can be reached by your Kubernetes site.
> ___

As you can see below, the `RouterAccess` is set to use the default ports: `55671` and `45671`, to accept incoming links from other sites.

```yaml
apiVersion: skupper.io/v2alpha1
kind: RouterAccess
metadata:
  name: router-access-fedora
spec:
  tlsCredentials: router-access-fedora
  roles:
  - name: inter-router
    port: 55671
  - name: edge
    port: 45671
  subjectAlternativeNames:
  - my.podman.host
```

* **`Listener`**

The `Listener` below will bind all IPv4 addresses on port `5672` locally on your bare-metal Site running with Podman.

Note that we are using `routingKey: mybroker`. Therefore, the corresponding `Connector` that will be created on the Kubernetes Site, needs to use the same value.

```yaml
apiVersion: skupper.io/v2alpha1
kind: Listener
metadata:
  name: mybroker
spec:
  host: 0.0.0.0
  port: 5672
  routingKey: mybroker
  type: tcp
```

Just to illustrate, you can also create these resources using the CLI directly. The CLI has two different commands that can be used to create resources.

To create the resources above you can use:

1. The CLI to create the resources

```bash
skupper --platform podman site create fedora --enable-link-access
skupper --platform podman listener create mybroker 5672
```

> ___
> **_NOTE_**
>
> You need to tweak the generated `RouterAccess` resource, adding the `my.podman.host` entry to the `subjectAlternativeNames` if you use the CLI to genereate them.
> ___

Alternatively, you could also use the `generate` command (instead of `create`). The main difference, is that the `create` command actually create the resources on the respective platform (in this case, `podman`).

While the `generate` command simply displays the content of the YAML to the standard output.

2. The CLI to apply the existing YAML file to your namespace (alternative)

If you combine all your resources in a single YAML file, you can run as shown below:

```bash
skupper --platform podman system apply -f ./definition.yaml
```

### Initialize the Podman site

Before intializing your Podman `Site`, it is recommended that you have the `podman.socket` service enabled as well as install the Skupper V2 Controller for System sites.

Run the following command to ensure you have everything ready:

```bash
skupper --platform podman system install
```

To start your Podman site, now that all your resources have been created, just run:

```bash
skupper --platform podman system start
```

It will create a static Podman `Site` on your bare metal Linux machine.
If, for any reason, you need to make changes to your resources, you'll need to reload your `Site` definition, using:

```bash
skupper --platform podman system reload
```

The ouput of `system start` or `system reload` includes the location of your site definition, like:

```
Definition is available at: ~/.local/share/skupper/namespaces/default/input/resources
```

Save the directory above, as it will be used in the next step.

### Install the custom certificates

Custom certificates have been prepared and placed under `ansible/podman/router-access-fedora` and `ansible/podman/client-router-access-fedora`. The first directory contains the server certificates and the second contains the client certificates.

> ___
> **_NOTE_**
>
> The directory name for the **Server** certificates is defined after the `RouterAccess` name, or the `RouterAccess.spec.tlsCredentials` value. In this case `router-access-fedora`.
> The directory name where the **Client** certificates are provided, must be the directory named used by the **Server** certificates, but prefixed with `client-`. Therefore, `client-router-access-fedora`.

Now that the Podman `Site` has been started, you need to copy your custom certificates into the `input/certs` directory. If your site definition is available at: `~/.local/share/skupper/namespaces/default/input/resources`, you should copy your certificate directories into: `~/.local/share/skupper/namespaces/default/input/certs`.

Adjust the path below, and copy the provided certificates to the correct location:

```bash
cp -r ansible/podman/router-access-fedora/ ~/.local/share/skupper/namespaces/default/input/certs
cp -r ansible/podman/client-router-access-fedora/ ~/.local/share/skupper/namespaces/default/input/certs
```

Now that the custom certificates have been provided, reload the Podman `Site` definition.

```bash
skupper --platform podman system reload
```

Note that you should see the following message as part of the output:

```
-> User provided server certificate found: router-access-fedora
-> User provided client certificate found: client-router-access-fedora
```

And you will also see another message saying something like:

```
Static links have been defined at: ~/.local/share/skupper/namespaces/default/runtime/links
```

The path above contains the static links generated for your Podman `Site`.
You should have a file named `link-router-access-fedora-my.podman.host.yaml` inside this directory.

Copy that file, as it will be used to establish a Link from your Kubernetes `Site` to your Podman `Site`.

## Running your Kubernetes Site

### Creating the resources on your Kubernetes site

The first resource that needs to be created is the `Site` CR.
Let's create it using the CLI:

> ___
> **_NOTE_**
>
> Make sure your **KUBECONFIG** environment variable is pointing to a valid Kubernetes cluster definition, as the CLI relies on it.
> ___

1. Create your **`Site`**

```bash
skupper -n broker site create ocp-site
```

2. Create the **`Connector`**

The Connector resource, exposes a workload into the VAN.
We are going to use the selector `ActiveMQArtemis=artemis-broker` to target the ActiveMQ Artemis pod.

Here is the `Connector` definition:

```yaml
apiVersion: skupper.io/v2alpha1
kind: Connector
metadata:
  name: mybroker
spec:
  port: 5672
  routingKey: mybroker
  selector: ActiveMQArtemis=artemis-broker
  type: tcp
```

Using the CLI:

```bash
skupper -n broker connector create mybroker 5672 --selector "ActiveMQArtemis=artemis-broker"
```

3. Create the **`Link`**

Apply the `Link` definition from the Podman `Site` into your Kubernetes cluster, using:

```bash
kubectl -n broker apply -f ~/.local/share/skupper/namespaces/default/runtime/links/link-router-access-fedora-my.podman.host.yaml
```

### Validate your VAN

All the steps have been performed and now it is time to validate if your VAN is working as expected.

Run:

```bash
kubectl -n broker get site
```

Expected output:

```
NAME       STATUS   SITES IN NETWORK   MESSAGE
ocp-site   Ready    2                  OK
```

Then validate if the `Listener` and `Connector` are working, run:

```bash
kubectl -n broker get connector
```

Expected output:

```
NAME       ROUTING KEY   PORT   HOST   SELECTOR                         STATUS   HAS MATCHING LISTENER   MESSAGE
mybroker   mybroker      5672          ActiveMQArtemis=artemis-broker   Ready    true                    OK
```

## Validation

Now get back to the [README.md](README.md#validation-your-scenario) and proceed with the validation.
