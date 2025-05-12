## What is Systemd and systemctl
Systemd is an **init system** and a suite of system management tools that has largely replaced **SysVinit** on Linux distributions. It manages the startup and shutdown of the operating system and its services, offering a more robust and efficient approach than its predecessor.
\
Systemd is controlled using the `systemctl` command.

## Features
Systemd provides several advantages over traditional init systems:
- **Parallel Service Startup**: Reduces boot time by starting services concurrently.
- **Dependency Management**: Ensures correct startup order by declaring dependencies.
- **On-Demand Activation**: Services start only when needed, optimizing resource usage.
- **Journaling**: Centralized logging system for comprehensive event tracking.
- **cgroups Integration**: Allows resource limitations on services to prevent system overload.
- **Socket-Based Activation**: Enables faster and more reliable service startup.

## Commands

### Basic Commands
```bash
# Check systemd version
systemctl --version

# List all running services
systemctl list-units --type=service

# Start a service
sudo systemctl start <service-name>
# Stop a service
sudo systemctl stop <service-name>

# Enable a service to start at boot
sudo systemctl enable <service-name>
# Disable a service from starting at boot
sudo systemctl disable <service-name>

# View logs using journalctl
journalctl -u <service-name>
```

### Status Command
This **Systemd service status** output provides detailed information about the service.
- Loaded: Indicates that the **service definition file** (`httpd.service`) has been successfully **loaded** from `/usr/lib/systemd/system/`.
- Active: Indicates the running status
  - **active (running)** → Confirms that it is running without errors.
  - **inactive (dead)** → The service is not running and has stopped.
  - **failed** → The service encountered an error and terminated unexpectedly.
  - **activating** → The service is in the process of starting.
  - **deactivating** → The service is in the process of stopping.
  - **reloading** → The service is running but is currently reloading its configuration.
- Main PID: <pid> <binary>
- CGroup: Control Group (cgroup) Hierarchy
  - Meaning this service runs inside this **cgroup**, meaning it’s under **Systemd’s process management**.
  - The **list of PIDs (`4349`–`4354`)** represents **Apache worker processes** handling web requests.
  - **`-DFOREGROUND`** → Apache is running in **foreground mode**, meaning **it doesn’t fork into the background**, allowing Systemd to manage it directly.
```bash
httpd.service - The Apache HTTP Server
Loaded: loaded (/usr/lib/systemd/system/httpd.service; en
Active: active (running) since 金 2014-12-05 12:18:22 JST
Main PID: 4349 (httpd)
Status: "Total requests: 1; Current requests/sec: 0; Curr
CGroup: /system.slice/httpd.service
        ├─4349 /usr/sbin/httpd -DFOREGROUND
        ├─4350 /usr/sbin/httpd -DFOREGROUND
        ├─4351 /usr/sbin/httpd -DFOREGROUND
        ├─4352 /usr/sbin/httpd -DFOREGROUND
        ├─4353 /usr/sbin/httpd -DFOREGROUND
        └─4354 /usr/sbin/httpd -DFOREGROUND

12月 05 12:18:22 localhost.localdomain systemd[1]: Starting T
12月 05 12:18:22 localhost.localdomain systemd[1]: Started T
12月 05 12:22:40 localhost.localdomain systemd[1]: Started T
```

## Commands start/stop VS enable/disable
If you only start <service-name>, it will run now, but after a reboot, it won’t start automatically unless enabled.
\
If you enable <service-name>, it will start every time the system boots, even if manually stopped. (eanble will **create the symlink** pointing to actual service, while **removes the symlink**)

## Services

### Service Types
Systemd operates using **units**, which define various system resources:
- **Service Units (`.service`)**:
  - Manage system services.
  - Contain `[Unit]`, `[Service]`, and `[Install]` sections.
  - Use `ExecStart=`, `ExecStop=`, and other process control options.
