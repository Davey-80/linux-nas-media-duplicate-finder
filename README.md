# Linux NAS Media Duplicate Finder

Find duplicate media content across multiple NAS shares and storage volumes from Linux using **fclones** and **SMB/CIFS**.

This project documents a practical workflow for analysing a media library distributed across multiple network shares. Instead of searching individual folders separately, SMB shares are mounted below a single directory tree on Linux, allowing `fclones` to compare media files across otherwise separate shares and NAS volumes.

> **Safe by design:** this workflow uses `fclones group`. It reads and compares files and generates a report without deleting, moving, linking or modifying the original media.

## Workflow

The duplicate detection runs on the Linux/Debian system, while the media itself remains stored on the NAS.

```text
Linux / Debian
      │
      ▼
   fclones
      │
      ▼
  SMB / CIFS
      │
      ▼
Synology / NAS
      │
      ▼
 Media Content
```

**Linux/Debian → fclones → SMB/CIFS → Synology/NAS → Media Content**

No media needs to be copied to the Linux system. `fclones` accesses the mounted NAS shares directly and performs the duplicate analysis from the Linux client.

## Overview

Media libraries tend to grow organically. Entertainment, downloads and other media may eventually exist in several locations:

```text
/mnt/NAS/
├── Downloads
├── Entertainment
└── Media
```

A duplicate stored in two different shares can easily go unnoticed.

Scanning each share independently does not solve that problem: files located in different SMB shares must be part of the **same comparison** to be detected as duplicates.

The solution used here is:

```text
Synology / NAS
      │
      ├── SMB share
      ├── SMB share
      └── SMB share
             │
             ▼
       Debian / Linux
             │
             ▼
          /mnt/NAS/
             │
             ▼
           fclones
             │
             ▼
   Duplicate media report
```

## 1. Discover NAS SMB Shares

Available shares can be queried from Linux with `smbclient`:

```bash
smbclient -L //NAS-IP -A ~/.smbcredentials
```

Example:

```text
Sharename
---------
Downloads
Entertainment
Media
Music
Archive
```

`IPC$` is an SMB service share and should not be mounted as a filesystem.

## 2. Install Required Packages

On Debian:

```bash
sudo apt update
sudo apt install cifs-utils smbclient
```

Verify the CIFS mount helper:

```bash
/usr/sbin/mount.cifs -V
```

## 3. Install fclones

`fclones` performs the actual duplicate-file detection in this workflow.

### Install Rust and Cargo

On Debian, install Cargo from the Debian repositories:

```bash
sudo apt update
sudo apt install cargo
```

Verify the installation:

```bash
cargo --version
```

### Install fclones

Install `fclones` using Cargo:

```bash
cargo install fclones
```

The executable is normally installed in:

```text
~/.cargo/bin/fclones
```

Verify that `fclones` is available:

```bash
fclones --version
```

If `fclones` is not found in your shell's `PATH`, try:

```bash
~/.cargo/bin/fclones --version
```

or add Cargo's binary directory to your `PATH`:

```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

To make this permanent for Bash:

```bash
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

> **Tested with:** `fclones 0.35.0`

## 4. Store SMB Credentials

Create:

```text
~/.smbcredentials
```

Example:

```ini
username=YOUR_USERNAME
password=YOUR_PASSWORD
```

Protect the credentials file:

```bash
chmod 600 ~/.smbcredentials
```

> **Important:** never commit this file or your actual NAS credentials to Git.

Add it to `.gitignore`:

```gitignore
.smbcredentials
```

## 5. Create a Central NAS Mount Structure

Create one parent directory for the NAS:

```bash
sudo mkdir -p /mnt/NAS
```

Then create mount points for the required shares:

```bash
sudo mkdir -p \
  /mnt/NAS/Downloads \
  /mnt/NAS/Entertainment \
  /mnt/NAS/Media
```

The actual share names are not important. What matters is that all relevant shares become accessible below one predictable directory structure:

```text
/mnt/NAS/
```

This makes the storage convenient for `fclones` as well as other Linux utilities and scripts.

## 6. Configure `/etc/fstab`

A typical SMB share can be configured as:

```fstab
//NAS-IP/Entertainment /mnt/NAS/Entertainment cifs credentials=/home/USER/.smbcredentials,vers=3.0,iocharset=utf8,_netdev,nofail,x-systemd.automount,x-systemd.device-timeout=10 0 0
```

For shares containing spaces, escape spaces in `/etc/fstab` with:

```text
\040
```

For example:

```text
Media Content
```

becomes:

```fstab
//NAS-IP/Media\040Content /mnt/NAS/Media-Content cifs credentials=/home/USER/.smbcredentials,vers=3.0,iocharset=utf8,_netdev,nofail,x-systemd.automount,x-systemd.device-timeout=10 0 0
```

After modifying `/etc/fstab`:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Verify the CIFS mounts:

```bash
findmnt -t cifs -o TARGET,SOURCE
```

## 7. Duplicate Detection with fclones

The actual duplicate detection is performed with `fclones`.

