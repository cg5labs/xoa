# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.network "forwarded_port", guest: 443, host: 8443

  config.vm.provider :libvirt do |libvirt|
    libvirt.cpu_model = "qemu64"
    libvirt.cpus = 2
    libvirt.memory = 8192
  end

  config.trigger.before :up, :provision do |trigger|
    trigger.name = "Create self-signed nginx TLS certificate"
    trigger.run = {
      inline: <<~SHELL
        /bin/bash -ec '
          mkdir -p ansible/roles/nginx/files
          openssl genrsa -out ansible/roles/nginx/files/nginx.key 2048
          openssl req -new -key ansible/roles/nginx/files/nginx.key -out ansible/roles/nginx/files/nginx.csr -subj "/CN=localhost"
          openssl x509 -req -days 3650 -in ansible/roles/nginx/files/nginx.csr -signkey ansible/roles/nginx/files/nginx.key -out ansible/roles/nginx/files/nginx.crt
        '
      SHELL
    }
  end

  config.vm.provision "shell", inline: "mkdir -p /data"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbooks/xoa.yaml"
    ansible.raw_arguments = [
      "-e", "xo_cli_email=abc@def.com",
      "-e", "xo_cli_password=secret"
    ]
    # Optional: Fix the Vagrant warning by telling it which compatibility mode to expect
    ansible.compatibility_mode = "2.0" 
  end

end
