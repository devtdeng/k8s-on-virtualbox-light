# Install a k8s cluster with NFSServer

1. Download and [install Vagrant](https://www.vagrantup.com/downloads.html) (Latest Version: 2.2.4)
2. Download and [install VirtualBox](https://www.virtualbox.org/wiki/Downloads)
4. `git clone https://github.com/devtdeng/k8s-on-virtualbox.git`
5. `cd k8s-on-virtualbox-light`
6. `vagrant up`
7. `vagrant global-status` returns the state of all 3 vms
```
id       name         provider   state   directory
-----------------------------------------------------------------------------------
59697f8  k8s-master   virtualbox running /Users/tdeng/work/github-devtdeng/k8s-on-virtualbox/automatic 
64281c6  k8s-worker-1 virtualbox running /Users/tdeng/work/github-devtdeng/k8s-on-virtualbox/automatic 
62a2cc7  k8s-worker-2 virtualbox running /Users/tdeng/work/github-devtdeng/k8s-on-virtualbox/automatic 
```
8. `vagrant ssh k8s-master`, copy /etc/kubernetes/admin.conf from k8s-master VM to host ~/.kube/config
9. `kubectl get componentstatus` on host or `k8s-master` VM
```
vagrant@k8s-master:~$ kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```
10. If more than two nodes are required, you can edit the servers array in the /Users/mnagle/workspace/install-k8s-macos/Vagrantfile
```servers = [
   {
       :name => "worker-3",
       :type => "node",
       :box => "ubuntu/xenial64",
       :box_version => "20180831.0.0",
       :eth1 => "192.168.205.13",
       :mem => "2048",
       :cpu => "2"
   }
]
```
11. You can power down the vm's by running `vagrant suspend -a` (State goes from running to saved) and run vagrant up to bring them back up again (commands need to be run from the directory with Vagrantfile).
12. You can remove the k8s cluster by running `vagrant destroy -f`