Check the installed version:

```bash
fclones --version
```

For duplicate analysis, this project deliberately uses:

```bash
fclones group
```

`group` scans files and reports groups containing identical content. It does **not** delete the files.

This distinction is particularly important when analysing an existing media library.

## 8. Select Media Content

For the initial use case, the scan targets video media.

The following file extensions are included:

```text
mkv   mp4   m4v   avi   mov
qt    mpeg  mpg   mpe   ts
m2ts  mts   vob   evo   wmv
asf   webm  flv   f4v   ogv
ogm   3gp   3g2   rm    rmvb
```

ISO images are deliberately excluded because an `.iso` file is not necessarily media content.

Other media categories, such as audio, can be added separately.

## 9. Scan for Duplicate Media Content

Run:

```bash
fclones --progress=true group \
  --cache \
  --ignore-case \
  --name '*.mkv' \
  --name '*.mp4' \
  --name '*.m4v' \
  --name '*.avi' \
  --name '*.mov' \
  --name '*.qt' \
  --name '*.mpeg' \
  --name '*.mpg' \
  --name '*.mpe' \
  --name '*.ts' \
  --name '*.m2ts' \
  --name '*.mts' \
  --name '*.vob' \
  --name '*.evo' \
  --name '*.wmv' \
  --name '*.asf' \
  --name '*.webm' \
  --name '*.flv' \
  --name '*.f4v' \
  --name '*.ogv' \
  --name '*.ogm' \
  --name '*.3gp' \
  --name '*.3g2' \
  --name '*.rm' \
  --name '*.rmvb' \
  --output duplicates-media.txt \
  /mnt/NAS
```

### `--cache`

Allows `fclones` to reuse previously calculated hashes for unchanged files. This is particularly useful for recurring scans of network storage.

### `--ignore-case`

Makes the extension filters case-insensitive. Therefore:

```text
movie.mkv
movie.MKV
movie.Mkv
```

are all included by:

```bash
--name '*.mkv'
```

### `--progress=true`

Displays progress while the media collection is being analysed.

### `--output`

Writes the duplicate groups to a report instead of relying on terminal output:

```text
duplicates-media.txt
```

## 10. Why Scan All Shares Together?

Consider the following files:

```text
/mnt/NAS/Downloads/Movie.mkv
/mnt/NAS/Entertainment/Movie.mkv
```

If both directories are analysed in the same `fclones` operation, the files can be compared with each other.

This is one of the main reasons for creating the central `/mnt/NAS` structure.

The same principle applies when the SMB shares reside on different NAS volumes. From the Linux side, they can still participate in the same duplicate scan.

## 11. What fclones Actually Detects

`fclones` identifies files with **identical content**.

The filenames and locations do not have to be identical.

For example:

```text
/mnt/NAS/Entertainment/Movie.mkv
/mnt/NAS/Media/Old-Movie-Copy.mkv
```

can be reported as duplicates when their file contents are identical.

This makes content-based comparison considerably more useful than simply searching for duplicate filenames.

## 12. Same Media ≠ Identical File

There is an important limitation.

These two files may represent exactly the same movie:

```text
Movie.1080p.H264.mkv
Movie.1080p.H265.mkv
```

but they are not identical files.

Likewise:

```text
Movie.mkv
Movie.mp4
```

may contain the same underlying media while having different container structures or encodes.

A standard `fclones` comparison will correctly treat these as different files.

Detecting **semantically duplicate media content** requires another layer of analysis, potentially involving:

- title matching
- duration
- resolution
- video codec
- audio tracks
- metadata
- perceptual or video fingerprinting

This project currently focuses on **exact duplicate media files**, while providing a foundation for more advanced media-content analysis.

## Safety

This workflow deliberately uses:

```bash
fclones group
```

for analysis.

Destructive operations such as:

```bash
fclones remove
fclones move
```

are intentionally **not** part of this workflow.

Always review the generated report before making changes to a media library. For important or irreplaceable media, maintain a verified backup before performing cleanup operations.

## Use Cases

This workflow can be useful for:

- Synology media libraries
- Linux-based media management
- home servers and homelabs
- movie and TV libraries
- family media collections
- download archives
- media spread across multiple NAS volumes
- Linux-accessible SMB storage

The workflow is not tied to one particular NAS layout or storage volume.

## Tested Environment

| Component | Environment |
|---|---|
| Client | Debian Linux |
| NAS | Synology DSM |
| Network storage | SMB / CIFS |
| Duplicate detection | fclones |
| Storage layout | Multiple SMB shares and NAS volumes |
| Scan mode | Non-destructive analysis |

## Future Improvements

Potential extensions to the project include:

- audio content scanning
- configurable media extension profiles
- CSV and JSON reporting
- calculating reclaimable storage
- sorting duplicate groups by potential savings
- comparing specific media shares
- detecting different encodes of the same media
- media fingerprinting
- recurring cached scans
- interactive review before cleanup

## License

MIT License

Contributions and additional media-library use cases are welcome.
