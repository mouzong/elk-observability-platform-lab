# -*- mode: ruby -*-
# vi: set ft=ruby :

# Executed inside each VM by Vagrant.
NETWORK_SCRIPT = <<-'SHELL'
set -euo pipefail

mac="$1"
address="$2"
profile="$3"

# Find the interface by MAC rather than assuming eth1/enp0s8.
device=""
for attempt in $(seq 1 15); do
  for path in /sys/class/net/*; do
    if [ "$(cat "$path/address")" = "$mac" ]; then
      device="${path##*/}"
      break
    fi
  done

  [ -n "$device" ] && break
  sleep 1
done

if [ -z "$device" ]; then
  echo "ERROR: Private adapter with MAC $mac is missing."
  ip -br link
  exit 1
fi

systemctl start NetworkManager
nmcli device set "$device" managed yes

# Create once; update the same profile on subsequent provisioning runs.
if ! nmcli connection show "$profile" >/dev/null 2>&1; then
  nmcli connection add \
    type ethernet \
    con-name "$profile" \
    ifname "$device"
fi

nmcli connection modify "$profile" \
  connection.interface-name "$device" \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  802-3-ethernet.mac-address "$mac" \
  ipv4.method manual \
  ipv4.addresses "$address" \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  ipv6.method disabled

nmcli connection up "$profile" ifname "$device"

# Remove the malformed file left by the previous failed configuration.
if [ -f /etc/sysconfig/network-scripts/ifcfg- ]; then
  rm -f /etc/sysconfig/network-scripts/ifcfg-
fi

echo "Configured $device with $address"
ip -4 -br address show dev "$device"
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "oraclelinux/8-aarch64"
  config.vm.box_url =
    "https://oracle.github.io/vagrant-projects/boxes/oraclelinux/8-aarch64.json"

  config.ssh.insert_key = true
  config.vm.synced_folder ".", "/vagrant", disabled: true

  vms = [
    {
      name: "es01", cpus: 2, memory: 8192,
      ips: ["192.168.10.11"]
    },
    {
      name: "es02", cpus: 2, memory: 5120,
      ips: ["192.168.10.12"]
    },
    {
      name: "es03", cpus: 2, memory: 5120,
      ips: ["192.168.10.13"]
    },
    {
      name: "ing01", cpus: 2, memory: 4096,
      ips: ["192.168.20.21"]
    },
    {
      name: "app01", cpus: 2, memory: 3072,
      ips: ["192.168.20.22"]
    },
    {
      name: "obs01", cpus: 2, memory: 4096,
      ips: ["192.168.30.31"]
    },
    {
      name: "ops01", cpus: 1, memory: 2048,
      ips: ["192.168.30.32"]
    },
    {
      name: "gw01", cpus: 1, memory: 1024,
      ips: [
        "192.168.10.10",
        "192.168.20.10",
        "192.168.30.10"
      ]
    }
  ]

  vms.each_with_index do |vm, vm_index|
    config.vm.define vm[:name] do |node|
      node.vm.hostname = vm[:name]

      node.vm.provider "virtualbox" do |vb|
        vb.name = vm[:name]
        vb.memory = vm[:memory]
        vb.cpus = vm[:cpus]
      end

      vm[:ips].each_with_index do |ip, network_index|
        # Adapter 1 remains Vagrant's NAT interface.
        adapter = network_index + 2

        # Stable, unique MAC for each private adapter.
        mac_raw = format(
          "020000%02X%02X01",
          vm_index + 1,
          network_index + 1
        )
        mac = mac_raw.scan(/../).join(":").downcase

        node.vm.network "private_network",
          ip: ip,
          netmask: "255.255.255.0",
          nic_type: "virtio",
          auto_config: false

        node.vm.provider "virtualbox" do |vb|
          vb.customize [
            "modifyvm", :id,
            "--nictype#{adapter}", "virtio",
            "--macaddress#{adapter}", mac_raw,
            "--cableconnected#{adapter}", "on"
          ]
        end

        node.vm.provision "shell",
          name: "configure-private-network-#{network_index + 1}",
          privileged: true,
          inline: NETWORK_SCRIPT,
          args: [
            mac,
            "#{ip}/24",
            "lab-private-#{network_index + 1}"
          ]
      end
    end
  end
end