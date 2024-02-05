#!/bin/bash
## On a fresh install of Ubuntu 20.04 ...
### Tested using ubuntu-20.04.1-legacy-server-amd64.iso image
#### Hypervisor details: KVM, qemu-system-x86_64, Q35, BIOS
#### Guest system: 4 cores, 16G RAM, 150G storage, default networking (NAT)
#### Host system: Ubuntu 23.10, Kernel 6.5.0, Intel i7-1370P, 64G RAM

    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y openssh-server


## Clone OAIC repo and install
### The scripts will pull install any packages required at this stage
### This will install the requirements, such as Docker, Helm, k8s
### It will also create the 'kube-system' namespace and run the containers

    git clone https://github.com/openaicellular/oaic.git
    cd oaic/
    git submodule update --init --recursive --remote
    cd RIC-Deployment/tools/k8s/bin
    ./gen-cloud-init.sh
    sudo ./k8s-1node-cloud-init*.sh


## Creature Comforts
### Set up some useful shortcuts and enable docker and kubectl commands to be run as user
### 'pods' will list all running pods
### 'srv' will list services.  Useful for getting the IP and ports of pods
### 'flushpods' will delete all pods that have failed or been evicted
### 'myip' is a handy shortcut to get the IP address of the 'host' OS

    echo 'alias pods="kubectl get pods -A"' >> ~/.bashrc
    echo 'alias srv="kubectl get services -A"' >> ~/.bashrc
    echo 'alias flushpods="kubectl delete pods -A --field-selector=\"status.phase==Failed\""' >> ~/.bashrc
    echo "alias myip=\"echo \`hostname  -I | cut -f1 -d' '\`\"" >> ~/.bashrc
    echo 'export KUBECONFIG="${HOME}/.kube/config"' >> ~/.bashrc
    echo 'export HELM_HOME="${HOME}/.helm"' >> ~/.bashrc
    source ~/.bashrc
    sudo usermod -aG docker $USER
    newgrp docker
    mkdir -p ~/.kube
    sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
    sudo chown -R $USER:$USER ~/.kube
    chmod o+rx ~/.kube/config


## Deploy the Base RIC Components
### This will create the 'ricinfra' and 'ricplt' namespaces and deploy all the
### main RIC components from the O-RAN alliance 'e' release

    cd ~
    mkdir -p ~/.helm
    helm init --upgrade
    mkdir charts
    docker kill chartmuseum
    docker run --rm -u 0 -it -d --name chartmuseum -p 8090:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest
    cd ~/oaic/RIC-Deployment/bin
    sed -i 's/ricip: "[^"]*"/ricip: "$myip"/g' ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified.yaml
    sed -i 's/auxip: "[^"]*"/ricip: "$myip"/g' ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified.yaml
    . ./deploy-ric-platform ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified_e2.yaml

