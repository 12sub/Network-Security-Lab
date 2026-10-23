Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.hostname = "netsec-lab"

  config.vm.provider "virtualbox" do |vb|
    vb.name   = "netsec-lab"
    vb.memory = 1024
    vb.cpus   = 1

  config.vm.provision "shell", name: "proxy", inline: <<~'SHELL'
    set -euo pipefail
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y nginx python3

    # Backend on port 3000, localhost only
    mkdir -p /srv/demo
    echo "Hello from the backend on port 3000" > /srv/demo/index.html

    cat > /etc/systemd/system/demo-backend.service <<'EOF'
    [Unit]
    Description=Demo backend on port 3000
    After=network.target

    [Service]
    ExecStart=/usr/bin/python3 -m http.server 3000 --bind 127.0.0.1 --directory /srv/demo
    Restart=always
    DynamicUser=yes

    [Install]
    WantedBy=multi-user.target
    EOF

    # Reverse proxy: 8080 -> 3000
    cat > /etc/nginx/conf.d/proxy.conf <<'EOF'
    server {
        listen 8080;
        server_name _;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    EOF

    nginx -t
    systemctl daemon-reload
    systemctl enable --now demo-backend
    systemctl enable --now nginx
    systemctl reload nginx
  SHELL

  pubkey = File.expand_path("~/.ssh/id_ed25519.pub")
  abort "Public key not found at #{pubkey}" unless File.exist?(pubkey)

  config.vm.provision "file", source: pubkey, destination: "/tmp/host_key.pub"

  config.vm.provision "shell", name: "ssh", inline: <<~'SHELL'
    set -euo pipefail
    install -d -m 700 -o vagrant -g vagrant /home/vagrant/.ssh
    touch /home/vagrant/.ssh/authorized_keys
    grep -qxF "$(cat /tmp/host_key.pub)" /home/vagrant/.ssh/authorized_keys \
      || cat /tmp/host_key.pub >> /home/vagrant/.ssh/authorized_keys
    chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
    chmod 600 /home/vagrant/.ssh/authorized_keys

    # Disable password logins (keys only)
    echo "PasswordAuthentication no" > /etc/ssh/sshd_config.d/10-hardening.conf
    sshd -t
    systemctl reload ssh
  SHELL
  config.vm.provision "shell", name: "tailscale",
    env: { "TS_AUTHKEY" => ENV["TS_AUTHKEY"].to_s },
    inline: <<~'SHELL'
      set -euo pipefail
      if [ -z "${TS_AUTHKEY:-}" ]; then
        echo "TS_AUTHKEY is not set; aborting" >&2
        exit 1
      fi
      apt-get install -y curl ca-certificates
      curl -fsSL https://tailscale.com/install.sh | sh
      tailscale up --authkey="$TS_AUTHKEY" --hostname=netsec-lab
      tailscale ip -4
    SHELL
  end

  # Debian boxes often lack working VirtualBox shared folder support
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Bind forwards to localhost only so your LAN can't reach them
  config.vm.network "forwarded_port", guest: 22,   host: 2222, host_ip: "127.0.0.1", id: "ssh"
  config.vm.network "forwarded_port", guest: 8080, host: 8080, host_ip: "127.0.0.1"
end
