ENV["VAGRANT_DEFAULT_PROVIDER"] = "vmware_desktop"

Vagrant.require_version ">= 2.3.0"

BOX_IMAGE = "bento/ubuntu-22.04"
ETCD_NODE_COUNT = 3

Vagrant.configure("2") do |config|
  config.vm.box = BOX_IMAGE

  # Shared folder disabled globally.
  # We want each VM to be self-contained unless explicitly configured otherwise.
  config.vm.synced_folder ".", "/vagrant", disabled: true

  #
  # etcd nodes
  #
  (1..ETCD_NODE_COUNT).each do |i|
    config.vm.define "etcd#{i}" do |node|
      node.vm.hostname = "etcd#{i}"

      node.vm.provider "vmware_desktop" do |v|
        v.cpus = 1
        v.memory = 1024
      end
    end
  end

  #
  # Observability node: Prometheus + Grafana
  #
  config.vm.define "obs" do |node|
    node.vm.hostname = "obs"

    # Prometheus UI
    node.vm.network "forwarded_port",
      guest: 9090,
      host: 9090,
      host_ip: "127.0.0.1",
      auto_correct: true

    # Grafana UI
    node.vm.network "forwarded_port",
      guest: 3000,
      host: 3000,
      host_ip: "127.0.0.1",
      auto_correct: true

    node.vm.provider "vmware_desktop" do |v|
      v.cpus = 2
      v.memory = 2048
    end
  end
end