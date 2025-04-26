Vagrant.configure("2") do |config|
  # Base box
  config.vm.box = "ubuntu/jammy64"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "512"
    vb.cpus = 1
  end

  # Common provisioning (for all VMs)
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y net-tools iproute2 iputils-ping curl ncat
  SHELL

  # Define server VM
  config.vm.define "server" do |server|
    server.vm.hostname = "server"
    server.vm.network "private_network", ip: "192.168.56.10"
    server.vm.provision "shell", inline: <<-SHELL
      echo 'Starting echo server on port 5000...'
      nohup ncat -lk 5000 --keep-open --exec "/bin/cat" &
    SHELL
  end

  # Define proxy VM
  config.vm.define "proxy" do |proxy|
    proxy.vm.hostname = "proxy"
    proxy.vm.network "private_network", ip: "192.168.56.11"
    proxy.vm.provision "shell", inline: <<-SHELL
      apt-get install -y haproxy
      echo 'Enabling IP forwarding...'
      sysctl -w net.ipv4.ip_forward=1

      cat > /etc/haproxy/haproxy.cfg <<EOF
global
    daemon
    maxconn 256

defaults
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend ft_tcp
    bind *:5000
    default_backend be_tcp

backend be_tcp
    server srv1 192.168.56.10:5000
EOF

      systemctl restart haproxy
    SHELL
  end

  # Define client VM
  config.vm.define "client" do |client|
    client.vm.hostname = "client"
    client.vm.network "private_network", ip: "192.168.56.12"
    client.vm.provision "shell", inline: <<-SHELL
      apt-get install -y apache2-utils
      echo 'Client is ready. You can now benchmark.'
    SHELL
  end
end