- **Target Units (`.target`)**:
  - Group services for specific system states.
  - Used for **boot sequences and system states** (e.g., graphical vs. command-line).
  ```ini
  [Unit]
  Description=Multi-user System Target
  Requires=network.target
  After=network.target
  ```
- **Timer Units (`.timer`)**:
  - Schedule tasks similar to cron jobs.
  - Work **alongside** a `.service` file.
  - Contain `[Timer]` and `[Install]` sections.
  ```ini
  # This runs `mytask.service` on the first day of every month
  
  [Timer]
  OnCalendar=*-*-01 00:00:00
  Unit=mytask.service
  
  [Install]
  WantedBy=timers.target
  ```
- **Mount Units (`.mount`)**:
  - Handle filesystem mounts.
  - Use `[Mount]` sections.
  - Define mount points and options.
  ```ini
  [Mount]
  What=/dev/sdb1
  Where=/mnt/data
  Type=ext4
  ```
- **Device Units (`.device`)**:
  - Manage hardware devices.
  - Auto-generated based on detected devices.
  - Can trigger services **when a device appears**.
  ```ini
  # This ensures the service **stops if the device is unplugged**.
  
  [Unit]
  Description=USB Device Monitoring
  BindsTo=dev-sda.device
  ```
- **Socket Units (`.socket`)**:
  - Manage network sockets (like Unix domain sockets).
  - Used to **activate services on demand**.
  - Contains `[Socket]` section.
  ```ini
  # Systemd **starts `myapp.service` only when the socket receives a connection**.
  [Socket]
  ListenStream=<service path>/<service name>.sock
  
  ...
  ```
  - When to Use Which ?
    - **Use `ListenStream=<service path>/<service name>.sock`** when **only local processes** need to communicate (e.g., a web server talking to a backend).
    - **Use `ListenStream=8080`** when the service **must be accessible over the network**.
    - Compare Table
    | **Method**              | **Scope**              | **Speed** | **Security** | **Use Case** |
    |----------------------|------------------|------------|-------------|-------------|
    | `ListenStream=/run/myapp.sock` | Local system only | Fast | File permissions | IPC between local services |
    | `ListenStream=8080` | Local & remote | Moderate | Requires firewall | Public-facing services |
  - Using a Separate bmcweb.socket
    - Socket Activation: Systemd listens on port 443 and only starts bmcweb.service when a connection arrives. This reduces resource usage when idle.
    - Automatic Restart: If bmcweb.service crashes, the socket remains open, and systemd can restart the service when needed.
    - Parallel Handling: With ReusePort=true, multiple instances of bmcweb.service can handle requests efficiently.
    - Security & Isolation: Systemd manages the socket separately, reducing the risk of service failures affecting availability.`
    ```ini
    [Socket]
    ListenStream=443
    ReusePort-true
    
    [Install]
    WantedBy=sockets.target
    ```

### Serice File (.service)
A Systemd service file (.service) consists of three primary **sections**.
- `[Unit]` → Defines **dependencies** and metadata.
- `[Service]` → Controls **execution behavior**.
- `[Install]` → Manages **startup settings**.
```ini
[Unit]
Description=Apache HTTP Server
After=network.target
Requires=network.target
PartOf=web-server-group.target

[Service]
Type=forking
ExecStart=/usr/sbin/httpd -DFOREGROUND
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
User=www-data
Group=www-data
WorkingDirectory=/var/www

