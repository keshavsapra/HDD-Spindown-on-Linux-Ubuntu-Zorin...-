
# hd-idle Setup: Automatic Spindown for Extra HDDs on Ubuntu / Zorin OS (Using Stable Disk IDs)

This guide shows how to configure **hd-idle** to automatically spin down your additional hard disk drives (HDDs) when idle, while safely ignoring your NVMe/SSD OS drive.

**Why hd-idle?**  
It is lightweight, reliable, and works better than `hdparm` or GNOME Disks for many users — especially with mixed SSD + HDD setups or drives that ignore standard power management settings.

**Goal of this repo**  
Provide a clear, step-by-step tutorial using **persistent disk identifiers** (`/dev/disk/by-id/`) instead of volatile device letters (`/dev/sdb`, etc.) that can change after reboot or drive changes.

## Prerequisites

- Ubuntu-based system (Zorin OS, Ubuntu 20.04+, Linux Mint, etc.)
- At least one extra HDD (SATA/USB) you want to spin down
- Your OS drive should be NVMe or SSD (do **not** spin it down!)
- Terminal access with `sudo` privileges

## Step 1 – Install hd-idle

```bash
sudo apt update
sudo apt install hd-idle
```

This installs the daemon and creates `/etc/default/hd-idle`.

## Step 2 – Identify Your Drives Using Stable IDs

Device names like `/dev/sdb` can change. Use persistent IDs instead.

Run:

```bash
# List stable disk IDs (most useful command)
ls -l /dev/disk/by-id/ | grep -E 'ata-|nvme-'
```

Example output:

```
lrwxrwxrwx 1 root root  9 Feb 26 10:00 ata-ST1000LM035-1RK172_Z12345678 -> ../../sdb
lrwxrwxrwx 1 root root  9 Feb 26 10:00 ata-WDC_WD10EZEX-08WN4A0_WD-WCC6Y1234567 -> ../../sdc
lrwxrwxrwx 1 root root 13 Feb 26 10:00 nvme-Samsung_SSD_970_EVO_1TB_ABCDEF123456 -> ../../nvme0n1
```

Copy the **ata-Model_Serial** parts for your extra HDDs (ignore nvme- entries).

Also check which are spinning drives:

```bash
lsblk -o NAME,TYPE,SIZE,MODEL,ROTA
```

(ROTA=1 means traditional HDD)

## Step 3 – Configure hd-idle with Stable IDs

Edit the configuration file:

```bash
sudo nano /etc/default/hd-idle
```

Replace its content with (using **your actual IDs**):

```bash
HD_IDLE_OPTS="-i 0 \
  -a ata-ST1000LM035-1RK172_Z12345678 -i 900 \
  -a ata-WDC_WD10EZEX-08WN4A0_WD-WCC6Y1234567 -i 900 \
  -l /var/log/hd-idle.log"
```

### Key options explained

| Option                  | Meaning                                                                 |
|-------------------------|-------------------------------------------------------------------------|
| `-i 0`                  | Global/default timeout = disabled (protects OS NVMe/SSD and unlisted drives) |
| `-a ata-...`            | Targets specific drive using stable ID substring (hd-idle matches symlinks) |
| `-i 900`                | Spin down after 900 seconds (15 minutes) of no I/O. Common values: 300 (5 min), 1800 (30 min), 3600 (1 hr) |
| `-l /var/log/...`       | Optional: log spindown events to a file (helpful for debugging)        |

Save and exit nano:  
`Ctrl+O` → `Enter` → `Ctrl+X`

## Step 4 – Apply and Enable the Service

```bash
sudo systemctl restart hd-idle
sudo systemctl enable hd-idle     # start on boot
```

Check status:

```bash
sudo systemctl status hd-idle
```

Should show: `Active: active (running)`

## Step 5 – Verify It Works

Watch real-time logs:

```bash
journalctl -u hd-idle -f
```

Look for lines like:

```
hd-idle: probing sdb: model=ST1000LM035...
hd-idle: spindown: sdb after 900 seconds
```

Test spindown:

1. Make sure nothing is accessing your extra HDDs (close file managers, stop backups, etc.)
2. Wait > your timeout (e.g. 15+ minutes)
3. Check drive state:

```bash
sudo hdparm -C /dev/sdb     # replace with your actual device name
```

Expected result when spun down:

```
drive state is: standby
```

To wake it: just access a file on the drive.

## Troubleshooting

- **No spindown?**  
  → Test with short timeout first (`-i 300`)  
  → Some WD/Seagate drives need APM set via `hdparm -B 127 /dev/sdX`  
  → Check logs: `tail -f /var/log/hd-idle.log` (if enabled)

- **Wrong drive targeted?**  
  → Double-check IDs match exactly (case-sensitive)

- **Service won't start?**  
  → Run manually in debug: `sudo hd-idle -d -i 0 -a ata-ST1000LM035-... -i 300`

- **Frequent wake-ups?**  
  → Increase timeout (e.g. 1800–7200 s) or find what's accessing the drive (`iotop`, `lsof`)

## Recommended Idle Times

| Use Case                      | Timeout (seconds) | Real time     |
|-------------------------------|-------------------|---------------|
| Testing                       | 300               | 5 minutes     |
| Daily light use               | 900–1800          | 15–30 minutes |
| Backup/archive drive          | 3600–7200         | 1–2 hours     |
| Rarely used external drive    | 14400+            | 4+ hours      |

**Note:** Very frequent spin-up/down can wear mechanical drives faster than constant spinning.

## License

This guide and configuration examples are released under the **MIT License**.

---

Feel free to fork, modify, and share.  
Questions or improvements? Open an issue or PR!

Happy spinning down! 🛠️💾
