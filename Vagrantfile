# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']

MACHINES = {
    :inetrouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "router-net"},
               ]
            },
    :inetrouter2 => {
        :box_name => "centos/7",
        :net => [
                  {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "router-net"},
                ]
            },
    :centralrouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.3', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "router-net"},
                   {ip: '192.168.0.1', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: "dir-net"},
               
                ]
            },
    :centralserver => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.0.2', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "dir-net"},
                ]
            }
        }

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig| 
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s

      config.vm.provider "virtualbox" do |v|
        v.memory = 256
        v.cpus = 1
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", ipconf
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
      
      case boxname.to_s
        when "inetrouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
              yum install -y iptables-services
              systemctl enable iptables 
              systemctl start iptables
              iptables -F
              
              echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf
              
              iptables-restore < /vagrant/inetrouter_iptables.rules
              iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
              echo "192.168.0.0/24 via 192.168.255.3 dev eth1" >> /etc/sysconfig/network-scripts/route-eth1
              sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config

              systemctl restart sshd
              service iptables save
              systemctl restart network
              reboot
          SHELL
        
      when "inetrouter2"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
              yum install -y iptables-services
              systemctl enable iptables 
              systemctl start iptables
              iptables -F
      
              echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf

              echo "192.168.0.0/24 via 192.168.255.3 dev eth1" > /etc/sysconfig/network-scripts/route-eth1
              echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1

              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0

              EXT_IP="10.0.2.15" # ?????????????? IP-?????????? ??????????.
              INT_IP="192.168.255.2" # ?????????????????? IP-?????????? ??????????.
              PORT=8080  # ???????? ?????????? ?????????????? ?????????? ???????????????? ???????????????????? ????????????.
              LAN_IP="192.168.0.2" # ???????????????????? ?????????? ??????????????.
              SRV_PORT=80 # ???????? ?????? ?????????????????????? ?? ?????????????????????? ??????????????.

              iptables -t nat -A PREROUTING -d $EXT_IP -p tcp -m tcp --dport $PORT -j DNAT --to-destination $LAN_IP:$SRV_PORT
              iptables -t nat -A POSTROUTING -d $LAN_IP -p tcp -m tcp --dport $SRV_PORT -j SNAT --to-source $INT_IP
              iptables -t nat -A OUTPUT -d $EXT_IP -p tcp -m tcp --dport $PORT -j DNAT --to-destination $LAN_IP:$SRV_PORT

              iptables -t nat -A PREROUTING -d $INT_IP -p tcp -m tcp --dport $PORT -j DNAT --to-destination $LAN_IP:$SRV_PORT
              iptables -t nat -A POSTROUTING -d $LAN_IP -p tcp -m tcp --dport $SRV_PORT -j SNAT --to-source $INT_IP
              iptables -t nat -A OUTPUT -d $INT_IP -p tcp -m tcp --dport $PORT -j DNAT --to-destination $LAN_IP:$SRV_PORT
              
              service iptables save
              systemctl restart network
              reboot
          SHELL

      when "centralrouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
              echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf
              echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
              sysctl -p /etc/sysctl.conf
              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
              echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1

              systemctl restart network
              setenforce 0

              yum install nmap -y
              reboot
          SHELL

      when "centralserver"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
              echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1

              systemctl restart network
              yum install -y epel-release
              yum install -y nginx
              yum install -y traceroute

              systemctl enable nginx
              systemctl start nginx
              systemctl restart network
              
              setenforce 0
              reboot
          SHELL
      end
    end
  end
end