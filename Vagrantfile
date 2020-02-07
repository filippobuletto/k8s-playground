# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "ubuntu/xenial64",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "ubuntu/xenial64",
        :eth1 => "192.168.205.11",
        :mem => "4096",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "ubuntu/xenial64",
        :eth1 => "192.168.205.12",
        :mem => "4096",
        :cpu => "2"
    }
]

$configureMaster = <<-SCRIPT
#!/bin/bash
#!/bin/bash -x

echo "Disable swap until next reboot"
echo
sudo swapoff -a
# keep swap off after reboot
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "Update the local node"
sudo apt-get update && sudo apt-get upgrade -y
echo
echo "Install Docker"
sleep 3

sudo apt-get install -y docker.io
echo
echo "Install kubeadm and kubectl"
sleep 3

sudo sh -c "echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' >> /etc/apt/sources.list.d/kubernetes.list"

sudo sh -c "curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -"

sudo apt-get update

sudo apt-get install -y kubeadm=1.17.2-00 kubelet=1.17.2-00 kubectl=1.17.2-00

sudo apt-mark hold kubelet kubeadm kubectl

echo
echo "Installed - now to get Calico Project network plugin"

sleep 3

export CALICO_IPV4POOL_CIDR=172.16.0.0/16

sudo kubeadm init --kubernetes-version 1.17.2 \
   --pod-network-cidr $CALICO_IPV4POOL_CIDR \
   --apiserver-advertise-address=192.168.205.10 \
   --token ktorbm.2xrevimbq274bwg6 \
   --token-ttl 0 | tee /vagrant/kubeadm-init.log

sleep 5

echo "Running the steps explained at the end of the init output for you"

mkdir -p /home/vagrant/.kube

sleep 2

sudo cp -f -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
sudo cp -f -i /etc/kubernetes/admin.conf /vagrant/

sleep 2

sudo chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

echo "Download Calico plugin and RBAC YAML files and apply"

wget https://docs.projectcalico.org/manifests/calico.yaml -O calico.yaml
sed -i -e "s?192.168.0.0/16?$CALICO_IPV4POOL_CIDR?g" calico.yaml
kubectl apply -f calico.yaml --kubeconfig /etc/kubernetes/admin.conf

echo
echo
sleep 3
echo "You should see this node in the output below"
echo "It can take up to a mintue for node to show Ready status"
echo
kubectl get node --kubeconfig /etc/kubernetes/admin.conf
echo
echo
echo "Script finished. Move to the next step"

SCRIPT

$configureNode = <<-SCRIPT
#!/bin/bash
#!/bin/bash -x

echo "  Disable swap until next reboot"
echo
sudo swapoff -a
# keep swap off after reboot
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "  Update the local node"
sleep 2
sudo apt-get update && sudo apt-get upgrade -y
echo
sleep 2

echo "  Install Docker"
sleep 3
sudo apt-get install -y docker.io

echo
echo "  Install kubeadm and kubectl"
sleep 2
sudo sh -c "echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' >> /etc/apt/sources.list.d/kubernetes.list"

sudo sh -c "curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -"

sudo apt-get update

sudo apt-get install -y kubeadm=1.17.2-00 kubelet=1.17.2-00 kubectl=1.17.2-00

sudo apt-mark hold kubelet kubeadm kubectl

sudo kubeadm join --token ktorbm.2xrevimbq274bwg6 --discovery-token-unsafe-skip-ca-verification 192.168.205.10:6443 | tee -a /vagrant/kubeadm-join.log

echo
echo "  Script finished."
echo

SCRIPT

$configureInternalIp = <<-SCRIPT

echo "  Setting node IP to $PRIVATE_IP as per https://github.com/kubernetes/kubeadm/issues/203"

echo "KUBELET_EXTRA_ARGS=--node-ip=$PRIVATE_IP" | sudo tee /etc/default/kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

SCRIPT

$disableFirewall = <<-SCRIPT

echo "  Disabling firewall"
sudo ufw disable

SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	v.customize ["modifyvm", :id, "--groups", "/Kubernetes Development"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $disableFirewall
            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end
            config.vm.provision "shell", inline: $configureInternalIp, env: {"PRIVATE_IP" => opts[:eth1]}
        end

    end

end