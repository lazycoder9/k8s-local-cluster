# frozen_string_literal: true

BASE_CONFIG = {
  box: 'ubuntu/xenial64',
  box_version: '20180831.0.0',
  mem: '2048',
  cpu: '2'
}.freeze

SERVERS = [
  { name: 'k8s-head', type: 'master', eth1: '192.168.205.10' },
  { name: 'k8s-node-1', type: 'node', eth1: '192.168.205.11' },
  { name: 'k8s-node-2', type: 'node', eth1: '192.168.205.12' }
].map { |config| config.merge(BASE_CONFIG) }

configure_box = <<~SCRIPT
  # install kubeadm
  apt-get install -y apt-transport-https curl
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
  EOF
  apt-get update
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl

  # kubelet requires swap off
  swapoff -a

  # keep swap off after reboot
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  # ip of this box
  IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
  # set node-ip
  sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
  sudo systemctl restart kubelet
SCRIPT

configure_master = <<-SCRIPT
  # ip of this box
  IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`

  # install k8s master
  HOST_NAME=$(hostname -s)
  kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

  #copying credentials to regular user - vagrant
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # install Calico pod network addon
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml

  kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # required for setting up password less ssh between guest VMs
  sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  sudo service sshd restart
SCRIPT

configure_node = <<-SCRIPT
  apt-get install -y sshpass
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
  sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure('2') do |vm_config|
  SERVERS.each do |opts|
    vm_config.vm.define opts[:name] do |config|
      config.vm.box = opts[:box]
      config.vm.box_version = opts[:box_version]
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:eth1]

      config.vm.provider 'virtualbox' do |v|
        v.name = opts[:name]
        v.customize ['modifyvm', :id, '--memory', opts[:mem]]
        v.customize ['modifyvm', :id, '--cpus', opts[:cpu]]
      end

      config.vm.provision 'docker'
      config.vm.provision 'shell', inline: configure_box

      conf_script = opts[:type] == 'master' ? configure_master : configure_node
      config.vm.provision 'shell', inline: conf_script
    end
  end
end

