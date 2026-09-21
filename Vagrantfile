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
          mkdir -p roles/nginx/files
          openssl genrsa -out roles/nginx/files/nginx.key 2048
          openssl req -new -key roles/nginx/files/nginx.key -out roles/nginx/files/nginx.csr -subj "/CN=localhost"
          openssl x509 -req -days 3650 -in roles/nginx/files/nginx.csr -signkey roles/nginx/files/nginx.key -out roles/nginx/files/nginx.crt
        '
      SHELL
    }
  end

  # vagrant-libvirt implements forwarded ports as host-side SSH tunnels.
  # The guest reboot during provisioning can drop that tunnel, so restore it after `up`.
  config.trigger.after :up do |trigger|
    trigger.name = "Restore HTTPS port forward"
    trigger.run = {
      inline: <<~SHELL
        /bin/bash -ec '
          if ! vagrant status --machine-readable default | awk -F, '"'"'$3 == "provider-name" && $4 == "libvirt" { found = 1 } END { exit !found }'"'"'; then
            exit 0
          fi

          pid_file=.vagrant/machines/default/libvirt/pids/ssh_8443.pid
          if [ -s "$pid_file" ] && kill -0 "$(cat "$pid_file")" 2>/dev/null; then
            exit 0
          fi

          mkdir -p "$(dirname "$pid_file")"
          rm -f "$pid_file"

          host=$(vagrant ssh-config default | awk "/HostName / { print \\$2 }")
          user=$(vagrant ssh-config default | awk "/User / { print \\$2 }")
          key=$(vagrant ssh-config default | awk "/IdentityFile / { print \\$2 }" | tr -d "\\"")

          ssh -f -N \
            -o ExitOnForwardFailure=yes \
            -o StrictHostKeyChecking=no \
            -o UserKnownHostsFile=/dev/null \
            -i "$key" \
            -L 127.0.0.1:8443:127.0.0.1:443 \
            "$user@$host"

          pgrep -f "127.0.0.1:8443:127.0.0.1:443" | tail -n 1 > "$pid_file"
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
