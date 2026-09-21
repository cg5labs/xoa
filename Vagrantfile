# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  nginx_tls_dir = "roles/nginx/files"
  nginx_tls_key = "#{nginx_tls_dir}/nginx.key"
  nginx_tls_csr = "#{nginx_tls_dir}/nginx.csr"
  nginx_tls_cert = "#{nginx_tls_dir}/nginx.crt"
  nginx_tls_subject = "/CN=localhost"
  nginx_tls_days = 3650
  nginx_tls_key_bits = 2048

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
          tls_dir="#{nginx_tls_dir}"
          tls_key="#{nginx_tls_key}"
          tls_csr="#{nginx_tls_csr}"
          tls_cert="#{nginx_tls_cert}"
          tls_subject="#{nginx_tls_subject}"
          tls_days="#{nginx_tls_days}"
          tls_key_bits="#{nginx_tls_key_bits}"

          if [ -s "$tls_key" ] && [ -s "$tls_cert" ]; then
            exit 0
          fi

          mkdir -p "$tls_dir"
          openssl genrsa -out "$tls_key" "$tls_key_bits"
          openssl req -new -key "$tls_key" -out "$tls_csr" -subj "$tls_subject"
          openssl x509 -req -days "$tls_days" -in "$tls_csr" -signkey "$tls_key" -out "$tls_cert"
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
