---
layout: post
title: "Running a Windows VM on KubeVirt on K3s"
date: 2020-11-03
tags: kubernetes windows kubevirt k3s
---

To get the obvious "WTF, why would you wanna do this?" out of the way: We had to. For reasons.

### Reasons

In cases where you need a Windows VM, but don't want to leave your Kubernetes platform, [KubeVirt](https://kubevirt.io/) is your friend. Be warned that this is going to be a heavy workload, so go for something with a lot of CPU, memory and disk space.

### Getting started

Let's start with installing K3s and KubeVirt on an Ubuntu 20.10 instance.
1. Install K3s ([docs](https://rancher.com/docs/k3s/latest/en/installation/install-options/)).

    ```
    apt update && apt -y upgrade
    curl -sfL https://get.k3s.io | sh -
    ```

    Check if the install succeeded, e.g. by creating a deployment and seeing if pods are running.

2. Install KubeVirt ([docs](https://kubevirt.io/quickstart_minikube/)). I used the operator, but any of those should do the trick.

    ```
    export VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt/releases | grep tag_name | grep -v -- '-rc' | head -1 | awk -F': ' '{print $2}' | sed 's/,//' | xargs)
    
    echo $VERSION
    kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
    kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
    ```
    
    Check if the install succeeded with
    
    ```
    kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.phase}"
    kubectl get all -n kubevirt
    ```

3. Install virtctl to control KubeVirt VMs. I used the Kubernetes plugin manager [Krew](https://krew.sigs.k8s.io/).

    ```
    apt-get install git
    (   set -x; cd "$(mktemp -d)" &&   curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&   tar zxvf krew.tar.gz &&   KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&   "$KREW" install krew; )

    export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
    kubectl krew install virt
    ```

4. Install [Containerized-Data-Importer (CDI)](https://github.com/kubevirt/containerized-data-importer/blob/master/README.md) to manage your VM disk later on. I used the operator version.

    ```
    export VERSION=$(curl -s https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -o "v[0-9]\.[0-9]*\.[0-9]*")
    kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
    kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
    ```

5. Download a Windows ISO file (the [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019) lets you trial a Windows ISO for 180 days).

6. Upload the ISO.

    ``` 
    # Get the CDI upload proxy service IP:
    kubectl get svc -n cdi
    
    # Upload 
    kubectl virt image-upload --image-path </path/to/iso> \
        --pvc-name iso-win2k19 --access-mode ReadWriteOnce \
        --pvc-size 10G --uploadproxy-url <upload-proxy service:443> \
        --insecure --wait-secs=240
    ```
    More info on the upload command can be found [here](https://kubevirt.io/2019/How-To-Import-VM-into-Kubevirt.html). I had to change the access mode of the PVC to ReadWriteOnce, as the default local-path storageclass of K3s doesn't support any other modes at this point. Since we're only using one node this doesn't make a difference, but will be an issue for larger K3s clusters.

7. Use containerd to pull the virtio-container-disk image containing the drivers. If you're using Docker, change the command to pull via docker.

    ```
    ctr image pull kubevirt/virtio-container-disk
    ```

8. Create a YAML file for the VM.

    ```
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: winhd
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 15Gi
      storageClassName: hostpath
    
    ---
    apiVersion: kubevirt.io/v1alpha3
    kind: VirtualMachine
    metadata:
      name: win2k19-iso
    spec:
      running: false
      template:
        metadata:
          labels:
            kubevirt.io/domain: win2k19-iso
        spec:
          domain:
            cpu:
              cores: 4
            devices:
              disks:
              - bootOrder: 1
                cdrom:
                  bus: sata
                name: cdromiso
              - disk:
                  bus: virtio
                name: harddrive
              - cdrom:
                  bus: sata
                name: virtiocontainerdisk
            machine:
              type: q35
            resources:
              requests:
                memory: 8G
          volumes:
          - name: cdromiso
            persistentVolumeClaim:
              claimName: iso-win2k19
          - name: harddrive
            persistentVolumeClaim:
              claimName: winhd
          - containerDisk:
              image: kubevirt/virtio-container-disk
            name: virtiocontainerdisk
    ```

9. Create and use the VM.

    ```
    kubectl apply -f win2k19.yaml
    kubectl virt start win2k19-iso
    # If you're running this on a remote machine, use X-forwarding and
    # apt-get install virt-viewer
    kubectl virt vnc win2k19-iso
    ```

10. Walk through the Windows install to get a running Windows VM.
![MAC screenshot with Windows VM](/assets/images/windows.png "Windows VM")

11. **Note:** If your drivers didn't install successfully and your Network connection isn't working because of that, you need to go into the Device Manager on Windows, click on the OtherDevices and update the driver from your local drive where virtio has stored it's files. In my case it was `E:\`.

12. Final step is to add a service to allow for RDP connections.

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: windows-nodeport
    spec:
      externalTrafficPolicy: Cluster
      ports:
      - name: nodeport
        nodePort: 30000
        port: 27017
        protocol: TCP
        targetPort: 3389
      selector:
        kubevirt.io/domain: win2k19-iso
      type: NodePort
    ```
    Which then allows you to access your Windows VM via RDP on `<NodeIp>:30000`.


### Further reading

1. Adapted to K3s from [https://kubevirt.io/2020/KubeVirt-installing_Microsoft_Windows_from_an_iso.html](https://kubevirt.io/2020/KubeVirt-installing_Microsoft_Windows_from_an_iso.html)
