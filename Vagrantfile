# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.205.10",
        :mem => "4096",
        :cpu => "2"
    },
    {
        :name => "k8s-node1",
        :type => "node",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.205.11",
        :mem => "4096",
        :cpu => "2",
        :nfs => "true"
    },
    {
        :name => "k8s-node2",
        :type => "node",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.205.12",
        :mem => "4096",
        :cpu => "2"
    },
# Uncomment section below to enable a 3rd worker node.
#    {
#        :name => "k8s-node3",
#        :type => "node",
#        :box => "ubuntu/bionic64",
#        :eth1 => "192.168.205.13",
#        :mem => "4096",
#        :cpu => "2"
#    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    # sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    # sudo add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    sudo apt-get update && sudo apt-get install -y ntpdate neofetch socat apt-transport-https ca-certificates curl software-properties-common nfs-common

    # Install Docker - Specify manual version, known validated, until auto-logic added
    export VERSION=18.09 && curl -sSL get.docker.com | sh

    # Setup daemon.
    sudo bash -c 'cat <<EOF> /etc/docker/daemon.json 
    {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "storage-driver": "overlay2"
    }
EOF'

    sudo mkdir -p /etc/systemd/system/docker.service.d

    # Restart docker.
    sudo systemctl daemon-reload
    sudo systemctl restart docker

    # run docker commands as vagrant user (sudo not required)
    sudo usermod -aG docker vagrant

    # Set time to ensure IdP works
    sudo ntpdate pool.ntp.org 
    
    # install kubeadm
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF'
    # sudo apt update && apt install -y kubeadm=1.14\* kubectl=1.14\* kubelet=1.14\* kubernetes-cni=0.7\*
    sudo apt update && sudo apt install -y kubeadm kubectl kubelet kubernetes-cni

    # sudo snap install --classic kubelet
    # sudo snap install --classic kubeadm
    # sudo snap install --classic kubectl

    # kubelet requires swap off
    sudo swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep mask | awk '{print $2}'| cut -f2 -d:`
    # set node-ip
    sudo echo "KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" >> /etc/default/kubelet
    echo "source <(kubectl completion bash)" >> /home/vagrant/.bashrc
    echo "echo" >> /home/vagrant/.bashrc
    echo "echo" >> /home/vagrant/.bashrc
    echo "/usr/bin/neofetch" >> /home/vagrant/.bashrc
    echo "echo" >> /home/vagrant/.bashrc
    echo "echo" >> /home/vagrant/.bashrc

    # Update apt sources to avoid 404s, do so non-interactively to deal with libssl
    # 
    echo "libssl1.1 libssl1.1/restart-services boolean true" | sudo debconf-set-selections
    sudo DEBIAN_FRONTEND=noninteractive apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -yq
    # sudo DEBIAN_FRONTEND=noninteractive sudo dpkg-reconfigure -a
SCRIPT

$configureMaster = <<-SCRIPT
    echo "##################################################"
    echo "Configuring as master node"
    echo "##################################################"
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep mask | awk '{print $2}'| cut -f2 -d:`

    # install k8s master node
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16 --ignore-preflight-errors=SystemVerification

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    # install Calico pod network addon
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/calico/rbac-kdd.yaml
    
    # Install CNI Calico (deprecated?).
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/calico/calico.yaml
    
    # Install CNI Flannel
    # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart
SCRIPT

$configureNode = <<-SCRIPT
    echo "This is a worker node"
    sudo apt-get install -y sshpass
    sudo apt update && sudo apt upgrade -y
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
    sudo sh ./kubeadm_join_cmd.sh
SCRIPT

$configureNFS = <<-SCRIPT
    echo "##################################################"
    echo "Configuring NFS Provisioner Pre-Work"
    echo "##################################################"
    sudo mkdir -p /storage/dynamic
    sudo mkdir -p /export
SCRIPT

#
# This part can be automated, but has been added as manual commands in the wiki so team can be exposed to commands and steps.
#
$configureK8s = <<-SCRIPT
    echo "We're running a series of kubectl commands to setup our private cloud..."
    
    # Set the default service account as an admin
    kubectl create clusterrolebinding default-admin --clusterrole cluster-admin --serviceaccount=default:default

    # Pull and apply the MetalLB load-balancer service, hardcoded to serve private IPs in our host-only network
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/metallb/metallb.yaml
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/metallb/layer-2.yaml

    # Label the node that will host NFS pvs
    kubectl label nodes k8s-node1 role=nfs

    # Pull and apply the nfs-provisioner
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/nfs-provisioner/nfs-deployment.yaml
    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/nfs-provisioner/nfs-class.yaml

    # Define the new storage class as default
    kubectl patch storageclass nfs-dynamic -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

    sudo snap install helm --classic

    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/qseok/rbac-config.yaml

    helm init --service-account tiller

    helm repo add qlik https://qlik.bintray.com/stable
    helm repo add qlik-edge https://qlik.bintray.com/edge

    kubectl apply -f https://raw.githubusercontent.com/jprdonnelly/kubernetes-cluster/master/qseok/nfs-vol-pvc.yaml

SCRIPT

Vagrant.configure("2") do |config|

    required_plugins = %w( vagrant-vbguest vagrant-disksize )
	_retry = false
	required_plugins.each do |plugin|
		unless Vagrant.has_plugin? plugin
			system "vagrant plugin install #{plugin}"
			_retry=true
		end
	end

	if (_retry)
		exec "vagrant " + ARGV.join(' ')
    end
    
    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	v.customize ["modifyvm", :id, "--groups", "/QSEoK"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
                            else
                config.vm.provision "shell", inline: $configureNode
            end
            if opts[:nfs] == "true"
                config.disksize.size = "40GB"
                config.vm.provision "shell", inline: $configureNFS
            end
        end
    end  
end
