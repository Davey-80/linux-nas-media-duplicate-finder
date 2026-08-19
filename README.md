
## Synology Media Content Duplicate Finder
Find duplicate media content across multiple Synology volumes and SMB shares from a Debian/Linux system using fclones.
This project documents a practical workflow for analysing a media library that is distributed across multiple NAS shares and storage volumes.
Instead of searching individual folders separately, Synology SMB shares are mounted below a single directory tree on Linux. This allows fclones to compare media files across otherwise separate shares and volumes.
Safe by design: this workflow uses fclones group. It reads and compares files and generates a report without deleting, moving, linking or modifying the original media.

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

Overview
Media libraries tend to grow organically.
entertainment, downloads and other media may eventually exist in several locations:
/mnt/NAS/

├── Downloads
├── Entertainment
└── Media

A duplicate stored in two different shares can easily go unnoticed.
Scanning each share independently does not solve that problem: a duplicate in a SMB share and another SMB share must be part of the same comparison to be detected.
The solution used here is:

Synology
   │
   ├── SMB share
   ├── SMB share
   ├── SMB share
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

1. Discover Synology SMB Shares
Available shares can be queried from Linux with smbclient:
smbclient -L //NAS-IP -A ~/.smbcredentials
Example:
Sharename
---------
Downloads
Entertainment
Media
Music
Archive
IPC$ is an SMB service share and should not be mounted as a filesystem.

2. Install Required Packages
On Debian:
sudo apt update
sudo apt install cifs-utils smbclient
Verify the CIFS mount helper:
/usr/sbin/mount.cifs -V

3. Store SMB Credentials
Create:
~/.smbcredentials
Example:
username=YOUR_USERNAME
password=YOUR_PASSWORD
Protect the file:
chmod 600 ~/.smbcredentials
Never commit credentials to Git.
For example, add this to .gitignore:
.smbcredentials

4. Create a Central NAS Mount Structure
Create one parent directory for the NAS:
sudo mkdir -p /mnt/NAS
Then create mount points for the required shares:
sudo mkdir -p \
  /mnt/NAS/Downloads \
  /mnt/NAS/Entertainment \
  /mnt/NAS/Media \

The important part is not the names themselves.
The important part is that all relevant shares become accessible below one predictable directory structure:
/mnt/NAS/
This makes the storage convenient for fclones as well as other Linux utilities and scripts.

5. Configure /etc/fstab
A typical SMB share can be configured as:
//NAS-IP/Entertainment /mnt/NAS/Entertainment cifs credentials=/home/USER/.smbcredentials,vers=3.0,iocharset=utf8,_netdev,nofail,x-systemd.automount,x-systemd.device-timeout=10 0 0
For shares containing spaces, escape spaces in /etc/fstab with:
\040
For example:
Media Content
becomes:
//NAS-IP/Media\040Content /mnt/NAS/Media-Content cifs credentials=/home/USER/.smbcredentials,vers=3.0,iocharset=utf8,_netdev,nofail,x-systemd.automount,x-systemd.device-timeout=10 0 0
After modifying /etc/fstab:
sudo systemctl daemon-reload
sudo mount -a
Verify the CIFS mounts:
findmnt -t cifs -o TARGET,SOURCE

6. Duplicate Detection with fclones
The actual duplicate detection is performed with fclones.
Check the installed version:
fclones --version
For duplicate analysis we deliberately use:
fclones group
group scans files and reports groups with identical content.
It does not delete the files.
This distinction is important when analysing an existing media library.

7. Select Media Content
For the initial use case, the scan targets video media.
Example extensions:
mkv
mp4
m4v
avi
mov
qt
mpeg
mpg
mpe
ts
m2ts
mts
vob
evo
wmv
asf
webm
flv
f4v
ogv
ogm
3gp
3g2
rm
rmvb
ISO images are deliberately excluded because an .iso file is not necessarily media content.
Other media categories, such as audio, can be added separately.

8. Scan for Duplicate Media Content
Example:
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
--cache
Caching allows fclones to reuse previously calculated hashes for unchanged files.
This is particularly useful for recurring scans of network storage.
--ignore-case
The extension filters become case-insensitive.
Therefore:
movie.mkv
movie.MKV
movie.Mkv
are all included by:
--name '*.mkv'
--progress=true
Displays progress while the collection is being analysed.
--output
Writes the duplicate groups to a report instead of relying on terminal output:
duplicates-media.txt

9. Why Scan All Shares Together?
Consider this structure:
/mnt/NAS/Downloads/Movie.mkv

/mnt/NAS/Entertainment/Movie.mkv
If both directories are analysed in the same fclones operation, the files can be compared with each other.
This is one of the main reasons for creating the central /mnt/NAS structure.
The same principle applies when the SMB shares reside on different NAS volumes.
From the Linux side, they can still participate in the same duplicate scan.

10. What fclones Actually Detects
fclones identifies files with identical content.
The filenames and locations do not have to be identical.
For example:
/mnt/NAS/Entertainment/Movie.mkv
/mnt/NAS/Media/Old-Movie-Copy.mkv
can be reported as duplicates if their file contents are identical.
This makes content-based comparison considerably more useful than simply searching for duplicate filenames.

11. Same Media ≠ Identical File
There is an important limitation.
These two files may represent exactly the same movie:
Movie.1080p.H264.mkv
Movie.1080p.H265.mkv
but they are not identical files.
Likewise:
Movie.mkv
Movie.mp4
may contain the same underlying media while having different container structures or encodes.
A standard fclones comparison will correctly treat these as different files.
Detecting semantically duplicate media content requires a different layer of analysis, potentially involving:
    • title matching
    • duration
    • resolution
    • video codec
    • audio tracks
    • metadata
    • perceptual or video fingerprinting
This project currently focuses on exact duplicate media files while providing a foundation for more advanced media-content analysis.

Safety
This workflow deliberately uses:
fclones group
for analysis.
Avoid automatically following a duplicate scan with destructive operations such as:
fclones remove
fclones move
Review the generated report before making changes to a media library.
For important or irreplaceable media, maintain a verified backup before performing cleanup operations.

Use Cases
This workflow can be useful for:
    • Synology media libraries
    • home servers
    • homelabs
    • movie and TV libraries
    • family media collections
    • download archives
    • media spread across multiple NAS volumes
    • Linux-accessible SMB storage
It is not tied to one particular NAS layout.

Tested Environment
The workflow was developed and tested with:
Component	Environment
Client	Debian Linux
NAS	Synology DSM
Network storage	SMB / CIFS
Duplicate detection	fclones
Storage layout	Multiple SMB shares and NAS volumes
Scan mode	Non-destructive analysis

Future Improvements
Potential extensions to the project include:
    • audio content scanning
    • configurable media extension profiles
    • CSV and JSON reporting
    • calculating reclaimable storage
    • sorting duplicate groups by potential savings
    • comparing specific media shares
    • detecting different encodes of the same media
    • media fingerprinting
    • recurring cached scans
    • interactive review before cleanup

License
MIT License
Contributions and additional media-library use cases are welcome.