[Install]
WantedBy=multi-user.target
Alias=webserver.service
```
#### [Unit] Section
Defines **metadata** and general behavior of the service.
- **`Description=`** → A short summary of the service.
- **`After=`** → Specifies services that should start **before** this service.
- **`Requires=`** → Defines required dependencies; if one fails, this service stops.
- **`Wants=`** → Similar to `Requires`, but failures don’t stop this service.
- **`Conflicts=`** → Defines services that should not run alongside this service.
- **`PartOf=`** → Groups the service under a larger unit, ensuring that if the parent unit **stops**, this service stops as well. However, starting httpd.service does NOT automatically start after parant !!!
#### [Service] Section
Defines **how the service runs and its execution parameters**.
- **`ExecStart=`** → Command to start the service.
- **`ExecStop=`** → Command to stop the service.
- **`ExecReload=`** → Command to reload the service without stopping.
- **`Type=`** → Defines process handling
  - `simple (when no fork)`
  - `forking (parent exit, child remains)`
  - `oneshot (inactive once exit)`
  - `dbus (Systemd waits until the DBus name appears before marking it as running)`
  - `notify (The service notifies Systemd when it has finished starting. More reliable than simple since the process explicitly signals readiness.)`
  - `idle (The service starts after all active jobs are finished)`
- **`Restart=`** → Defines restart behavior on failure
  - `always (no matter what)`
  - `on-failure (exit not 0)`
  - `on-success (exit 0)`
  - `on-abnormal`
  - `on-abort`
  - `on-watchdog (timeout)`
  - `no`
- **`RestartSec=`** → Defines the time span before restarting. 
- **`KillMode=`** → Defines **how Systemd terminates processes** when the service stops or restarts. (`none`, `control-group`, `process (Only kills the main, remains childs)`, `mixed (Sends `SIGTERM` to the main process and `SIGKILL` to remaining child processes.)`)
- **`User=`** → Specifies the user under which the service runs.
- **`Group=`** → Defines the service’s group permissions.
- **`WorkingDirectory=`** → Sets the working directory of the process.
\
Additional commands
- **`RemainAfterExit=yes*`* → Command to ensures Systemd considers it "active", even after the command finishes.
  - Often used for initialization tasks.
- **`ExecStartPre=`** → Command(s) to run **before `ExecStart`** starts the service.
  - Used for **pre-start checks**, setting environment variables, or preparing resources.
- **`ExecStartPost=`** → Command(s) to run **after `ExecStart`** starts the service.
  - Useful for **logging, notifications, or dependent actions** after the service starts.
- **`ExecStopPost=`** → Command(s) to run **after `ExecStop`** stops the service.
  - Helps **clean up resources or execute final tasks** after shutdown.
\
Advance commands for priority / policy
- Nice:
  - Sets the nice value for the service. The value ranges from -20 (highest priority) to 19 (lowest priority).
- CPUSchedulingPolicy:
  - Sets the CPU scheduling policy. Common values are other, fifo, and rr (round-robin).
- CPUSchedulingPriority:
  - The priority range depends on the scheduling policy being used. For real-time policies like FIFO (First In, First Out) and RR (Round Robin), the range is usually from 1 (lowest) to 99 (highest).
```bash
[Service]
Nice=-10
CPUSchedulingPolicy=rr
CPUSchedulingPriority=50
```
#### [Install] Section
Defines the **startup behavior** when the service is enabled.
- **`WantedBy=`** → Specifies the target group that enables the service (e.g., `multi-user.target`).
- **`Alias=`** → Creates an **alternative name** for the service.

When you run `systemctl enable <service>`, Systemd:
1. **Creates a symlink** in `/etc/systemd/system/` pointing to the actual service file.
2. Ensures the service **automatically starts on boot**.
3. Follows the **target specified in `WantedBy=<.target>`**, determining when the service should start.

Here are some common targets:
- **`multi-user.target`** Ensure the service will **start automatically in multi-user mode**.
- **`graphical.target`** → For GUI-based systems.
- **`network.target`** → Ensures the service starts **after networking is available**.
- **`default.target`** → The default boot target.

## If Service Files have changed, please re-load and re-start to apply the change
After re-load, systemd will check which have been modified then restart the specific service without affecting others.
```bash
# reload
$ systemctl daemon-reload

# restart
$ systemctl restart <service name>
```

#### Where are Service Files used to locate
The common places:
```bash
/etc/systemd/system/*
/run/systemd/system/*
/lib/systemd/system/*
...
```
For a specific service, to see what systemd is reading
```bash
systemctl status <service name>
systemctl show <service name>
```
