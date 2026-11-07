# 1. Host Setup

**English** | [Русский](../ru/01-host-setup.md)

The lab runs on a single Linux machine (the **host**): Vagrant creates four virtual machines through libvirt/KVM. This document lists what has to be installed on the host before the [manual build](02-manual.md) or the [Ansible build](03-ansible.md).

## Hardware Requirements

| Resource | Minimum | Why |
| --- | --- | --- |
| CPU | 4 cores with hardware virtualization (Intel VT-x / AMD-V) | the VMs get 7 vCPUs in total, KVM allows overcommit |
| RAM | 16 GB | the VMs use 9.5 GB, the host needs headroom |
| Disk | 15 GB free | Debian box, VM disks (thin provisioned), about 1 GB of binaries on the jumpbox |
| Network | Internet access | box, Debian packages, Kubernetes binaries and container images |

| VM | vCPU | RAM | IP |
| --- | --- | --- | --- |
| `jumpbox` | 1 | 1.5 GB | `192.168.56.9` |
| `manager01` | 2 | 4 GB | `192.168.56.10` |
| `worker01` | 2 | 2 GB | `192.168.56.11` |
| `worker02` | 2 | 2 GB | `192.168.56.12` |

Check virtualization support:

```bash
grep -cE 'vmx|svm' /proc/cpuinfo   # greater than 0
ls -l /dev/kvm                     # the device exists
```

## What to Install

| Component | Purpose | Required for |
| --- | --- | --- |
| QEMU/KVM and libvirt | hypervisor and VM management | all options |
| Vagrant | lab definition in `Vagrantfile`, VM creation | all options |
| `vagrant-libvirt` plugin | libvirt provider for Vagrant | all options |
| `git`, `openssh-client` | cloning the repository, SSH to the VMs | all options |
| Ansible and the `ansible.posix`, `community.crypto`, `community.general` collections | running the playbook | [Ansible build](03-ansible.md) only |

> [!NOTE]
> The commands below cover common distributions. How `vagrant-libvirt` is installed depends on the distribution and the Vagrant version. If a command does not work for you, follow the [official vagrant-libvirt installation guide](https://vagrant-libvirt.github.io/vagrant-libvirt/installation.html).

### Debian 12 / Ubuntu

Debian 12 ships Vagrant and the plugin in its main repositories:

```bash
sudo apt update
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients \
  vagrant vagrant-libvirt git openssh-client
```

If your distribution has no `vagrant` package, install Vagrant from the [HashiCorp repository](https://developer.hashicorp.com/vagrant/install) and build the plugin with `vagrant plugin install vagrant-libvirt` (build dependencies are listed in the vagrant-libvirt guide).

### Fedora

```bash
sudo dnf install -y @virtualization vagrant vagrant-libvirt git openssh-clients
```

### Arch Linux

```bash
sudo pacman -S --needed qemu-base libvirt dnsmasq vagrant git openssh
vagrant plugin install vagrant-libvirt
```

### Permissions and the libvirt Service

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt "$USER"
```

Log out and back in after `usermod` so the group membership takes effect.

To avoid passing `--provider=libvirt` to every command, set the default provider (for example in `~/.bashrc` or `~/.zshrc`):

```bash
export VAGRANT_DEFAULT_PROVIDER=libvirt
```

### Ansible (Ansible build only)

The installation method recommended by the Ansible documentation is `pipx`:

```bash
pipx install --include-deps ansible
```

The `ansible` package already bundles the required collections. If only `ansible-core` is installed (for example from distribution packages), install the collections from the repository file:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

Tested versions: `ansible-core` 2.21.4, `ansible.posix` 2.2.2, `community.crypto` 3.4.0, `community.general` 13.4.0.

## Verification

```bash
virsh -c qemu:///system list --all   # no permission errors
vagrant --version
vagrant plugin list                  # vagrant-libvirt is listed
ansible --version                    # Ansible build only
```

> [!WARNING]
> The lab creates the libvirt network `k8s-hard-way` with addresses from `192.168.56.0/24`. If this range is already in use (for example by the VirtualBox host-only network `vboxnet0`), the network will not be created. Remove the conflicting network or change the addresses in `Vagrantfile`, `ansible/inventory.ini` and the guides.

## Starting the Lab

From the repository root:

```bash
git clone https://github.com/ZeroNiki/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way

vagrant up --provider=libvirt
vagrant status
```

> [!NOTE]
> **Expected result:** `vagrant status` shows four machines (`jumpbox`, `manager01`, `worker01`, `worker02`) in the `running` state.

<details>
<summary>What the Vagrantfile does</summary>

- Creates four VMs from the `debian/bookworm64` box; the parameters of each one live in the `nodes` array.
- Attaches the VMs to a dedicated libvirt network `k8s-hard-way` (`private_network`) with static addresses. The nodes talk to each other over this network.
- In addition, vagrant-libvirt attaches every VM to its management network `vagrant-libvirt` (`192.168.121.0/24`), which carries Internet access and `vagrant ssh`. The default route points there, so some Kubernetes components must be told explicitly to use the `192.168.56.0/24` address.
- Disables the `/vagrant` shared folder: files move between machines over SSH only, as on real servers.
- `cpu_mode = "host-passthrough"` exposes the host CPU features to the VMs.
- Provisioning allows root login with the password `root` (lab only), adds all node names to `/etc/hosts`, adds the `k=kubectl` alias and fixes `TERM` for terminals whose terminfo Debian lacks (for example ghostty).

</details>

Useful commands:

| Command | Action |
| --- | --- |
| `vagrant ssh jumpbox` | log in to a VM |
| `vagrant halt` | shut down all VMs |
| `vagrant reload` | reboot the VMs (pod routes have to be added again afterwards) |
| `vagrant destroy -f` | delete all VMs |

Next: [manual build](02-manual.md) or [Ansible build](03-ansible.md).
