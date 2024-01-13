# -*- mode: ruby -*-
# vim: set ft=ruby :


MACHINES = {
  :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        :ip_addr => '192.168.56.101',
        :cpus => 4,
        #:provision => "test.sh",       
  },
}


Vagrant.configure("2") do |config|


  MACHINES.each do |boxname, boxconfig|


      config.vm.define boxname do |box|


        box.vm.box = boxconfig[:box_name]
        box.vm.box_version = boxconfig[:box_version]


        box.vm.host_name = "selinux"
        box.vm.network "private_network", ip: boxconfig[:ip_addr]
        box.vm.network "forwarded_port", guest: 4881, host: 4881


        box.vm.provider :virtualbox do |vb|
              vb.customize ["modifyvm", :id, "--memory", "4096"]
              needsController = false
        end


        box.vm.provision "shell", inline: <<-SHELL
          sudo yum install dnf-plugins-core
	  sudo yum config-manager --set-enabled PowerTools
	  sudo yum install policycoreutils-python
	  sudo yum install -y nano
          #install epel-release
          rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY*
          yum install -y epel-release
          #install nginx
          yum install -y nginx
          #change nginx port
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
          #disable SELinux
          #setenforce 0
          #start nginx
          systemctl start nginx
          systemctl status nginx
          #check nginx port
          ss -tlpn | grep 4881
        SHELL
    end
  end
end

