# OAIC RIC All-in-One Install Script
#### This README file is also the script that does all of the things.  You can run it with this command: -
#### curl -L https://raw.githubusercontent.com/philrod1/oaic-ric-installer/master/README.md | bash
#### Alternatively, you can click on the 🖉 symbol in Github and copy the raw markdown.
#### You could also run each section by using the copy option

## Start with a fresh install of Ubuntu 20.04
### Tested using ubuntu-20.04.1-legacy-server-amd64.iso image
#### Hypervisor details: KVM, qemu-system-x86_64, Q35, BIOS
#### Guest system: 4 cores, 16G RAM, 150G storage, default networking (NAT)
#### Host system: Ubuntu 23.10, Kernel 6.5.0, Intel i7-1370P, 64G RAM


## Setup Useful Aliases

    message () { echo -e "\e[1;93m$1\e[0m"; }
    message "Modifying .bashrc"
    export myip=`hostname  -I | cut -f1 -d' '`
    echo 'alias pods="kubectl get pods -A"' >> ~/.bashrc
    echo 'alias srv="kubectl get services -A"' >> ~/.bashrc
    echo 'alias flushpods="kubectl delete pods -A --field-selector=\"status.phase==Failed\""' >> ~/.bashrc
    echo "export myip=`hostname  -I | cut -f1 -d' '`" >> ~/.bashrc
    echo 'export KUBECONFIG="${HOME}/.kube/config"' >> ~/.bashrc
    echo 'export HELM_HOME="${HOME}/.helm"' >> ~/.bashrc


## Refresh apt

    message "Refresh apt"
    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y openssh-server nfs-common


## Clone OAIC repo and install

    message "Clone OAIC repo"
    git clone https://github.com/openaicellular/oaic.git
    cd oaic/
    git submodule update --init --recursive --remote


## Deploy kube-system Pods

    message "Deploy kube-system pods"
    cd RIC-Deployment/tools/k8s/bin
    ./gen-cloud-init.sh
    sudo ./k8s-1node-cloud-init*.sh


## Configure 'docker' and 'kubectl' For Non-root User 

    message "Enabling docker and kubectl as standard user"
    sudo usermod -aG docker $USER
    mkdir -p ~/.kube
    sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
    sudo chown -R $USER:$USER ~/.kube
    chmod o+rx ~/.kube/config


## Configure Helm

    message "Setup Helm"
    cd ~
    mkdir -p ~/.helm
    helm init --upgrade
    helm install stable/nfs-server-provisioner --namespace ricinfra --name nfs-release-1
    kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'


## Configure ChartMuseum

    message "Configure ChartMuseum"
    mkdir charts
    docker kill chartmuseum
    docker run --rm -u 0 -it -d --name chartmuseum -p 8090:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest


## Build Modified E2 Termination Pod

    message "Deploying E2 Termination"
    docker run -v /registry-storage:$HOME/registry -d -p 5001:5000 --restart=always --name ric registry:2
    cd ~/oaic/ric-plt-e2/RIC-E2-TERMINATION
    docker build -f Dockerfile -t localhost:5001/ric-plt-e2:5.5.0 .
    docker push localhost:5001/ric-plt-e2:5.5.0


## Deploy the Base RIC Components
#### This will create the 'ricinfra' and 'ricplt' namespaces and deploy all the
#### main RIC components from the O-RAN alliance 'e' release

    message "Deploying the RIC"
    cd ~/oaic/RIC-Deployment/bin
    sed -i 's/ricip: "[^"]*"/ricip: "$myip"/g' ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified.yaml
    sed -i 's/auxip: "[^"]*"/ricip: "$myip"/g' ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified.yaml
    . ./deploy-ric-platform ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_e_release_modified_e2.yaml
    message "DONE!"
    message "Run 'su - $USER' or re-login to finish up."
    message "After that, you can type 'pods' to check the status of the containers."

#### That's it for now.  Just re-login and wait for the pods to start.
#### The SRS UE, ENb and EPC components will be added later. 