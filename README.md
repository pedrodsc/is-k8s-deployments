# Step by step kubernetes deployment on a single node with GPU

This guide was made for a specific project, but most of it can be used for any single node "cluster"

**1.1** prepare your system

Disable swap

    $ sudo swapoff -a

Comment this line in `/etc/fstab`

    #/swap.img      none    swap    sw      0       0

This will disable swap at boot.

**1.2** install docker

    $ sudo apt-get install docker.io
    
Add yourself to the `docker` group

    $ sudo usermod -aG docker $USER
    
**1.3** install kubectl
    
    $ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    $ chmod +x kubectl
    $ sudo mv kubectl /usr/local/bin/

**1.4** install kubernetes

    $ sudo apt-get install kubeadm kubelet kubectl
    
    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
**1.5** Enable deploys on master

    $ kubectl taint nodes --all node-role.kubernetes.io/master-
    
**1.6** Deploy weave

    $ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    
**1.7** install nvidia driver

    $ sudo apt install nvidia-driver-440
    
    $ kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.6.0/nvidia-device-plugin.yml

**1.8**

reboot.

# Deploying

At this point you should have a working cluster.
Check with `kubectl get pods -A`.

**2.1** How to deploy

    $ kubectl apply -f "your deployment"

If you don't know what 'is' is, you are done. Have fun with your cluster!

## is specific

To be able to run the 'is' you need to make some changes to the existing deployments in labviros/is-k8s-deployments.

**2.2** Modify deployments files
    
Change all `apiVersion: extensions/v1beta1` with `apiVersion: apps/v1` and add the `selector` field
    
eg:
    
    - apiVersion: extensions/v1beta1
    + apiVersion: apps/v1
    ...
    spec:
        replicas: 1
    +   selector:
    +       matchLabels:
    +       app: rabbitmq
        template:
            metadata:
            
 Deploy as in *2.1*
    

    
