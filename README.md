# UniFi AP Outdoor SWX-M2: an experimental OpenWrt installation report

**One original 2.4 GHz UniFi AP Outdoor (SWX-M2 / U2O) successfully booted OpenWrt 22.03.7 and provided working Wi-Fi. This is a hardware-specific case report, not an official port, a maintained release, or a guarantee that another device can be flashed safely.**

## Important limitations

- OpenWrt 22.03 is end-of-life. Do not expose management services to the Internet.
- The tested image is a device-specific configuration repack and is deliberately NOT included here: it contains a hard-coded bridge MAC address. Distributing it unchanged would cause different devices to share that address.
- Removing that override produces a different image. A sanitized rebuild must not be described as the exact binary tested in this report.
- The OEM updater printed ten CRC verification failures during the one installation attempt. Subsequent independent readback and successful boot were confirmed, but the cause of those warnings remains unknown. This report does not advise ignoring CRC failures or retrying a failed flash.
- No later controlled reboot/power-cycle persistence test, formal throughput measurement, or long-term reliability test was performed.
- Recovery can require physical access, serial hardware and a verified recovery procedure. A backup alone does not guarantee remote recovery.

## Hardware and software tested

| Item | Observed value |
| --- | --- |
| Physical device | Original UniFi AP Outdoor, 2.4 GHz, FCC ID SWX-M2 |
| OEM board identifiers | UAP-Outdoor, U2O, system ID 0xe532 |
| CPU / memory | Atheros AR7241, 8 MiB flash, 64 MiB RAM |
| OEM firmware before installation | BZ.v4.3.28 |
| Retained OpenWrt profile | ubnt_nanostation-m, ath79/tiny |
| OpenWrt version | 22.03.7, r20341-591b7e93d3 |
| Running kernel | Linux 5.10.221 |
| Detected wireless PHY | Atheros AR9280 Rev:2, ath9k |

**Do not confuse this model with Outdoor+, Outdoor 5, AC Outdoor, XW hardware, or ordinary indoor UniFi AP models.** The NanoStation model name shown by OpenWrt reflects the retained device profile, not the label on the physical unit.

## What was changed in the tested image

The historical NanoStation M kernel, device tree, executables, kernel modules and packages were retained. Four filesystem configuration files were changed:

- `network`: bridge `eth0` and `eth1` into `br-lan`, enable STP, use a DHCP management client, and remove WAN interfaces. The original experimental build also included the owner-specific MAC override described above.
- `dhcp`: disable DHCPv4/v6 serving, router advertisements, NDP and the dnsmasq DNS listener.
- `firewall`: a single trusted-LAN zone, without WAN, NAT or LAN-to-WAN forwarding.
- `dropbear`: bind SSH to logical LAN, retaining the original authentication defaults.

No Wi-Fi credentials, raw calibration partitions, OEM configuration backup, or private SSH key were injected. Wi-Fi was disabled in the image. Secure wireless settings and a root password were configured separately after boot.

The root filesystem was repacked with OpenWrt host utilities and the existing kernel/payload layout was retained. The OEM update container used a `BZ.ar7240` header. Merely renaming a generic firmware download is not equivalent to constructing and validating that container.

## Installation sequence actually used

These are historical steps, not a ready-to-run installer. Do not substitute an unreviewed image into the commands below.

1. Identify the exact board and original firmware through OEM SSH. Record the flash layout and native updater hash.
2. Back up all original flash regions and the OEM configuration. Copy backups off the device and verify their checksums. Keep calibration, configuration and full-flash backups private.
3. Inspect image container boundaries, CRCs, kernel header, firmware size and protected-region exclusions. Independently review the changes and the exact image hash.
4. Transfer the reviewed candidate to `/tmp/` and verify its hash again on the device.
5. Run the OEM native check-only operation on that exact candidate:

   ```sh
   /sbin/fwupdate.real -c -d /tmp/reviewed-candidate.bin
   ```

   A successful check is not proof that the image will boot or that writing it is safe.

6. After explicit acceptance of physical recovery risk, exactly one native write was issued:

   ```sh
   /sbin/fwupdate.real -m -d /tmp/reviewed-candidate.bin
   ```

   The updater printed CRC failures and reported writing the U-Boot environment. The SSH session did not provide an explicit successful write exit status. No automatic retry, second flash, or manual reset was issued.

7. The device was later discovered at a new DHCP lease and identified by its hardware identity. Its new OpenWrt SSH host key was then recorded; it does not establish continuity with the OEM SSH key. Do not assume it will retain its OEM static address or use a generic recovery address after boot.
8. Verify the running system, filesystem, radio, flash readback and protected partitions before making further changes. Set a unique root password immediately on an isolated, trusted management network.

No network boot or RAM-only boot was used for this successful installation. TFTP recovery was not tested as part of this procedure.

## Verified results

- Actual OpenWrt boot and authenticated root SSH.
- A writable JFFS2 overlay and DHCP management client, with no LAN DHCP server.
- Both Ethernet ports in the LAN bridge. Only one uplink should be connected unless the topology has been deliberately designed to avoid loops.
- Readback of the actual firmware matched the candidate for all bytes before offset `0x580000`.
- The complete 1,835,008-byte kernel/device-tree prefix matched, and both uImage header and data CRCs were valid.
- The four-byte initial JFFS2 end marker changed from `deadc0de` to `19852003` after initialization. This is not established as the cause of the OEM updater warnings.
- Bootloader, environment, OEM configuration and ART/calibration contents hashed identically to the original backups after boot. The updater nevertheless did report an environment write.
- After separate wireless configuration, two WPA2-PSK AP interfaces were enabled and bridged to LAN. Their original SSIDs and keys are intentionally omitted here.
- The regulatory country was configured for the actual installation, with automatic channel selection and 20 MHz channel width. Configure the correct country for your own location, not this installation's country by default.
- Gateway connectivity and external DNS resolution succeeded. The owner subsequently reported that Wi-Fi worked very well; this is anecdotal user feedback, not a benchmark.

OpenWrt's partition indices differ from those of the OEM firmware. Never reuse OEM numeric MTD indices blindly after installation.

## Artifact identity and publication policy

The exact, non-distributed, device-specific tested image was 5,767,516 bytes, SHA-256:

```text
c0e1e9c0d01459989b46d62fb0fcca462109c1fd2ce4d5f1000329178171a37d
```

This checksum is a historical identity, not a download recommendation. No sanitized replacement binary has been hardware-tested or released here.

Before distributing a rebuild, remove device-specific identifiers, audit the unpacked filesystem and metadata for credentials and keys, validate the new artifact independently, and provide the applicable component licenses and corresponding source/build materials. A license applied to a new helper script does not replace the licensing obligations of Linux, OpenWrt or other included packages.

## References

- [Original Outdoor support discussion](https://forum.openwrt.org/t/adding-support-for-ubiquiti-unifi-ap-outdoor/114882)
- [Exact SWX-M2 hardware entry](https://openwrt.org/toh/hwdata/ubiquiti/ubiquiti_unifi_ap_outdoor_swx-m2)
- [Historical OpenWrt 22.03.7 ath79/tiny downloads](https://downloads.openwrt.org/releases/22.03.7/targets/ath79/tiny/)
- [OpenWrt 22.03.7 image recipes](https://github.com/openwrt/openwrt/tree/v22.03.7/target/linux/ath79/image)
- [OpenWrt release security support](https://openwrt.org/docs/guide-developer/security)

This report is independent of Ubiquiti and the OpenWrt project.
