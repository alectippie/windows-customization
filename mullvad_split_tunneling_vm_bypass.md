# Mullvad Split Tunneling and VM Bypass Notes

This document explains how to configure Mullvad split tunneling on Windows and how to route virtual machines through a physical network adapter instead of the Mullvad tunnel.

> **Important:** Excluded applications or bypassed VMs will use your normal internet connection instead of Mullvad. That means they may expose your normal public IP address.

---

## 1. Mullvad split tunneling

Mullvad split tunneling can be configured either through the Mullvad GUI or with the Mullvad CLI.

Run **Command Prompt** or **PowerShell** as Administrator.

### Check split tunneling status

```powershell
mullvad split-tunnel get
```

This shows whether split tunneling is enabled and lists excluded applications.

### Turn split tunneling on or off

```powershell
mullvad split-tunnel set on
```

```powershell
mullvad split-tunnel set off
```

### Add an excluded application

Example with Firefox:

```powershell
mullvad split-tunnel app add "C:\Program Files\Mozilla Firefox\firefox.exe"
```

### Remove an excluded application

```powershell
mullvad split-tunnel app remove "C:\Program Files\Mozilla Firefox\firefox.exe"
```

### Clear all excluded applications

```powershell
mullvad split-tunnel app clear
```

---

## 2. Mullvad GUI settings file

Some Mullvad GUI settings may be visible here:

```text
%LOCALAPPDATA%\Mullvad VPN\gui_settings.json
```

Do **not** rely on manually editing this file to change Mullvad settings.

Use the Mullvad GUI or CLI instead, because manual edits may be overwritten, ignored, or not properly applied by the Mullvad service.

---

## 3. Important Mullvad settings

### Lockdown mode

For VM bypass testing, **Lockdown mode should be off**.

If Lockdown mode is on, Mullvad may block all internet traffic that does not go through the VPN tunnel.

### Local network sharing

Local network sharing should usually be **on** if you want access to local network devices such as:

```text
192.168.x.x devices
routers
printers
local VMs
NAS devices
```

If local network sharing is off, Mullvad may block access to local LAN devices.

---

## 4. Why VMs are different from normal apps

Normal Mullvad split tunneling excludes individual programs.

Virtual machines are different because their traffic usually goes through virtual network adapters, NAT adapters, bridges, or Hyper-V switches instead of acting like one normal Windows application.

For VMs, the cleaner method is usually:

```text
VM → external/bridged adapter → physical network adapter → router
```

Instead of:

```text
VM → host NAT → Mullvad tunnel
```

---

## 5. Hyper-V external switch for VM bypass

A Hyper-V external switch binds the VM network to a real physical network adapter.

This can give the VM a normal local network IP address, such as:

```text
192.168.x.x
```

### Create an external switch

Run PowerShell as Administrator:

```powershell
New-VMSwitch -Name "VM-Bypass-Mullvad" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

Replace `"Ethernet"` with the name of the physical adapter you want to use.

Examples:

```text
Ethernet
Wi-Fi
Realtek Gaming GbE Family Controller
```

### List Hyper-V switches

```powershell
Get-VMSwitch | Format-Table Name,SwitchType,NetAdapterInterfaceDescription,Id
```

Use this to get the switch name and switch ID.

### Map Windows network adapters

```powershell
Get-NetAdapter | Format-Table Name,InterfaceDescription,ifIndex,MacAddress
```

This helps match adapter names like:

```text
Ethernet
Wi-Fi
Mullvad
vEthernet (Default Switch)
vEthernet (VM-Bypass-Mullvad)
```

---

## 6. NanaBox network configuration

NanaBox can use a Hyper-V switch by setting the VM network adapter to the created switch.

Example:

```json
"NetworkAdapters": [
  {
    "Connected": true,
    "EndpointId": "",
    "MacAddress": "YOUR_MAC_ADDRESS",
    "SuggestedSwitchId": "YOUR_SWITCH_ID",
    "SuggestedSwitchName": "VM-Bypass-Mullvad"
  }
]
```

### Field notes

```text
EndpointId:
  Usually generated when the VM runs.

MacAddress:
  Can be chosen manually.

SuggestedSwitchId:
  Should match the ID from Get-VMSwitch.

SuggestedSwitchName:
  Should match the Hyper-V switch name.
