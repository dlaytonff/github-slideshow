# The prompt: LG OLED C1 (webOS 6)

Paste this into Claude Code (or another coding assistant with shell access) on a computer that is on the
same network as the TV. It is modelled on the Android TV prompt at
https://github.com/Gathin23/cleanyourtv/blob/main/PROMPT.md, but it uses the webOS port
https://github.com/lasharor/cleanyourtv-WebOS (branch `dev`), because the Android tooling (ADB, `pm`, APKs)
does not exist on webOS.

## Before you start (you do this, not the assistant)

1. **Root the TV yourself** with the Homebrew Channel, following the official guides only:
   https://www.webosbrew.org/rooting/ and https://rootmy.tv. Whether your C1 can be rooted depends on its
   exact firmware (Settings > General > About This TV > Software Version). The assistant does not do this step.
2. In the Homebrew Channel: Settings > **SSH Server** on. Put your computer's public key in the TV's
   `/home/root/.ssh/authorized_keys` (the Homebrew Channel docs say how) so `ssh root@<tv-ip>` works without a password.
3. Reserve a fixed IP for the TV in your router.
4. TV Settings > General > Home Settings > **Home Style: Basic** (from the install troubleshooting notes).
5. Turn **Quick Start+** on and leave the TV on for the whole session.
6. Write down your Software Version and webOS version from About This TV. You will be asked for them.

### On Windows

