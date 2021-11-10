# -*- mode: ruby -*-
# vi: set ft=ruby :
default_box = "opensuse/Leap-15.2.x86_64"
box_version = "15.2.31.584"
# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  # set up root access
  config.ssh.username = 'root'
  config.ssh.password = 'vagrant'
  config.ssh.insert_key = 'true'
  config.vm.define "dashboard" do |dashboard|
    dashboard.vm.box = default_box
    dashboard.vm.box_version
    dashboard.vm.hostname = "dashboard"
    dashboard.vm.network 'private_network', ip: "192.168.33.10",  virtualbox__intnet: true
    dashboard.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", disabled: true
    dashboard.vm.network "forwarded_port", guest: 22, host: 2000 # Master Node SSH
    dashboard.vm.network "forwarded_port", guest: 6443, host: 6443 # Kubectl API Access
    dashboard.vm.network "forwarded_port", guest: 8080, host: 8080 # API Access
    dashboard.vm.network "forwarded_port", guest: 16686, host: 16686 # Jaeger HTTP Access
    dashboard.vm.network "forwarded_port", guest: 16687, host: 16687 # Jaeger CR Access
    for p in 30000..32767 # expose NodePort IP's
      dashboard.vm.network "forwarded_port", guest: p, host: p, protocol: "tcp"
    end
    dashboard.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = "4096"
      vb.name = "dashboard"
      # https://stackoverflow.com/a/17126363
      vb.customize ["modifyvm", :id, "--ioapic", "on"]
    end
    dashboard.vm.provision "shell", inline: <<-SHELL
      echo "******** Step 1: Installing dependencies ********"
      sudo zypper refresh
      sudo zypper --non-interactive install bzip2
      sudo zypper --non-interactive install etcd
      sudo zypper --non-interactive install lsof
      sudo zypper --non-interactive install htop
      sudo zypper --non-interactive install net-tools
      sudo zypper --non-interactive install wget
      sudo zypper --non-interactive install apparmor-parser
      sudo zypper --non-interactive install k9s
      sudo zypper --non-interactive install bind-utils
      echo -e "******** Begin installing k3s ********\n"
      curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
      mkdir -p /home/vagrant/.kube
      /usr/local/bin/kubectl config view --raw >/home/vagrant/.kube/config
      echo -e "******** End installing k3s ********\n\n"
      echo -e "******** Step 2: Setting up Observability ********\n\n"
      echo -e "******** Begin installing Helm ********\n"
      export PATH=$PATH:/usr/loca/bin
      curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
      chmod 700 get_helm.sh
      ./get_helm.sh
      echo -e "******** End installing Helm ********\n\n"
      echo -e "******** Begin installing Grafana and Prometheus ********\n"
      /usr/local/bin/kubectl create namespace monitoring
      /usr/local/bin/helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
      /usr/local/bin/helm repo add stable https://charts.helm.sh/stable
      /usr/local/bin/helm repo update
      /usr/local/bin/helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --kubeconfig /etc/rancher/k3s/k3s.yaml
      echo -e "******** End installing Grafana and Prometheus ********\n\n"
      echo -e "******** Begin installing Jaeger ********\n"
      /usr/local/bin/kubectl create namespace observability
      /usr/local/bin/kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
      /usr/local/bin/kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
      /usr/local/bin/kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
      /usr/local/bin/kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
      /usr/local/bin/kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
      echo -e "******** End installing Jaeger ********\n\n"
      echo -e "******** Begin configuring Cluster wide Jaeger ********\n"
      /usr/local/bin/kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
      /usr/local/bin/kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml
      echo -e "******** End configuring Cluster wide Jaeger ********\n\n"
      echo "******** Verify helm is installed ********"
      /usr/local/bin/helm ls --kubeconfig /etc/rancher/k3s/k3s.yaml --namespace=monitoring
      echo "******** Verify prometheus is installed ********"
      /usr/local/bin/kubectl get pods --namespace=monitoring
      echo "******** Verify jaeger is running ********"
      /usr/local/bin/kubectl get pods --namespace=observability
    SHELL
  end
end