# Automated procedure using Ansible

This procedure does not cover the details about the Skupper resources needed, if just sets the exact same scenario automatically.

If you want to know the details about the Skupper Resources needed, as well as all the other steps, please review the [Manual Procedure](MANUAL.md).

## The Playbook

An Ansible playbook is available to set the whole scenario up.
You can find it at: `ansible/playbook.yaml`.

It does exactly the same things that is done through the manual process.

So we will focus on explaining just the `skupper.v2` Ansible collection here.

> ___
> **_NOTE_**
>
> The new `skupper.v2` collection, in spite of the previous collections used in v1, do not rely on the CLI to execute commands.
> ___

### Creating the Podman Site resources

To create resources, you can use the `skupper.v2.resource` module.
It works for all supported platforms.

In this example, we have already prepared the YAML files to be applied, so the module is executed with the following options:

```yaml
    - name: Create the Skupper V2 resources
      skupper.v2.resource:
        path: "{{ item }}"
        platform: podman
      loop:
        - ./podman/Site-fedora.yaml
        - ./podman/RouterAccess-router-access-fedora.yaml
        - ./podman/Listener-mybroker.yaml
```

This will place the provided resources on the correct location in the filesystem for the given user and namespace.

### Start the Podman Site

Next we start the Podman site, by executing the `skupper.v2.system` module.
This module returns a few variables and we will use the `path` return variable next, to place the custom certificates in the correct location, as it provides the directory where the `Site` has been defined.

To start the Podman `Site` and store the returned values into the `system` variable, we use:

```yaml
    - name: Starting the default namespace
      skupper.v2.system:
        action: start
        platform: podman
      register: system
```

### Copying the custom certificates

Next we place the custom certificates in the correct diretory, by using the `ansible.builtin.copy` module. It simply copies the `./podman/router-access-fedora` and `./podman/client-router-access-fedora` directories into the correct location.

The location is defined under the `system` variable stored in the previous task, at `system.path`.
Here is how we copy the certificates to the correct place:

```yaml
    - name: Copy user provided certificates to skupper input certs path
      ansible.builtin.copy:
        src: "{{ item }}"
        dest: "{{ system.path }}/input/certs/"
      loop:
        - ./podman/router-access-fedora
        - ./podman/client-router-access-fedora
```

### Reload the Podman site to include the custom certs

Now we need to reload the Site definition. When we `start` or `reload` a site definition using the `skupper.v2.system` module, along with the `path` a dictionary of `links` is also returned.

So we will store the returned values into the `system` variable again, as we will use the `links` dictionary to create the `Link` from the Kubernetes `Site` to the Podman `Site`.

```yaml
    - name: Reload the podman site
      skupper.v2.system:
        action: reload
        platform: podman
      register: system
```

### Create the resources on the Kubernetes site

To create the resources on the Kubernetes site, we just need to apply the resources needed using the `skupper.v2.resource` module.


```yaml
    - name: Create the Skupper resources on OCP
      skupper.v2.resource:
        path: "{{ item }}"
        namespace: broker
        state: latest
      loop:
        - ./ocp/Site-ocp-site.yaml
        - ./ocp/Connector-mybroker.yaml
```

### Create the Link from Kubernetes to the Podman site

Again, we will use the `skupper.v2.resource` module, but this time, we will use the definition that was returned by the `skupper.v2.system` module execution, applying the static link defined for the `my.podman.host` hostname.

```yaml
    - name: Create the Link to the Podman site on OCP
      skupper.v2.resource:
        def: "{{ system.links['my.podman.host'] }}"
        namespace: broker
        state: latest
```

### Install the Controller for System sites

The last step is just to install the Controller for System sites, so that we can see updated status for the custom resources for the Podman site.

```yaml
    - name: Enabling the system controller on your local host
      skupper.v2.controller:
        action: install
        platform: podman
```

## Executing the playbook

The playbook assumes you're using your own host machine, as the Bare Metal machine where a Podman `Site` will be created.

To run the playbook, execute:

```bash
(cd ansible; ansible-playbook -i localhost, playbook.yaml -v)
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

Now get back to the [README.md](README.md) and proceed with the validation.
