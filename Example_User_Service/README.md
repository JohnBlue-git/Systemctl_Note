## About

Here are examples to run the service in user-level (non system-wide-level)

## Way 1. service files locate in /etc

We can use `User=` and `Group=` in the `[Service]` section **allows a systemd service to run as another user**.
```ini
[Service]
User=<user>
Group=<user>
```

Then, we can control this service using **`sudo systemctl`**
```bash
sudo systemctl daemon-reexec         # optional, in case of edits to systemd itself
sudo systemctl daemon-reload         # reloads service units
sudo systemctl enable <*.service>
sudo systemctl start <*.service>
sudo systemctl status <*.service>
```

But there are important details and limitations:
- **cannot** control a **user service** (`systemctl --user`) for another user from your own user account — that requires logging in as that user.
- **cannot** use `User=` in a **user-level** service unit (`~/.config/systemd/user/`) — it's **ignored** in user mode.

## Way 2. service files locate in ~/.config/systemd/user

We can place service files `~/.config/systemd/user` (I have given the example service with timer in this folder).
\
Then **.timer** will automaticall find and exexcute the **.service** with the same service name.

\
~/.config/systemd/user/daily_build.service
```ini
[Unit]
Description=Run daily task script
After=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/mnt/ssd1/buildfarm/yujen.lan
ExecStart=/mnt/ssd1/buildfarm/yujen.lan/daily_build.sh
Environment=PATH=/home/yujen.lan/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
We can use `%h` to represent `~/`
```ini
Environment=PATH=%h/bin
```
We can also replace `ExecStart=` with direct commands 
```ini
ExecStart=/bin/bash -c '... && ...'
```
All the `PATH` mentioned in the .service files are recommented to be in absolute path for clear clarification.
\
Also, `WorkingDirectory=` and `Environment=PATH` are better to be explicitly set for well defined configuration.

\
~/.config/systemd/user/daily_build.timer
```bash
[Unit]
Description=Run daily task every day

[Timer]
OnCalendar=7:00
Persistent=true

[Install]
WantedBy=timers.target
```
Timer can alse be set to any time or be launched multiple times
```ini
OnCalendar=7:00
OnCalendar=8:00
...

OnCalendar=Mon 09:00
OnCalendar=Tue 09:00
OnCalendar=Wed 09:00
OnCalendar=Thu 09:00
OnCalendar=Fri 09:00
OnCalendar=Sat 09:00
OnCalendar=Sun 09:00

OnCalendar=2025-06-06 6:66:66
...

# or every period of intertval
OnUnitActiveSec=30min
```

\
Then, we can control this service using **`systemctl --user`**
```bash
# (optional) if we are going to execute .sh
#chmod +x daily_build.sh

sudo systemctl --user daemon-reexec         # optional, in case of edits to systemd itself
sudo systemctl --user daemon-reload         # reloads service units
sudo systemctl --user enable daily_build.timer
sudo systemctl start daily_build.timer
sudo systemctl status daily_build.timer
```

\
It's also worthy to note that:
- Using **.timer** with **.service** somehow better then using **crontab**, because corntab may fail
  - due to permission or envirment variable
  - crontab is not easy to trigger or debug compared to .service
- Using .service make it easier to manage starting or killing a group of programs