```

Expected result:

```text
NanaBox VM → Hyper-V external switch → physical adapter → router
```

The VM should receive a normal LAN IP address like:

```text
192.168.x.x
```

---

## 7. Hyper-V Manager method

You can also attach a VM to the bypass switch through Hyper-V Manager.

1. Open **Hyper-V Manager**.
2. Right-click the VM.
3. Click **Settings**.
4. Go to **Network Adapter**.
5. Set **Virtual switch** to:

```text
VM-Bypass-Mullvad
```

6. Start the VM.
7. Inside the VM, check its IP address.

Windows command inside the VM:

```cmd
ipconfig
```

The VM should have a LAN IP address such as:

```text
192.168.x.x
```

---

## 8. VirtualBox bridged networking options

VirtualBox can also bypass Mullvad by using bridged networking.

### Option A: Bridge directly to the physical adapter

Example path:

```text
VM → VirtualBox bridge driver → Realtek physical adapter → router
```

In VirtualBox:

1. Open the VM settings.
2. Go to **Network**.
3. Set **Attached to**:

```text
Bridged Adapter
```

4. Select the physical adapter, such as:

```text
Realtek Gaming GbE Family Controller
Wi-Fi adapter
```

### Option B: Bridge to the Hyper-V vEthernet adapter

If you created a Hyper-V external switch, VirtualBox may also see a related adapter such as:

```text
vEthernet (VM-Bypass-Mullvad)
```

Example path:

```text
VM → VirtualBox bridge driver → Hyper-V vEthernet adapter → Hyper-V external switch → physical adapter → router
```

Use this command to map adapters:

```powershell
Get-NetAdapter | Format-Table Name,InterfaceDescription,ifIndex,MacAddress
```

---

## 9. Testing the VM connection

Inside the VM, check the local IP.

### Windows VM

```cmd
ipconfig
```

### Linux VM

```bash
ip addr
```

Expected local IP:

```text
192.168.x.x
```

Then open a browser inside the VM and check whether the VM is using Mullvad.

Expected result for a bypassed VM:

```text
Not using Mullvad VPN
```

Expected result for a VPN-routed VM:

```text
Using Mullvad VPN
```

---

## 10. Troubleshooting

### VM has no internet

Check:

```text
- The VM network adapter is connected.
- The VM is attached to the correct switch or bridged adapter.
- The physical adapter has internet access.
- Mullvad Lockdown mode is off.
- The VM received a valid LAN IP address.
```

### VM gets a 169.254.x.x address

A `169.254.x.x` address usually means the VM did not receive an IP address from DHCP.

Check:

```text
- Router DHCP is working.
- The VM is bridged to the correct adapter.
- The selected physical adapter is actually connected.
- The Hyper-V external switch is bound to the intended adapter.
```

### VM still uses Mullvad

Check:

```text
- The VM is not using Hyper-V Default Switch NAT.
- The VM is not using a NAT adapter routed through the host.
- The VM is attached to the external switch or bridged adapter.
- Mullvad Lockdown mode is off.
- The selected adapter is not being forced through Mullvad.
```

### Local devices are blocked

Check:

```text
- Mullvad local network sharing is on.
- The VM has a normal LAN IP address.
- The host firewall or VM firewall is not blocking LAN traffic.
```

---

## 11. Quick command reference

### Mullvad

```powershell
mullvad split-tunnel get
mullvad split-tunnel set on
mullvad split-tunnel set off
mullvad split-tunnel app add "C:\Path\To\App.exe"
mullvad split-tunnel app remove "C:\Path\To\App.exe"
mullvad split-tunnel app clear
```

### Hyper-V

```powershell
New-VMSwitch -Name "VM-Bypass-Mullvad" -NetAdapterName "Ethernet" -AllowManagementOS $true
Get-VMSwitch | Format-Table Name,SwitchType,NetAdapterInterfaceDescription,Id
Get-NetAdapter | Format-Table Name,InterfaceDescription,ifIndex,MacAddress
```

### Windows VM

```cmd
ipconfig
```

### Linux VM

```bash
ip addr
```

---

## 12. Recommended setup

For the cleanest VM bypass setup:

```text
VM network adapter
→ Hyper-V external switch or VirtualBox bridged adapter
→ physical Ethernet/Wi-Fi adapter
→ router
→ normal internet connection
```

Avoid using the Hyper-V Default Switch if the goal is to bypass Mullvad, because the Default Switch uses host NAT and may follow the host's Mullvad routing behavior.
