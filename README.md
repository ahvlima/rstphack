# RSTPHack – UniFi RSTP/STP Per-Port Cost Hack

This tool enables the modification of RSTP/STP per-port cost on UniFi switches and provides a method to keep the configuration quasi-persistent.

---
## Version History

## Quick Start

1. **Install dependencies**  
   ```bash
   pip3 install requests
   ```
2. **Clone or copy the script** to your server (can be the UniFi Controller host).  
3. **Update script configuration** with your UniFi Controller URL, username, password, and site:  
   ```python
   self.controller_url = "https://localhost:8443"
   self.username = "admin"
   self.password = "password"
   self.site = "default"
   ```
4. **Ensure SSH key authentication** works to the target switch.  
   Test with:  
   ```bash
   ssh admin@192.168.1.10
   ```
5. **Run the script manually**:  
   ```bash
   rstphack <switch_mac> <port_idx> <required_stp_cost> <user@ip_address>
   ```

**Example:**  
```bash
rstphack -q 00:11:22:33:44:55 3 200000 admin@192.168.1.10
```

6. **Automate with cron (recommended)**  

Open the crontab for editing:  
```bash
crontab -e
```

Add a line to run the script every 5 minutes (adjust as needed):  
```bash
*/5 * * * * /usr/local/bin/rstphack -q 00:11:22:33:44:55 3 200000 admin@192.168.1.10
```

This ensures the preferred STP cost is enforced automatically after reboots or reprovisioning.  

7. **Alternative: Automate with systemd**  

Create a service file `/etc/systemd/system/rstphack.service`:  
```ini
[Unit]
Description=RSTPHack UniFi STP Port Cost Enforcer

[Service]
Type=oneshot
ExecStart=/usr/local/bin/rstphack -q 00:11:22:33:44:55 3 200000 admin@192.168.1.10
```

Create a timer file `/etc/systemd/system/rstphack.timer`:  
```ini
[Unit]
Description=Run RSTPHack every 5 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Unit=rstphack.service

[Install]
WantedBy=timers.target
```

