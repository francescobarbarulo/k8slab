# Lab 05

In this lab you are going to install the Cilium CNI plugin, add a worker node to the cluster. Create, scale and update a Deployment.

## Install a CNI plugin

🖥️ Open a shell in the `student` machine.

1. Kubernetes is not opinionated, it lets you choose your own CNI solution. Until a CNI plugin is installed the cluster will be inoperable. List the running Pods in all namespaces (`-A`):

    ```sh
    kubectl get pod -A
    ```

    You should see the CoreDNS Pods not ready and they will not start up before a network is installed.

2. Install [Cilium](https://cilium.io/) as our CNI plugin.

    ```sh
    curl -sL https://raw.githubusercontent.com/francescobarbarulo/kubernetes-starter-pack/main/scripts/cni-cilium-install.sh | sh
    ```

    The script will deploy two Pods:
    * `cilium-operator`
    * `cilium-agent`

    The Operator is responsible for managing duties in the cluster which should logically be handled once for the entire cluster.
    The Agent runs on each node in the cluster and configures Pod network interfaces as workloads are created and deleted.

3. Great! Now if you take a look at the cluster it should be in the `Ready` status.

    ```sh
    kubectl get nodes
    ```

    > If the node is still in the `NotReady` status, wait a few more seconds and retry.

4. Let's take a look at the running Pods too by running. CoreDNS and Cilium pods are now running.

    ```sh
    kubectl get pod -n kube-system
    ```

    Note that the Pods you have so far are part of the Kubernetes system itself, therefor they run in a namespace called `kube-system`.

## Add a worker node

⚠️ Do not go ahead until the `k8s-cp-01` is in `Ready` state.

🖥️ Open a shell in `k8s-cp-01` environment.

1. Create a new token and print the join command. Save it somewhere because it will be needed in step 4.

    ```sh
    kubeadm token create --print-join-command
    ```

🖥️ Open a shell in the `k8s-w-01` environment.

Like the control-plane node, the new worker node must be prepared installing and configuring prerequisites (e.g. enabling IPv4 forwarding and letting iptables see bridged traffic), a container runtime (`containerd` and `runc`), `kubeadm` and `kubelet`.

2. Set some environment variables.

    ```sh
    export ARCH=amd64
    export RUNC_VERSION=1.1.11
    export CONTAINERD_VERSION=1.7.11
    export CRICTL_VERSION=1.29.0
    export K8S_VERSION=1.29.4
    export REGISTRY=172.30.10.11:5000
    ```

3. Install required tools.
    ```sh
    curl -sL https://raw.githubusercontent.com/francescobarbarulo/kubernetes-starter-pack/main/scripts/lab/node-prep.sh | sh
    ```

4. Run the command printed at step 1 in order to join the worker node to the existing cluster.

🖥️ Open a shell in the `student` machine.

5. Verify the the newly joined worker node is in `Ready` state.

    ```sh
    kubectl get nodes
    ```

## Deploy an application

⚠️ Do not go ahead until the `k8s-w-01` is in `Ready` state.

🖥️ Open a shell in the `student` machine.

1. Let’s deploy our first app on Kubernetes with the kubectl create deployment command. You need to provide the deployment name and app image location.

    ```sh
    kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:1.0
    ```

    Great! You just deployed your first application by creating a deployment in the `default` namespace. This performed a few things for you:
    * searched for a suitable node where an instance of the application could be run (you have only 1 available node);
    * scheduled the application to run on that Node;
    * configured the cluster to reschedule the instance on a new Node when needed.

2. List the deployments by running:

    ```sh
    kubectl get deployments
    ```

    The output is similar to this:

    ```plaintext
    NAME        READY   UP-TO-DATE   AVAILABLE   AGE
    hello-app   1/1     1            1           12s
    ```

3. Run the following command to see the ReplicaSet created by the Deployment.

    ```sh
    kubectl get replicasets
    ```

    The output is similar to this:

    ```sh
    NAME                   DESIRED   CURRENT   READY   AGE
    hello-app-5c7f66c6b6   1         1         1       1m8s
    ```

    **Note**: The name of the ReplicaSet is always formatted as `[DEPLOYMENT-NAME]-[HASH]`. This name will become the basis for the Pods which are created.

4. The Deployment automatically generates labels for each Pod in order to use them in the selctor. Run the following to see the Pod's labels.

    ```sh
    kubectl get pods --show-labels
    ```

5. If you try to delete a Pod belonging to a Deployment, the ReplicaSet controller will recreate a new one.

    ```sh
    POD_NAME=$(kubectl get pods -l app=hello-app -o jsonpath='{range .items[*]}{.metadata.name}{end}')
    kubectl delete pod $POD_NAME
    ```

    **Note**: The Pod name is changed.

6. Anything that the application would normally send to `STDOUT` becomes logs for the container within the Pod. Retrieve these logs to verify the server is up:

    ```sh
    POD_NAME=$(kubectl get pods -l app=hello-app -o jsonpath='{range .items[*]}{.metadata.name}{end}')
    kubectl logs $POD_NAME
    ```

    The output is similar to this:

    ```plaintext
    1970/01/01 16:20:52 Server listening on port 8080
    ```

    > You don't need to specify the container name, because you only have one container inside the pod.

## Scaling the deployment

In order to facilitate more load, you may need to scale up the number of replicas for a microservice.

🖥️ Open a shell in the `student` machine.

1. Scale the deployment by running:

    ```sh
    kubectl scale deployments/hello-app --replicas=4
    ```

2. Show the deployments to verify the current number of replicas matches the desired one.

    ```sh
    kubectl get deployments
    ```

    You should see `4/4` in the `READY` column. If not, run the command above again.

3. The change was applied, and you have 4 instances of the application available. Next, let’s check if the number of Pods changed:

    ```sh
    kubectl get pods -o wide
    ```

4. There are 4 Pods now, with different IP addresses. The change was registered in the Deployment events log. To check that, use the describe command:

    ```sh
    kubectl describe deployments/hello-app
    ```

## Deploy a new version of the application

🖥️ Open a shell in the `student` machine.

1. To update the image of the application to version 2 run:

    ```sh
    kubectl set image deployments/hello-app hello-app=gcr.io/google-samples/hello-app:2.0
    ```

    The command notified the Deployment to use a different image for your app and initiated a rolling update.

2. Check the status of the new Pods, and view the old one terminating.

    ```sh
    kubectl get pods -o wide
    ```

    **Note**: The new Pods get new IP addresses.

3. Verify the rollout is successfully completed.

    ```sh
    kubectl rollout status deployment hello-app
    ```

4. Let's list the replicasets to see that the Deployment updated the Pods by creating a new ReplicaSet and scaling it up to 4 replicas, as well as scaling down the old ReplicaSet to 0 replicas.

    ```sh
    kubectl get replicasets
    ```

    The output is similar to this:

    ```plaintext
    NAME                   DESIRED   CURRENT   READY   AGE
    hello-app-5c7f66c6b6   4         4         4       15m
    hello-app-f4b774b69    0         0         0       18s
    ```

## Rolling back a deployment

🖥️ Open a shell in the `student` machine.

1. Suppose that you made a typo while updating the Deployment, by putting the image name as `hello-app:2.1` instead of `hello-app:2.0`.

    ```sh
    kubectl set image deployments/hello-app hello-app=gcr.io/google-samples/hello-app:2.1
    ```

2. The rollout gets stuck. You can verify it by checking the rollout status:

    ```sh
    kubectl rollout status deployment hello-app
    ```

    The output is similar to this:

    ```plaintext
    Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
    ```
    Press `Ctrl-C` to stop the above rollout status watch.

3. Looking at the Pods created, you can see that the 2 Pods created by new ReplicaSet are stuck in an image pull loop.

    ```sh
    kubectl get pods
    ```

    The output is similar to this:

    ```plaintext
    NAME                         READY   STATUS             RESTARTS   AGE
    hello-app-5c7f66c6b6-dm2zn   1/1     Running            0          35m
    hello-app-5c7f66c6b6-fklpb   1/1     Running            0          35m
    hello-app-5c7f66c6b6-nqll9   1/1     Running            0          35m
    hello-app-6f7cc84b47-2mqll   0/1     ImagePullBackOff   0          10s
    hello-app-6f7cc84b47-qhpwp   0/1     ImagePullBackOff   0          10s
    ```

    **Note**: The Deployment controller stops the bad rollout automatically, and stops scaling up the new ReplicaSet.

4. To fix this, you need to rollback to a previous revision of Deployment that is stable.

    ```sh
    kubectl rollout undo deployment hello-app
    ```

5. Check if the rollback was successful and the Deployment is running as expected, run:

    ```sh
    kubectl get deployment hello-app
    ```

    The output is similar to this:

    ```plaintext
    NAME        READY   UP-TO-DATE   AVAILABLE   AGE
    hello-app   4/4     4            4           1m21s
    ```

## Next

[Lab 06](./lab06.md)