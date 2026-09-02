# WSL distro - size and compaction


### shutdown wsl
```
wsl --shutdown
```

### get distro status

```
wsl --status
```

### list WSL distro

```
wsl -l

Windows Subsystem for Linux Distributions:
Ubuntu-26.04 (Default)

```

### get the location of the vistual disk

in powershell -

```
(Get-ChildItem `
-Path HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss `
| Where-Object { $_.GetValue("DistributionName") -eq 'Ubuntu-26.04' } `
| Get-ItemProperty `
| Select-Object -First 1).BasePath + "\ext4.vhdx"


C:\Users\username\AppData\Local\wsl\{xxxxxxxxxxxxxxxxxxxxx}\ext4.vhdx
```

#### run the diskpart 

```
diskpart
```

A new window will open up with diskpart

### eun the compact command in the sequence

```
select vdisk file="C:\Users\username\AppData\Local\wsl\{xxxxxxxxxxxxxxxxxxxxx}\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk

```

close the diskpart window 

```
exit
```

#### if you are running multiple distros, then repeat for each