Enable and start the timer:  
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rstphack.timer
```

---

## Motivation

Ubiquiti does not provide an official method to adjust RSTP/STP per-port costs on UniFi switches.  
While it is possible to connect via SSH and use well-known commands to modify these values, such changes are **not persistent**: they are lost after a reboot or reprovisioning.  

To date, no native mechanism ensures persistence of these settings.

---

## Workaround (Hack)

> **Note:** Not all UniFi switches support programmatic per-port cost changes. See [Prerequisites](#prerequisites).

This workaround is not straightforward to implement but provides a practical solution to make per-port cost settings “quasi-persistent.”  

The method relies on a Python script that:  
1. Queries the UniFi Controller for per-port configuration.  
2. Compares the actual STP cost with the desired one.  
3. If different, connects directly to the switch (not the controller) via SSH to apply the change.  

The script must run on a Unix-based system (including the UniFi Controller itself). Any scheduling mechanism (e.g., `cron`) can be used to execute the script periodically.  

**Important Considerations:**  
- Scheduling requires care: the verification happens at the controller level, but the configuration change occurs at the switch level.  
- There is no reliable programmatic method to confirm changes directly on the switch. It can take several minutes for updates to appear in the controller.  
- Typically, per-port costs are adjusted to influence path preference in spanning-tree calculations. If a switch reboots, the initial path may not be the preferred one until the script executes again.  

The scheduling interval must balance:  
- The propagation delay of configuration changes from switch to controller.  
- The maximum tolerable time that traffic flows over the non-preferred path.  

---

## How It Works

The script interacts with the unpublished UniFi Controller API and uses SSH to apply changes on the switch:

1. Authenticate with the Controller.  
2. Retrieve port configurations using the switch MAC address. 
3. Extract STP cost (`stp_pathcost`) and port media type.  
4. Compare current and desired costs.  
5. If changes are needed, generate the correct port identifier (based on media type).  
6. Execute the required commands on the switch via SSH:  
   ```
   cli -c "configure" -c "interface {if_id}" -c "spanning-tree cost {required_cost}"
   ```  

> **Note:** There is no programatically way to ensure the command executed correctly on the switch, except for waiting for the new state to propagate to the UniFi Controller.

---

## Prerequisites

- Target switch must support the `cli` command with the `-c` option  
- Python 3.6 or later  
- Python `requests` package  
- Access to the UniFi Controller (username and password)  
- SSH access to UniFi switches enabled, using private key authentication  
- Optional: Python virtual environment  

---

## Installation

1. Choose an installation directory (e.g., `/usr/local/bin` on a controller).  
2. (Optional) Create and activate a virtual environment:  
   ```bash
   python3 -m venv .venv
   . .venv/bin/activate
   ```  
3. Install dependencies:  
   ```bash
   pip3 install requests
   ```  
4. Clone or download the script.  

*If installing on a UniFi Controller, `pip` and `venv` may need to be installed first (`python3` is usually included).*  

---

## Configuration

Update the hardcoded Controller settings in `rstphack`:  

```python
# Change these values to match your environment
self.controller_url = "https://localhost:8443"
self.username = "admin"
self.password = "password"
self.site = "default"
```

The script includes two shebangs at the top:  
- One for running inside a virtual environment without activation.  
- One for using the system-wide Python installation.  

Select or adapt as required.  

> **Note:** The virtual environment shebang may not work on all systems (notably macOS, due to `env` inconsistencies). Verified to work on UniFi OS 4.3.6 (UCKG2).

---

## Usage

```
rstphack [-q|--quiet] [-d|--dryrun] <switch_mac> <port_idx> <required_stp_cost> <user@ip_address>
```

### Options

- `-q, --quiet` – Suppress informational messages (errors are still shown)  
- `-d, --dryrun` – Show the intended SSH command without executing it  

### Parameters

- `switch_mac` – Switch MAC address (e.g., `00:11:22:33:44:55`)  
- `port_idx` – Port index to modify  
- `required_stp_cost` – Desired RSTP/STP path cost (1–200000000)  
- `user@ip_address` – SSH target (e.g., `admin@192.168.1.10`)  

### Example

```
rstphack -q 00:11:22:33:44:55 3 200000 admin@192.168.1.10
```

This will:  
1. Connect to the Controller.  
2. Locate the switch with the specified MAC.  
3. Check the cost of port `3`.  
4. If not `200000`, connect via SSH and update the configuration.  

---

## Verification

Changes are verified only via the Controller API. Although the switch applies the modification immediately, the Controller may take 5 minutes or more to reflect the update.  

---

## Interface ID Mapping

The script maps ports automatically:  
- **Gigabit Ethernet (GE)** → `gi{port_idx}` (e.g., `gi1`, `gi2`)  
- **10 Gigabit Ethernet (XG)** → `te{port_idx}` (e.g., `te1`, `te2`)  

---

## SSH Authentication

Authentication with the switch is via SSH keys, that is setup using the UniFi Controller UI (`Device&rarr;->Device Updates and Settings` on the top right corner).

---

## Output and Exit Codes

When not run with `--quiet`, the script outputs status messages with:  
- ✓ for success  
- ✗ for failure  

Exit codes:  
- `0` – Success  
- `1` – Failure  

---

## Security Considerations

- **SSL Verification** is disabled for Controller access. This avoids connection issues and complex settings.   

---

## Possible Enhancements

- Support switches where `cli` is only an alias to `telnet localhost`.  
- Add direct verification of port cost on the switch.  

---

## License

This script is provided under the MIT License, without warranty, for educational and operational purposes.  


---

## Systemd Unit Files

Instead of using cron, you can use **systemd** timers for better integration with modern Linux systems.

### Files

Two unit files are provided as examples:

- [`rstphack.service`](./rstphack.service)  
- [`rstphack.timer`](./rstphack.timer)

Place both files in `/etc/systemd/system/`:

```bash
sudo cp rstphack.service /etc/systemd/system/
sudo cp rstphack.timer /etc/systemd/system/
```

### Activation

Reload the systemd daemon and enable the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rstphack.timer
```

This will run `rstphack.service` automatically every 5 minutes (as defined in the timer).

To check status:

```bash
systemctl status rstphack.timer
```

And to see logs of the service runs:

```bash
journalctl -u rstphack.service
```

---


## Version History

- **0.9.1** – Added systemd unit files and instructions.  
- **0.9** – Initial release with cron example.
