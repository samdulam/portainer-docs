# Deploy Portainer in Linux environments

## Deploy Portainer in Kubernetes

To deploy Portainer within a Kubernetes cluster, you can either use our HELM chart, or our provided manifests.

### Pre-Req Note:
Portainer requires data persistence, and as a result needs at least one storage-class available to use. Portainer will attempt to use the "default" storage class during deployment. If you do NOT have a storage class tagged as "default" the deployment will likely fail.

You can check if you have a default storage class by running:

<pre><code> > kubectl get sc </code></pre>

and looking for a storage class with (default) after its name:

![defaultsc](assets/defaultsc.png)

If you want to make a storage class the default, you can type the command:

<pre><code> >kubectl patch storageclass <storage-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' </code></pre>

and replace <storage-class-name> with the name of your storage class (eg: kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Alternatively, if you are using HELM you can use:
<pre><code> --set persistence.storageClass=<storage-class-name> </code></pre>

### Using Helm

Ensure you're using at least helm v3.2, which [includes support](https://github.com/helm/helm/pull/7648) for the `--create-namespace` argument.


First, add the Portainer helm repo running the following:

<pre><code> helm repo add portainer https://portainer.github.io/k8s/</code></pre>
<pre><code> helm repo update</code></pre>


<!-- Then, create the Portainer namespace in your cluster

<pre><code> kubectl create namespace portainer</code></pre> -->

#### For NodePort

Using the following command, Portainer will run in the port 30777

<pre><code> helm install --create-namespace -n portainer portainer portainer/portainer</code></pre>

#### For Load Balancer

Using the following command, Portainer will run in the port 9000.

<pre><code> helm install --create-namespace -n portainer portainer portainer/portainer \
--set service.type=LoadBalancer</code></pre>

#### For Ingress

<pre><code> helm install --create-namespace -n portainer portainer portainer/portainer \
--set service.type=ClusterIP</code></pre>

### Using YAML Manifest

<!-- First create the Portainer namespace in your cluster

<pre><code> kubectl create namespace portainer</code></pre> -->

#### For NodePort

Using the following command, Portainer will run in the port 30777

<pre><code> kubectl apply -n portainer -f https://raw.githubusercontent.com/portainer/k8s/master/deploy/manifests/portainer/portainer.yaml</code></pre>

#### For Load Balancer

<pre><code>kubectl apply -n portainer -f https://raw.githubusercontent.com/portainer/k8s/master/deploy/manifests/portainer/portainer-lb.yaml</code></pre>

---
**Note about Persisting Data**

The charts/manifests will create a persistent volume for storing Portainer data, using the default StorageClass.

In some Kubernetes clusters (microk8s), the default Storage Class simply creates hostPath volumes, which are not explicitly tied to a particular node. In a multi-node cluster, this can create an issue when the pod is terminated and rescheduled on a different node, "leaving" all the persistent data behind and starting the pod with an "empty" volume.

While this behaviour is inherently a limitation of using hostPath volumes, a suitable workaround is to use add a nodeSelector to the deployment, which effectively "pins" the portainer pod to a particular node.

The nodeSelector can be added in the following ways:

1. Edit your own values.yaml and set the value of nodeSelector like this:

        nodeSelector: kubernetes.io/hostname: \<YOUR NODE NAME>

2. Explicictly set the target node when deploying/updating the helm chart on the CLI, by including `--set nodeSelector.kubernetes.io/hostname=<YOUR NODE NAME>`
   
3. If you've deployed Portainer via manifests, without Helm, run the following one-liner to "patch" the deployment, forcing the pod to always be scheduled on the node it's currently running on:

        kubectl patch deployments -n portainer portainer -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "'$(kubectl get pods -n portainer -o jsonpath='{ ..nodeName }')'"}}}}}' || (echo Failed to identify current node of portainer pod; exit 1)

---

## Deploy Portainer in Docker

Portainer is comprised of two elements, the Portainer Server, and the Portainer Agent. Both elements run as lightweight Docker containers on a Docker engine or within a Swarm cluster. Due to the nature of Docker, there are many possible deployment scenarios, however, we have detailed the most common below. Please use the scenario that matches your configuration.

Note that the recommended deployment mode when using Swarm is using the Portainer Agent.

By default, Portainer will expose the UI over the port 9000 and expose a TCP tunnel server over the port 8000. The latter is optional and is only required if you plan to use the Edge compute features with Edge agents.

To see the requirements, please, visit the page of [requirements](/v2.0/deploy/requirements).

### Docker Standalone

Use the following Docker commands to deploy the Portainer Server; note the agent is not needed on standalone hosts, however it does provide additional functionality if used (see Portainer and agent scenario below):

<pre><code> docker volume create portainer_data</code></pre>

<pre><code> docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce</code></pre>

### Docker Swarm

Deploying Portainer and the Portainer Agent to manage a Swarm cluster is easy! You can directly deploy Portainer as a service in your Docker cluster. Note that this method will automatically deploy a single instance of the Portainer Server, and deploy the Portainer Agent as a global service on every node in your cluster.

<pre><code> curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml</code></pre>
<pre><code>docker stack deploy -c portainer-agent-stack.yml portainer</code></pre>

<b>Note</b>: By default this stack doesn't enable Host Management Features, you need to enable from the UI of Portainer.

## Portainer Agent Deployments Only

### Docker Standalone
Run the following command to deploy the Agent in your Docker host.

<pre><code>docker run -d -p 9001:9001 --name portainer_agent --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/docker/volumes:/var/lib/docker/volumes portainer/agent</code></pre>

Note: <code>--tlsskipverify</code> has to be present when deploying an agent, since injecting valid, signed certs in the agent is not a supported scenario at present.

### Docker Swarm
Deploy Portainer Agent on a remote LINUX Swarm Cluster as a Swarm Service, run this command on a manager node in the remote cluster.

First create the network:

<pre><code>docker network create portainer_agent_network</code></pre>

The following step is deploy the Agent:

<pre><code> docker service create --name portainer_agent --network portainer_agent_network --publish mode=host,target=9001,published=9001 -e AGENT_CLUSTER_ADDR=tasks.portainer_agent --mode global --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock --mount type=bind,src=//var/lib/docker/volumes,dst=/var/lib/docker/volumes --mount type=bind,src=/,dst=/host portainer/agent</code></pre>

Note: <code>--tlsskipverify</code> has to be present when deploy an agent and the certs in the agent is not a supported scenario at this moment.

## :material-note-text: Notes

[Contribute to these docs](https://github.com/portainer/portainer-docs/blob/master/contributing.md){target=_blank}