- Install **Git for Windows** (https://git-scm.com/download/win). Its Git Bash ships `sh`, `ssh` and `perl`,
  which `tools/diagnose.sh` needs. Run Claude Code from Git Bash, or from WSL (Ubuntu) if you already use it.
  Plain PowerShell has `ssh` (Windows 10/11 OpenSSH) but not `sh` or `perl`.
- Make the key in Git Bash: `ssh-keygen -t ed25519`, then copy the contents of `~/.ssh/id_ed25519.pub` to the
  TV as the Homebrew Channel's SSH docs describe.
- Allow `ssh` through Windows Firewall if asked. The connection goes out to the TV on port 22.

### Firmware 03.53.45 (webOS 6.5.3)

- In `compat.json` this firmware is listed as reported working (OLED48C16LA, OLED55G19LA, 65NANO753PR,
  OLED65B16LA, 55UP7550PVG), so the setup runs without **Set up anyway**.
- Root is the open question on this firmware. Check https://www.webosbrew.org/rooting/ for 03.53.45 first.
  Until you have root, do not accept a firmware update: a newer version is less likely to be rootable.

Known C1 reports in the project (`docs/tested-tvs.md`, `compat.json`): OLED55C16LA on firmware `03.34.80`
(SDK 6.3.2) and OLED48C16LA on `03.53.45` (SDK 6.5.3) both **work** with the native-Home stand-in.
On webOS 6, LG's Home is a native program, so the launcher takes over through a stand-in, not a mount.

---

I have a rooted LG OLED C1. Model: OLED55C1AUB. Software version: 03.53.45.
webOS version: 6.5.3-47 (SDK 6.5.3). TV IP: <IP>. My computer runs Windows; you are in Git Bash. I want LG's ads and telemetry removed and LG's Home
replaced with the CleanYourTV launcher. Work on the real TV, step by step, and verify each step.
Ask me before anything that depends on my preferences.

Ground rules
- Clone the project first and work from it, so its safety checks stay in place:
  `git clone -b dev https://github.com/lasharor/cleanyourtv-WebOS && cd cleanyourtv-WebOS`.
  Read `AGENTS.md` and `docs/ai.md` before running anything on the TV, and follow their rules.
- Follow AGENTS.md rule 1 strictly: read-only first. Before every command that changes the TV (install,
  setup, edit, restart, reboot, delete, boot scripts), tell me the exact command, what it does and the
  command that undoes it, and wait for my yes.
- Never write to `/usr`, `/etc` or the firmware. Never touch the Homebrew Channel itself, its failsafe or
  its update settings. Never install a firmware update or change update blocking.
- Never paste my IP, MAC, tokens or names into an issue, a pull request or a file in a repository.
- `luna-send` must have stdin redirected: `luna-send -n 1 -w 5000 -f <uri> '<json>' </dev/null`.
- Do not run rooting tools or exploits. If the TV turns out not to be rooted, stop and send me to
  https://www.webosbrew.org/rooting/.

Connect and check (read-only)
- `ssh -o BatchMode=yes -o ConnectTimeout=8 root@<ip> 'id; uname -a'` must print `uid=0`. If it asks for a
  password or is refused, stop and tell me what to fix in the Homebrew Channel's SSH settings.
- If your own commands cannot reach the TV while it answers from my ordinary terminal (a sandbox that blocks
  the local network), stop and tell me, do not try to work around it.
- Read model, firmware and SDK:
  `luna-send -n 1 -w 5000 -f luna://com.webos.service.tv.systemproperty/getSystemInfo '{"keys":["modelName","firmwareVersion","sdkVersion"]}' </dev/null`.
  Compare with what I told you and with `compat.json`. The SDK must start with 5 or higher (C1 ships 6.x).
  If my firmware is not listed, say so before setup: the setup will stop and ask for **Set up anyway**
  (`--force`), and that decision is mine.
- Check free space on `/media/developer` (`df -h /media/developer`) and that the TV can reach GitHub
  (`curl -sI https://github.com | head -1` on the TV).
- Check `/proc/cmdline` for `criu_enabled`. If present, tell me: on such TVs the stand-in can never run and
  the setup falls back to the mount way by itself (issues 72 and 84). That is expected, not a failure.

Back up first
- On my computer, make a dated folder `cytv-backup-<date>/` and save over SSH, read-only:
  the getSystemInfo answer, `ps`, `mount`, `cat /etc/hosts`, `ls -la /var/lib/webosbrew/init.d/`,
  `cat /home/root/.config/lginputhook/keybinds.json` (if it exists), the running and foreground apps
  (`applicationManager/running`, `getForegroundAppInfo`) and the installed app list
  (`applicationManager/listApps`).
- Note that the project's setup also saves the original state to `/media/developer/cleanyourtv/backup` on
  the TV. Tell me the three ways back before going on:
  1. Manager app > **CleanYourTV as Home** off (restarts the TV).
  2. `touch /media/developer/cleanyourtv/DISABLE; reboot`, or `sh /media/developer/cleanyourtv/undo-home.sh`.
  3. If a boot fails: the Homebrew Channel's failsafe mode skips all boot scripts.
  Full removal: `sh /media/developer/cleanyourtv/uninstall.sh`.

Ask me (one checklist question each)
- Debloat groups to enable (all on by default): Ads, Viewing tracking (LivePlus, `acr2`), Telemetry logs,
  Suggestions and nudges. Explain each in one line from `docs/debloat.md`. Ask whether I run Pi-hole,
  AdGuard Home or Glasshouse, because the Ads group is then partly redundant.
- Take over the remote's **Input** key with the Inputs list? (default on)
- Stable releases or dev builds? Recommend stable unless I am testing a fix.
- Dock: which four apps, and in what order.
- What is on each HDMI port, whether I use HDMI-CEC to control a box and whether audio goes to a
  soundbar over ARC or eARC. I need those to test that nothing broke (issues 14, 21, 23, 24).

Install (only after my yes)
- Install the manager app with the project's installer, which checks the SHA-256 of the package:
  `ssh root@<ip> 'curl -fsSL https://raw.githubusercontent.com/lasharor/cleanyourtv-WebOS/main/install.sh | sh'`
  (add `CYTV_CHANNEL=dev` before `sh` only if I chose dev builds). Read `install.sh` and show me a summary of
  what it does before running it.
- Run the setup either from the manager app on the TV (I press **Set up now**), or over SSH:
  `sh /media/developer/apps/usr/palm/services/com.cleanyourtv.manager.service/bootstrap.sh install`
  with `--no-debloat` and/or `--no-input-key` if I said no to those. Add `--force` only if I said yes to
  Set up anyway. Warn me first that the screen blinks and the TV may restart.
- Watch `/media/developer/cytv-setup.log` while it runs and stop on any `ERROR`. If the setup rolls back by
  itself, do not retry: read the log and `cytv-setup.log.service`, and tell me why.

Verify (after every change, and again after one full reboot)
- `curl -s -m 3 http://127.0.0.1:8765/api/service` and `/api/tv` on the TV: the service runs and the profile
  says webOS 6 with the stand-in (or mount, if CRIU).
- `getForegroundAppInfo` shows `com.webos.app.home` after I press Home, and I confirm the new launcher is on
  screen. Ask me to press: Home, Back, Input, Settings, and each streaming button on the remote, and tell
  you what each does.
- Open two apps from the dock (one of them Netflix or YouTube) and have me confirm picture and sound.
- Switch to each HDMI input I use: picture, sound to the soundbar if I have one, and CEC control of the box.
- For each debloat group I enabled: `ps | grep -E 'acr2|adoverlay|contentminer|nudge|sportsalarm'` and
  `grep cleanyourtv-debloat /etc/hosts`, about a minute after boot. Report what is stopped and what is not.
- Then propose one reboot, and after it repeat every check above. On webOS 6 LG's own Home can show for a
  while after a cold start before CleanYourTV (seen up to about 30 seconds on a 65NANO753PR): wait 60
  seconds before calling it a failure.
- If anything fails, run `sh tools/diagnose.sh <ip> > cytv-diagnose.txt` (read-only, masked), read it,
  and give me a diagnosis with the lines that support it before proposing any fix.

Finish
- Write a short `cytv-backup-<date>/README.md` with: what was installed, the options I chose, the
  firmware it was verified on, the results of each check (passed, failed, not tested), and the undo
  commands above.
- Remind me: SSH on the TV stays on and runs as root. Keep the key login, and do not expose port 22 to the
  internet. A firmware update from LG can remove root: do not update without checking the Homebrew
  Channel's notes first.
- If everything works, offer to draft (not post) a "works" report for
  https://github.com/lasharor/cleanyourtv-WebOS/issues with the masked Share log, so the C1 firmware gets
  added to `compat.json`.

---

Expect three or four checklist questions (debloat groups, Input key, dock apps, what is on HDMI).
Everything is reversible: nothing under `/usr`, `/etc` or the firmware is changed on disk.
