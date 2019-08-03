# kubernetes-raspi
Instructions/config files for setting up my Raspberry Pi Kubernetes cluster

## Raspberry Pi Build
This is the set of instructions for building out a Raspberry Pi image, getting Kubernetes installed, and nodes communicating.  

### All Nodes
1. Standard Build (timezone, hostnames, etc)
  1. Burn Image
      ```
      sudo diskutil UmountDisk /dev/disk2
      sudo dd bs=1m if=~/Downloads/2019-04-08-raspbian-stretch-lite.img of=/dev/rdisk2 conv=sync
      touch /Volumes/boot/ssh
      sudo diskutil UmountDisk /dev/disk2  
      ```
  1. Boot Image in raspberry pi
  1. Connect (initially will be raspberrypi.local)
  1. use `raspi-config` to set password, set timezone, hostname (k8s-master, k8s-worker-0n)
  1. Give static IP by modifying `/etc/dhcpcd.conf`
    - 192.168.1.50  k8s-master
    - 192.168.1.5*n*  k8s-worker-0*n*
        ```
        interface eth0
        static ip_address=192.168.1.50/24
        static routers=192.168.1.1
        static domain_name_servers=192.168.1.1
        ```
  1. Reboot
1. Install docker
    ```
    curl -sSL get.docker.com | sh
    sudo usermod pi -aG docker
    newgrp docker
    ```
1. Disable Swap
    ```
    sudo dphys-swapfile swapoff
    sudo dphys-swapfile uninstall
    sudo update-rc.d dphys-swapfile remove
    ```
1. Check swap (Make sure it's not there)
    ```
    sudo swapon --summary
    ```
1. Modify cmdline.txt to enable cgroup_memory
  1. Open file for editing
      ```
      sudo vi /boot/cmdline.txt
      ```
  1. Add to end (no return):
      ```
      cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
      ```
1. Ran `rpi-update` to get latest kernel.  Prior to that I had some issues with the cluster starting, something about pid limiting.
1. Reboot (Makes cgroup take effect)
1. Install kubernetes
    ```
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update -q
    sudo apt-get install -qy kubeadm
    ```

### Master Node
1. Pre-pull docker images:
    ```
    sudo kubeadm config images pull -v3
    ```
1. Init master node:
    ```
    sudo kubeadm init --token-ttl=0
    ```
1. Save copy of the .conf file with init parameters and stuff:
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
1. Save off join info.  I'm not sure how to regenerate this if needed:
    ```
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

     kubeadm join 192.168.1.50:6443 --token XXXXXX.XXXXXXXXXXXXXXXX \
        --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    ```

### Worker Node
1. Join master by using command generated in output of final step:
    ```
    sudo kubeadm join 192.168.1.50:6443 --token XXXXXX.XXXXXXXXXXXXXXXX \
        --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    ```

### Bring Up
1. List nodes using `kubectl get nodes` - Nodes have a status "NotReady" because they don't have a container network in place.  Install weave as the container network (from master):
    ```
    kubectl apply -f https://git.io/weave-kube-1.6
    kubectl get pods --all-namespaces
    ```



## Install Metallib
    [Metallib](https://metallb.universe.tf/concepts/) creates a LoadBalancer service type that lets me request a service of type LoadBalancer, which will be automatically assigned an IP from my configured pool.

    1. Install the deployment:
      ```
      kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
      ```

    1. Apply the configuration - In my file, setting up Layer 2 mode and reserving 192.168.1.40-192.168.1.49:
      ```
      kubectl apply -f metallib/metallb-conf.yaml
      ```


## Install Traefik
      [Traefik](https://traefik.io) will handle the load balancing on the service level, allowing any deployment to be exposed by subdomain.  Traefik will be deployed with a LoadBalancer service type (with a static IP), then mapped as a wildcard on my internal DNS.  This way any lookups into \*.my.domain.com will go to that IP and Traefik will forward to the appropriate pod based on it's rules.  

      1. The config files can be modified to suit the domain/IPs it's being configured for, and then deployed thusly:
        ```
        kubectl apply -f traefik/traefik-internal-configmap.yaml
        kubectl apply -f traefik/traefik-internal-deployment.yaml
        kubectl apply -f traefik/traefik-internal-service.yaml
        kubectl apply -f traefik/traefik-rbac.yaml
        ```





## Configure Persistent Volumes

### Create NFS Storage to Share
        1. Plug in an external disk - I had a USB disk I plugged in on k8s-master.
        1. Install NFS server software.  I did this on k8s-master.  I had to put the insecure tag in `/etc/exports`, else for some reason I couldn't mount it.
        ```
        sudo apt-get install nfs-kernel-server nfs-common
        sudo systemctl enable nfs-kernel-server

        # Add following line to /etc/exports

        /mnt/ext_disk/kubernetes .*(rw,sync,no_subtree_check,no_root_squash,insecure)

        sudo exportfs -a
        ```

### Create StorageClass in Kubernetes
        What we are doing here is creating a new StorageClass containing a persistent volume.  


        kubectl apply -f all the files
        kubectl patch storageclass nfs-ssd1 -p '{"metadata":{"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'

        How I use storageclass?

### List it:
        ```
        kubectl get storageclass
        ```

### Create a claim on it:
        ```
        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          name: test-claim
          annotations:
        #    volume.beta.kubernetes.io/storage-class: "nfs-master-ssd"
        spec:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 1Mi
        ```
