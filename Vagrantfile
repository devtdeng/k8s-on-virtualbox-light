servers = [
    {
        :name => "k8s-master",
        :type => "master",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :eth1 => "192.168.205.10",
        :mem => "1024",
        :cpu => "2",
    },
    {
        :name => "k8s-worker-1",
        :type => "worker",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :eth1 => "192.168.205.11",
        :mem => "1024",
        :cpu => "2",
    },
    {
        :name => "k8s-worker-2",
        :type => "worker",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :eth1 => "192.168.205.12",
        :mem => "1024",
        :cpu => "2",
    },
    {
        :name => "nfs-server",
        :type => "nfs",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :eth1 => "192.168.205.15",
        :mem => "256",
        :cpu => "1",
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    # install docker v17.03
    # reason for not using docker provision is that it always installs latest version of the docker, but kubeadm requires 17.03 or older
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')

    # run docker commands as vagrant user (not required)
    usermod -aG docker vagrant

    # install kubeadm
    apt-get update && apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo 'deb https://apt.kubernetes.io/ kubernetes-xenial main' >> /etc/apt/sources.list.d/kubernetes.list

    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl

    # kubelet requires swap off
    swapoff -a

    # keep swap off after reboot
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

SCRIPT

$configureMaster = <<-SCRIPT
    echo "This is master"
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`

    # install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

    # set the internal ip for k8s-master to same ip as $IP_ADDR
    sed -i 's/"/"--node-ip='"$IP_ADDR"' /' /var/lib/kubelet/kubeadm-flags.env
    systemctl restart kubelet

    #copying credentials to regular user - vagrant
    --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    # install Calico pod network addon
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    # required for setting up password less ssh between guest VMs
    sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    service sshd restart

SCRIPT

$configureWorker = <<-SCRIPT
    echo "This is worker"
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh

    # set correct internal ip for the k8s node (kubelet)
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    sed -i 's/"/"--node-ip='"$IP_ADDR"' /' /var/lib/kubelet/kubeadm-flags.env
    systemctl restart kubelet
SCRIPT

$configureNFSServer = <<-SCRIPT
    set -euo pipefail

    apt-get install -y nfs-kernel-server
    mkdir -p /mnt/k8s
    mkdir -p /mnt/k8s/data
    chown nobody:nogroup /mnt/k8s
    chmod 777 /mnt/k8s

    # /etc/exports 
    echo '/mnt/k8s 192.168.205.0/24(rw,sync,no_subtree_check,insecure,no_root_squash)' >> /etc/exports
    exportfs -a
    systemctl restart nfs-kernel-server    
    exportfs -v
SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|
            config.ssh.password = opts[:ssh_password]

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]


            config.vm.provider "virtualbox" do |v|
                v.name = opts[:name]
            	 v.customize ["modifyvm", :id, "--groups", "/Ballerina Development"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
            end

            # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
            #config.vm.provision "docker"


            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureBox
                config.vm.provision "shell", inline: $configureMaster
            elsif opts[:type] == "worker"
                config.vm.provision "shell", inline: $configureBox
                config.vm.provision "shell", inline: $configureWorker
            else
                config.vm.provision "shell", inline: $configureNFSServer
            end
        end
    end
end