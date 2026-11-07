# 1. Подготовка хоста

[English](../en/01-host-setup.md) | **Русский**

Стенд поднимается на одной Linux-машине (далее **хост**): Vagrant создаёт четыре виртуальные машины через libvirt/KVM. Этот документ описывает, что нужно установить на хосте перед [ручной сборкой](02-manual.md) или [сборкой через Ansible](03-ansible.md).

## Требования к железу

| Ресурс | Минимум | Откуда |
| --- | --- | --- |
| CPU | 4 ядра с аппаратной виртуализацией (Intel VT-x / AMD-V) | ВМ получают 7 vCPU суммарно, KVM допускает переподписку |
| ОЗУ | 16 ГБ | ВМ занимают 9,5 ГБ, хосту нужно оставить запас |
| Диск | 15 ГБ свободного места | образ Debian, диски ВМ (thin provisioning), около 1 ГБ бинарников на jumpbox |
| Сеть | доступ в интернет | загрузка образа, пакетов Debian, бинарников Kubernetes и образов контейнеров |

| ВМ | vCPU | ОЗУ | IP |
| --- | --- | --- | --- |
| `jumpbox` | 1 | 1,5 ГБ | `192.168.56.9` |
| `manager01` | 2 | 4 ГБ | `192.168.56.10` |
| `worker01` | 2 | 2 ГБ | `192.168.56.11` |
| `worker02` | 2 | 2 ГБ | `192.168.56.12` |

Проверка поддержки виртуализации:

```bash
grep -cE 'vmx|svm' /proc/cpuinfo   # больше 0
ls -l /dev/kvm                     # устройство существует
```

## Что установить

| Компонент | Зачем | Нужен для |
| --- | --- | --- |
| QEMU/KVM и libvirt | гипервизор и управление ВМ | всех вариантов |
| Vagrant | описание стенда в `Vagrantfile`, создание ВМ | всех вариантов |
| Плагин `vagrant-libvirt` | провайдер libvirt для Vagrant | всех вариантов |
| `git`, `openssh-client` | клонирование репозитория, SSH к ВМ | всех вариантов |
| Ansible и коллекции `ansible.posix`, `community.crypto`, `community.general` | запуск playbook | только [сборки через Ansible](03-ansible.md) |

> [!NOTE]
> Команды ниже приведены для распространённых дистрибутивов. Способ установки `vagrant-libvirt` зависит от дистрибутива и версии Vagrant. Если команда не подошла, сверьтесь с [официальной инструкцией vagrant-libvirt](https://vagrant-libvirt.github.io/vagrant-libvirt/installation.html).

### Debian 12 / Ubuntu

В Debian 12 Vagrant и плагин есть в штатных репозиториях:

```bash
sudo apt update
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients \
  vagrant vagrant-libvirt git openssh-client
```

Если пакета `vagrant` в вашем дистрибутиве нет, установите Vagrant из [репозитория HashiCorp](https://developer.hashicorp.com/vagrant/install), а плагин соберите командой `vagrant plugin install vagrant-libvirt` (зависимости для сборки перечислены в инструкции vagrant-libvirt).

### Fedora

```bash
sudo dnf install -y @virtualization vagrant vagrant-libvirt git openssh-clients
```

### Arch Linux

```bash
sudo pacman -S --needed qemu-base libvirt dnsmasq vagrant git openssh
vagrant plugin install vagrant-libvirt
```

### Права и служба libvirt

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt "$USER"
```

После `usermod` выйдите из сессии и войдите снова, чтобы группа применилась.

Чтобы не указывать `--provider=libvirt` в каждой команде, задайте провайдер по умолчанию (например, в `~/.bashrc` или `~/.zshrc`):

```bash
export VAGRANT_DEFAULT_PROVIDER=libvirt
```

### Ansible (только для сборки через Ansible)

Рекомендуемый способ из документации Ansible: установка через `pipx`.

```bash
pipx install --include-deps ansible
```

Пакет `ansible` уже включает нужные коллекции. Если установлен только `ansible-core` (например, из пакетов дистрибутива), доставьте коллекции из файла репозитория:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

Проверенные версии: `ansible-core` 2.21.4, `ansible.posix` 2.2.2, `community.crypto` 3.4.0, `community.general` 13.4.0.

## Проверка

```bash
virsh -c qemu:///system list --all   # без ошибок доступа
vagrant --version
vagrant plugin list                  # в списке есть vagrant-libvirt
ansible --version                    # только для варианта с Ansible
```

> [!WARNING]
> Стенд создаёт libvirt-сеть `k8s-hard-way` с адресами `192.168.56.0/24`. Если этот диапазон уже занят (например, host-only сетью VirtualBox `vboxnet0`), сеть не создастся. Удалите конфликтующую сеть или поменяйте адреса в `Vagrantfile`, `ansible/inventory.ini` и инструкциях.

## Запуск стенда

Из корня репозитория:

```bash
git clone https://github.com/ZeroNiki/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way

vagrant up --provider=libvirt
vagrant status
```

> [!NOTE]
> **Ожидаемый результат:** `vagrant status` показывает четыре машины (`jumpbox`, `manager01`, `worker01`, `worker02`) в состоянии `running`.

<details>
<summary>Что делает Vagrantfile</summary>

- Создаёт четыре ВМ из образа `debian/bookworm64`, параметры каждой заданы в массиве `nodes`.
- Подключает ВМ к отдельной libvirt-сети `k8s-hard-way` (`private_network`) со статическими адресами. Через неё ноды общаются между собой.
- Кроме неё, vagrant-libvirt подключает каждую ВМ к служебной сети `vagrant-libvirt` (`192.168.121.0/24`): через неё идут доступ в интернет и `vagrant ssh`. Маршрут по умолчанию ведёт именно туда, поэтому некоторым компонентам Kubernetes нужно явно указать адрес из `192.168.56.0/24`.
- Отключает общую папку `/vagrant`: файлы между машинами передаются только по SSH, как на реальных серверах.
- `cpu_mode = "host-passthrough"` передаёт ВМ инструкции процессора хоста.
- Provisioning разрешает вход root по паролю `root` (только для лаборатории), прописывает имена нод в `/etc/hosts`, добавляет алиас `k=kubectl` и исправляет `TERM` для терминалов, чьего terminfo нет в Debian (например, ghostty).

</details>

Полезные команды:

| Команда | Действие |
| --- | --- |
| `vagrant ssh jumpbox` | войти на ВМ |
| `vagrant halt` | выключить все ВМ |
| `vagrant reload` | перезагрузить ВМ (после неё нужно заново добавить маршруты подов) |
| `vagrant destroy -f` | удалить все ВМ |

Дальше: [ручная сборка](02-manual.md) или [сборка через Ansible](03-ansible.md).
