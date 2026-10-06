# Platforms — scan targets and cleanup commands

Read this file in phases 1-2 after OS detection. Commands run only after phase-4 validation.

## Table of contents

1. [macOS](#macos)
2. [Windows](#windows)
3. [Linux](#linux)

## macOS

### Scan (lecture seule)

```bash
df -h /System/Volumes/Data
du -sh ~/* 2>/dev/null | sort -rh | head -10
du -sh ~/Library/"Application Support"/* 2>/dev/null | sort -rh | head -15
du -sh ~/Library/Caches/* 2>/dev/null | sort -rh | head -15
du -sh ~/Library/Containers/* 2>/dev/null | sort -rh | head -10
du -sh ~/"Group Containers" ~/Library/"Group Containers" 2>/dev/null
du -sh ~/.npm ~/.cache ~/Library/pnpm 2>/dev/null
du -sh ~/Library/Caches/Homebrew ~/Library/Caches/pip ~/Library/Caches/go-build 2>/dev/null
docker system df 2>/dev/null
du -sh ~/Downloads; ls -lhS ~/Downloads | head -15
du -sh ~/Library/Developer/Xcode ~/Library/Developer/CoreSimulator 2>/dev/null
du -sh ~/Library/Android ~/.android 2>/dev/null
tmutil listlocalsnapshots / 2>/dev/null
du -sh ~/.Trash 2>/dev/null
find ~ -maxdepth 4 -type f -size +500M -not -path "$HOME/Library/*" -not -path "$HOME/.Trash/*" -exec du -h {} \; 2>/dev/null | sort -rh | head -15
```

Récence d'un répertoire ambigu :

```bash
mdls -name kMDItemLastUsedDate <chemin>
ollama list 2>/dev/null   # modèles IA actifs = NEVER
```

### Nettoyage

| Cible | Commande |
|---|---|
| Homebrew | `brew cleanup --prune=all` |
| npm | `npm cache clean --force` |
| pip | `pip3 cache purge` |
| go | `go clean -cache -modcache` |
| Docker build cache | `docker builder prune -af` |
| Docker images inutilisées | `docker image prune -af` |
| Playwright | `rm -rf ~/Library/Caches/ms-playwright` |
| DerivedData / DeviceSupport | `rm -rf ~/Library/Developer/Xcode/DerivedData` |
| Simulateurs obsolètes | `xcrun simctl delete unavailable` |
| CoreSimulator devices | wipe via `xcrun simctl` par device (REVIEW) |
| Android SDK/AVD | `sdkmanager --uninstall` ou dossier entier (REVIEW) |

Chemin des traces d'apps désinstallées : `find ~/Library -maxdepth 3 -iname "*<app>*"` →
Application Support, Caches, Preferences, LaunchAgents, HTTPStorages, Containers,
Saved Application State, Logs, WebKit, Group Containers.

## Windows

### Scan (PowerShell, lecture seule)

```powershell
Get-PSDrive -PSProvider FileSystem
Get-CimInstance Win32_OperatingSystem | Select Caption
"$env:TEMP", "C:\Windows\Temp" | ForEach-Object {
  "{0}: {1:N2} GB" -f $_, ((Get-ChildItem $_ -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB) }
@("Downloads","Desktop","Documents") | ForEach-Object {
  "{0}: {1:N2} GB" -f $_, ((Get-ChildItem "$env:USERPROFILE\$_" -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB) }
@("$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cache",
  "$env:LOCALAPPDATA\BraveSoftware\Brave-Browser\User Data\Default\Cache",
  "$env:APPDATA\Mozilla\Firefox") | ForEach-Object {
  if (Test-Path $_) { "{0}: {1:N2} GB" -f $_, ((Get-ChildItem $_ -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB) } }
@("$env:APPDATA\npm-cache", "$env:LOCALAPPDATA\pip\Cache", "$env:LOCALAPPDATA\go-build",
  "$env:USERPROFILE\.cache") | ForEach-Object {
  if (Test-Path $_) { "{0}: {1:N2} GB" -f $_, ((Get-ChildItem $_ -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB) } }
"C:\Windows\SoftwareDistribution\Download: {0:N2} GB" -f ((Get-ChildItem "C:\Windows\SoftwareDistribution\Download" -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB)
docker system df 2>$null
(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\*').PSChildName
```

### Nettoyage

| Cible | Commande |
|---|---|
| Temp utilisateur | `Remove-Item "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue` |
| Windows Temp (admin) | `Remove-Item "C:\Windows\Temp\*" -Recurse -Force` |
| Windows Update cache (admin) | `Remove-Item "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force` |
| npm / pip / go | `npm cache clean --force` ; `pip cache purge` ; `go clean -cache` |
| Recycle Bin | `Clear-RecycleBin -Force` |
| Docker | `docker builder prune -af` ; `docker image prune -af` |
| Outil d'appoint | `Dism.exe /online /Cleanup-Image /StartComponentCleanup` (admin) |

## Linux

### Scan (lecture seule)

```bash
df -h /
du -sh ~/* ~/.cache ~/.npm 2>/dev/null | sort -rh | head -10
du -sh /var/cache/apt/archives /var/log /tmp 2>/dev/null
docker system df 2>/dev/null
journalctl --disk-usage 2>/dev/null
du -sh ~/.local/share/Trash 2>/dev/null
find / -xdev -type f -size +500M 2>/dev/null | xargs -r du -h | sort -rh | head -15
```

### Nettoyage

| Cible | Commande |
|---|---|
| APT cache | `sudo apt clean` |
| Journaux | `sudo journalctl --vacuum-size=100M` |
| tmp | `find /tmp -mindepth 1 -mtime +7 -delete` (prudent) |
| Trash | `gio trash --empty` |
| Docker | `docker system prune -af --volumes=false` |
| Snap inutiles | `snap list --all | awk '/disabled/{print $1, $3}'` puis `snap remove` |
