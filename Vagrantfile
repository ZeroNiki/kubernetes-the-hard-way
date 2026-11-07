Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  nodes = [
    { name: "jumpbox",   ip: "192.168.56.9",  memory: 1536, cpus: 1 },
    { name: "manager01", ip: "192.168.56.10", memory: 4096, cpus: 2 },
    { name: "worker01",  ip: "192.168.56.11", memory: 2048, cpus: 2 },
    { name: "worker02",  ip: "192.168.56.12", memory: 2048, cpus: 2 }
  ]

  nodes.each do |node|
    config.vm.define node[:name] do |vm|
      vm.vm.hostname = node[:name]
      # Dedicated libvirt network so the lab does not clash with other environments
      vm.vm.network "private_network",
        ip: node[:ip],
        libvirt__network_name: "k8s-hard-way",
        libvirt__dhcp_enabled: false
      vm.vm.synced_folder ".", "/vagrant", disabled: true

      vm.vm.provider "libvirt" do |lv|
        lv.memory = node[:memory]
        lv.cpus = node[:cpus]
        lv.cpu_mode = "host-passthrough"
      end

      vm.vm.provision "shell", inline: <<-'SHELL'
        set -euo pipefail

        # Root login with a password (lab only)
        printf '%s\n' 'PermitRootLogin yes' 'PasswordAuthentication yes' \
          > /etc/ssh/sshd_config.d/10-lab.conf
        echo "root:root" | chpasswd
        systemctl restart ssh

        # All node names in /etc/hosts (re-running does not duplicate them)
        grep -q 'k8s-the-hard-way' /etc/hosts || printf '%s\n' \
          '# k8s-the-hard-way' \
          '192.168.56.9  jumpbox' \
          '192.168.56.10 manager01' \
          '192.168.56.11 worker01' \
          '192.168.56.12 worker02' >> /etc/hosts

        # Shared shell settings for all users (vagrant and root)
        grep -q 'k8s-lab-shell' /etc/bash.bashrc || cat >> /etc/bash.bashrc <<'EOF'

# k8s-lab-shell
# Debian has no terminfo for terminals like xterm-ghostty, which breaks clear, less, vim
if ! infocmp "$TERM" >/dev/null 2>&1; then
  export TERM=xterm-256color
fi

alias k=kubectl
# Completion for the k alias too, once kubectl is installed
if command -v kubectl >/dev/null 2>&1; then
  source <(kubectl completion bash)
  complete -o default -F __start_kubectl k
fi
EOF
      SHELL
    end
  end
end
