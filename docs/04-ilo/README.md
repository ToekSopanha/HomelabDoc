# 04 — HPE iLO 5

> A tiny separate computer inside the ML30 that lets you control the server even when it is off or frozen.

## 1. What is it?

**iLO** (Integrated Lights-Out) is a management chip built into HPE servers. It has its own network connection and web page. It works **even when Proxmox is crashed or the server is powered off** (as long as it's plugged in).

Comparison: if the server is a house, iLO is a **remote-controlled caretaker** who can turn the power on/off, look at the screen, and read the thermometer — without you being there.

## 2. Why this homelab uses it

- Power on/off/reset the server remotely
- See the boot screen and BIOS (e.g., if Proxmox doesn't start)
- Check hardware health: temperatures, fans, disk, memory errors
- Update BIOS and firmware

## 3. Key words

| Word | Meaning |
|---|---|
| **iLO port** | A network port for iLO (dedicated, or shared with a normal NIC depending on setup) |
| **Remote console** | See the server's screen and use keyboard in your browser |
| **iLO Standard vs Advanced** | Standard is included. Some features (e.g., full remote console after the OS boots, virtual media) may need an **Advanced** license — check what your iLO shows |
| **IML / IEL** | Integrated Management Log / Event Log — hardware error history |
| **SPP** | Service Pack for ProLiant — HPE's firmware update bundle |
| **Default password** | Printed on a pull-out tag on the server. Change it! |

## 4. Step by step for this project

1. Plug a network cable into the iLO port.
2. During boot, press **F9** (System Utilities) → System Configuration → **iLO 5 Configuration Utility** → Network Options. Set a static IP, e.g., `192.168.0.210/24`, gateway `192.168.0.1`.
3. Open `https://192.168.0.210` in your browser (accept the certificate warning).
4. Log in with `Administrator` and the password from the server's tag.
5. **Create your own admin user**, then change or disable the default one.
6. Check **Information → Integrated Management Log** for hardware errors.
7. Check **System Information → Health** for fans, temperatures, memory.
8. Update iLO firmware and BIOS (Firmware & OS Software → Update Firmware).
9. **Never expose iLO to the internet.** Reach it only on the LAN or through Tailscale ([11](../11-tailscale/README.md)).

## 5. Useful actions

| Need | Where in iLO |
|---|---|
| Server frozen | Power & Thermal → Server Power → **Reset** |
| See boot screen | Remote Console → HTML5 console |
| Temperatures | Power & Thermal → Fans / Temperatures |
| Hardware errors | Information → Integrated Management Log |

## 6. Common mistakes

- Leaving the **default password** unchanged.
- Putting iLO on the internet — iLO has had serious security bugs in the past.
- Forgetting firmware updates.
- Not writing down the iLO IP — then you can't find it when you need it.

## 7. Check yourself

1. Can iLO turn the server on if Proxmox is crashed?
2. Should iLO be published through Cloudflare Tunnel?

<details><summary>Answers</summary>

1. Yes — it is independent from the main OS.
2. No. LAN or Tailscale only.

</details>

## 8. Learn more

- HPE iLO 5 user guide: <https://support.hpe.com/hpesc/public/docDisplay?docId=a00105236en_us>
