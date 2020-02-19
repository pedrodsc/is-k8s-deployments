# Step by step 'is' deployment with minikube

I will describe you 2 methods, the first is faster, but less secure and not recommended unless you know what you are doing.
[Issues of this method here](https://minikube.sigs.k8s.io/docs/reference/drivers/none/)

The second one takes more time and is safer, with one caveat, if you only have one GPU, you will not be able to use it at the same time your cluster does.
(Also for now you will need a dualboot. Later I will try with a modified kernel entry)

## First Method

**1.1** install minikube

**1.2** install kubectl

**1.3** set vm-driver as none(be aware that you need root privileges)
    
    $ sudo minikube config set vm-driver none

**1.4** Enable GPU
    
To be able to use the GPU, if you only have one, you have to do the following:
    
    $ sudo minikube start --vm-driver=none --apiserver-ips 127.0.0.1 --apiserver-name localhost
    
    $ sudo kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/master/nvidia-device-plugin.yml

Be aware that by using this methods some commands like 'minikube start' and 'kubectl' must be run as root.

Done.

## Second Method

First both your CPU and Motherboard must support IOMMU. You need to enable IOMMU in BIOS before proceeding.

**2.1** Install your second Linux distro and boot into it.

**2.2** Install the following packages

  $ sudo apt-get install qemu-kvm libvirt-bin virt-top  libguestfs-tools virtinst bridge-utils

**2.3** Add those kernel parameters:

`intel_iommu=on` for intel or `amd_iommu=on` for AMD. Also add `iommu=pt`.
    
If you use GRUB add the parameters in `GRUB_CMDLINE_LINUX_DEFAULT` inside `/etc/default/grub` like this
    
    GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity amd_iommu=on iommu=pt"
    
Update grub.

    $ sudo upgrade-grub
    
Now reboot.

**2.4** Tell the kernel to not touch your GPU.

    If everything went right, now everything should be separated in IOMMU groups.
    
**2.4.1** Run this script to get the id of your GPU
    
    #!/bin/bash
    shopt -s nullglob
    for g in /sys/kernel/iommu_groups/*; do
        echo "IOMMU Group ${g##*/}:"
        for d in $g/devices/*; do
            echo -e "\t$(lspci -nns ${d##*/})"
        done;
    done;
    
Mine is an RTX2060 and the ids of the GPU can be seen here.
![RTX2060 IOMMU](/zimg/iommu.png)
    
**2.4.2** Create a file called vfio.conf inside /etc/modprobe.d/

  `$ sudo vim /etc/modprobe.d/vfio.conf`

and write this into it
  
  `options vfio-pci ids=10de:1f08,10de:10f9,10de:1ada,10de:1adb`
    
**MODIFY FOR YOUR GPU!!!**
    
**2.4.3** Update initramfs
    
    $ sudo update-initramfs -u

Now reboot.

**2.5** Install minikube, kubectl and docker-machine-driver-kvm2

**2.5.1** Minikube
    
    $ wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    $ chmod +x minikube-linux-amd64
    $ sudo mv minikube-linux-amd64 /usr/local/bin/minikube
    
Test with '$ minikube version'

**2.5.2** Kubectl
    
    $ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    $ chmod +x kubectl
    $ sudo mv kubectl  /usr/local/bin/
    
**2.5.3** docker-machine-driver-kvm2
    
    $ curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2
    $ chmod +x docker-machine-driver-kvm2
    $ sudo mv docker-machine-driver-kvm2 /usr/local/bin/
    

**2.6** Start minikube and install nvidia plugins

    $ minikube start --vm-driver kvm2 --kvm-gpu
    
Test with '$ kubectl get po -A'
    
    $ minikube addons enable nvidia-gpu-device-plugin
    $ minikube addons enable nvidia-driver-installer

Done.

Test with `$ kubectl get nodes -ojson | jq .items[].status.capacity`
    
# Deploying

At this point you should have a working cluster, but to run the 'is' you need to make some changes to the existent deployments in labviros/is-k8s-deployments.

**3** Modify deployments files
    
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
    
**4** Apply your deployments

    $ sudo kubectl apply -f "your deployment"
    
