# 1. pfSense Firewall

[⬅️ Back to README](README.md) · [Next: Bastion Host ➡️](DebianBastionHost_Checho.md)

pfSense is the core of the lab: it separates every subnet and decides which traffic can move between them and out to the Internet. This guide covers installing it on VMware Workstation and the initial WAN and LAN setup.

> 🎥 Reference video: <https://www.youtube.com/watch?v=p09ggIHt2j4>

## Requirements

| Resource | Value |
| -------- | ----- |
| Hypervisor | VMware Workstation |
| ISO | pfSense Community Edition (<https://www.pfsense.org/download/>) |
| vCPU | 2 (minimum) |
| RAM | 4 GB |
| Network adapters | 2 (WAN + LAN) |

---

## Installation

### 1. Download the ISO

Download the ISO image from <https://www.pfsense.org/download/>.

### 2. Create the virtual machine

Create a new VM in VMware using the pfSense ISO.

![Creating the VM with the pfSense ISO](images/Creacion_Maquina_virtual_pfSense_ISO.png)

### 3. Create the internal network in the Virtual Network Editor

Open the **Virtual Network Editor** as administrator and create a new network:

- Type: **Host-only**.
- **DHCP disabled**, so that addresses are handed out by the firewall rather than by VMware.

![Virtual Network Editor configuration](images/Virtual_Network_Editor_config.png)

### 4. Configure the VM's network adapters

Before powering on the VM, add a second network adapter and assign 2 processors and 4 GB of RAM.

- **Adapter 1 (WAN):** connection type **Bridged**.

  ![Adapter 1 - WAN in Bridged mode](images/pfSense_VM_settings_network_adapter_1.png)

- **Adapter 2 (LAN):** connection type **Custom**, selecting the virtual network created in step 3 (in this case **VMnet2**).

  ![Adapter 2 - LAN on VMnet2](images/pfSense_VM_settings_network_adapter_2.png)

### 5. Install pfSense

Power on the VM and follow the installer. When assigning interfaces, make sure that:

- The interface connected to **VMnet0** (Bridged) is assigned as **WAN**.
- The interface connected to **VMnet2** is assigned as **LAN**.

### 6. Validate the interface assignment

When it finishes, the console shows the IP assigned to each interface:

![Validating the WAN and LAN interfaces](images/Validacion_config_interfaces.png)

### 7. Test Internet connectivity

From the pfSense console (option `7) Ping host`) or from the shell:

```sh
ping 8.8.8.8
```

![Connectivity test](images/prueba_conexion.png)

---

## Troubleshooting: VMs lose connectivity until the firewall reboots

**Symptom:** VMs connected to the firewall lost connectivity and only recovered after pfSense was restarted.

**Fix:** change the virtual adapter type of the interfaces to `vmxnet3`, which has better support in FreeBSD.

1. Shut down the pfSense VM.
2. Open the VM's configuration file in a text editor (in this case `pfSense Firewall.vmx`), located in the folder where the VM is stored.
3. Find every line like this:

   ```ini
   ethernet0.virtualDev = "e1000"
   ```

   (it may also appear as `"vlance"`) and change the value to:

   ```ini
   ethernet0.virtualDev = "vmxnet3"
   ```

   Repeat for `ethernet1`, `ethernet2`, and so on.
4. Save and close the file.
5. Power on the VM and **reassign the interfaces** from the console (option `1) Assign Interfaces`), because their names change from `em0`/`le0` to `vmx0`, `vmx1`, etc.

---

## Not yet documented

- Creating the additional interfaces (OPT1–OPT5) and assigning their IPs for each lab subnet (see the addressing table in the [README](README.md#addressing)).
- Accessing the pfSense web UI and changing the default `admin` password.

[⬅️ Back to README](README.md) · [Next: Bastion Host ➡️](DebianBastionHost_Checho.md)
