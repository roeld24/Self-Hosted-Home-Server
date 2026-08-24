# Self-Hosted Home Media Server

## Project Overview

This project consists of a self-hosted home media server built from an old PC and running on Ubuntu Server, Docker, and a collection of specialized open-source applications.

The objective was to repurpose unused hardware into a centralized media platform capable of automatically discovering, downloading, organizing, and streaming movies and TV shows. The system was also extended with Live TV functionality and automatic subtitle management.

The project was designed around a modular architecture in which every major service runs inside its own Docker container. This provides process isolation, simplified deployment, independent configuration, and easier maintenance.

The main software components are:

- **Ubuntu Server**: base operating system
- **CasaOS**: web-based server and application management interface
- **Docker**: containerization platform
- **qBittorrent**: download client
- **Prowlarr**: indexer management and integration layer
- **Sonarr**: TV series automation
- **Radarr**: movie automation
- **Jellyfin**: media server and streaming platform
- **OpenSubtitles**: subtitle integration
- **Live TV**: television streaming through Jellyfin

The final system provides an automated media pipeline in which content can be discovered, downloaded, imported, organized, enriched with metadata and subtitles, and finally streamed through Jellyfin.

---

# 1. Hardware Preparation

The project started with an old PC that was no longer being used as a conventional desktop computer.

Instead of installing a desktop operating system, the machine was repurposed as a dedicated server. This approach reduces unnecessary resource consumption and allows the system to operate continuously without requiring a monitor, keyboard, or graphical desktop environment.

The server is intended to remain powered on and accessible through the local network.

The hardware does not need to be particularly powerful for the core services because most of the applications are lightweight. The main hardware requirements depend more on storage capacity, network connectivity, and whether Jellyfin needs to perform real-time transcoding.

---

# 2. Installing Ubuntu Server

The first step was installing Ubuntu Server as the underlying operating system.

Ubuntu Server was selected because it provides:

- strong Docker compatibility
- SSH support
- extensive documentation
- a large software ecosystem
- long-term stability
- good support for server administration
- low overhead compared with a desktop environment

After installation, the machine was configured as a network server and assigned a predictable local IP address.

The server was then accessible through SSH, allowing administration from another computer without directly interacting with the server hardware.

Example:

```bash
ssh username@192.168.1.208
```

Using SSH is particularly useful for a headless server because all administration can be performed remotely through the terminal.
The first administration tasks included updating the operating system and installing the required packages.

Example:

```bash
sudo apt update && sudo apt upgrade -y
```

<img width="960" height="504" alt="image" src="https://github.com/user-attachments/assets/6b21a27a-9a93-4a97-8ff7-9d8ba1cb6042" />


---

# 3. CasaOS

After Ubuntu Server was installed, CasaOS was introduced as the web-based management layer.

CasaOS provides a graphical interface for managing the server, applications, storage, and Docker-based services. It simplifies tasks that would otherwise require extensive command-line configuration.

CasaOS does not replace Docker. Instead, it operates on top of the existing Docker infrastructure and provides a convenient interface for deploying and managing applications.

This was particularly useful during the initial setup because the system consisted of several independent services that needed to be deployed and configured.

<img width="960" height="504" alt="image" src="https://github.com/user-attachments/assets/399e7cc0-11f5-44e3-a5c9-09bd3c6d5808" />


---

# 4. Docker Architecture

The central technology behind the project is Docker. Rather than installing every application directly into Ubuntu Server, each service runs inside its own Docker container.

This creates a clear separation between the operating system and the applications:

```text
Ubuntu Server
    |
    +-- Docker
         |
         +-- CasaOS
         +-- qBittorrent
         +-- Prowlarr
         +-- Sonarr
         +-- Radarr
         +-- Jellyfin
```

Each application is therefore treated as an independent service. This provides several advantages:

- Application isolation
- Easier deployment
- Simpler updates
- Easier replacement of applications
- Reduced dependency conflicts
- Independent configuration
- Easier troubleshooting
- Easier migration to another machine

### Docker Images
A Docker image is a packaged template containing everything required to run an application. An image can contain:
- Application binaries
- Libraries
- Dependencies
- Default configuration
- Startup instructions
- Required runtime components

For example, a Sonarr Docker image contains the software required to execute Sonarr. The image itself is not the running application; it is the template used to create a container.

### Docker Containers
A container is a running instance of a Docker image.

```text
Sonarr Docker image
        |
        v
Sonarr container
```

The same principle applies to every other service. Because the applications run in separate containers, a problem inside one application does not normally affect the runtime environment of the others.

---

# 5. Persistent Storage

One of the most important parts of the Docker architecture is persistent storage.

Containers are designed to be disposable. If a container is deleted and recreated, its internal filesystem should not be relied upon for permanent data. For this reason, important configuration files and media files were stored on the Ubuntu host and mounted into the appropriate containers.

The media storage was organized under `/DATA/Media`:

```text
/DATA/Media
    |
    +-- TV Shows
    |
    +-- Movies
```

The same directories are then mounted inside the relevant containers. For Jellyfin, for example:

- **Host**: `/DATA/Media`
- **Container**: `/Media`

This means Jellyfin can access the files without the files being physically stored inside the container. The same principle was applied to Sonarr, Radarr, and qBittorrent.

This separation between application runtime and persistent storage is one of the most important design decisions of the project.

---

# 6. qBittorrent

qBittorrent is the download engine used by the system. Its role is to perform downloads requested by Sonarr and Radarr.

qBittorrent does not decide which movies or TV episodes should be downloaded. Instead, it receives download tasks from the automation applications and executes them.

The basic workflow is:

```text
Sonarr or Radarr
        |
        v
   qBittorrent
        |
        v
     Download
        |
        v
Download directory
```

Dedicated directories were used to keep different download categories separated:

```text
/downloads
    |
    +-- Sonarr
    |
    +-- Radarr
```

This makes it easier for Sonarr and Radarr to identify and import completed downloads.

<img width="960" height="504" alt="image" src="https://github.com/user-attachments/assets/ded1c6ce-1dae-4958-b609-1dab485badb6" />


---

# 7. Prowlarr

Prowlarr is the indexer management component of the system. Its purpose is to centralize the configuration and management of indexers used by applications such as Sonarr and Radarr.

Without Prowlarr, each application would have to be configured independently with the same indexers. Prowlarr acts as an intermediary and exposes the configured indexers to the automation applications.

The result is a more centralized and maintainable architecture. When an indexer needs to be added, removed, or modified, the change can be handled centrally through Prowlarr rather than duplicated across multiple applications.

Prowlarr communicates with Sonarr and Radarr through their APIs.

---

# 8. Sonarr

Sonarr is responsible for automating TV series management.

Instead of manually searching for every episode, a user can add a series to Sonarr and define the desired quality and language preferences. Sonarr then monitors the series and searches for available releases through the configured indexers. Once a suitable release is found, Sonarr sends the download request to qBittorrent.

The complete workflow is:

```text
User selects a TV series
        |
        v
      Sonarr
        |
        v
Search available releases
        |
        v
    Prowlarr
        |
        v
     Indexers
        |
        v
Release selected
        |
        v
   qBittorrent
        |
        v
    Download
        |
        v
Sonarr imports and organizes the file
        |
        v
TV Shows library
```

Sonarr can automatically manage:
- Series tracking
- Season and episode information
- Release searching
- Quality selection
- Download initiation
- Completed download detection
- File importing
- File renaming
- Directory organization

For example, a series can be organized as:

```text
TV Shows/
└── The Sopranos/
    ├── Season 01/
    │   ├── The Sopranos - S01E01.ext
    │   ├── The Sopranos - S01E02.ext
    │   └── ...
    └── Season 02/
        └── ...
```

This predictable structure is particularly useful for Jellyfin.

---

# 9. Radarr

Radarr performs a similar role to Sonarr, but for movies.

A movie is added to Radarr and the application monitors the configured indexers for a suitable release. When a release is found, Radarr sends a download request to qBittorrent.

The general process is:

```text
User selects a movie
        |
        v
      Radarr
        |
        v
    Prowlarr
        |
        v
     Indexers
        |
        v
Release selected
        |
        v
   qBittorrent
        |
        v
    Download
        |
        v
Radarr imports and organizes the file
        |
        v
 Movies library
```

Radarr can automatically handle:
- Movie monitoring
- Release searching
- Quality selection
- Language preferences
- Download management
- File importing
- File renaming
- Directory organization

The movie library can therefore be structured like:

```text
Movies/
├── Movie A/
│   └── Movie A.ext
├── Movie B/
│   └── Movie B.ext
└── Movie C/
    └── Movie C.ext
```

Radarr was also configured to prefer Italian-language releases where available. This reduces the probability of automatically obtaining releases with English audio when Italian content is preferred.

---

# 10. API Integration

A major part of the system is the communication between applications.

The services are not isolated from each other from a functional perspective. They communicate over the Docker network using HTTP-based APIs:

- `Prowlarr` $\leftrightarrow$ `Sonarr`
- `Prowlarr` $\leftrightarrow$ `Radarr`
- `Sonarr` $\leftrightarrow$ `qBittorrent`
- `Radarr` $\leftrightarrow$ `qBittorrent`

Each service performs a specific role:
- **Prowlarr**: Indexers
- **Sonarr**: TV series
- **Radarr**: Movies
- **qBittorrent**: Downloads
- **Jellyfin**: Media playback

This is effectively a small distributed application composed of multiple specialized services.

---

# 11. File Permissions

One of the practical challenges during deployment was Linux and Docker filesystem permissions.

Docker containers may run processes under a particular user or group. If the host directory is owned by another user and does not provide sufficient permissions, an application may fail to read or modify the files.

A typical failure scenario is:

```text
qBittorrent downloads file
        |
        v
Sonarr attempts import
        |
        v
 Permission denied
```

The problem was resolved by configuring the ownership and permissions of the relevant directories so that the containerized applications could access the required files.

This highlights an important aspect of Docker administration: containerization provides isolation, but it does not automatically solve Linux filesystem permissions. User IDs (UID), group IDs (GID), ownership, and directory permissions must still be correctly aligned.

---

# 12. Jellyfin

Jellyfin is the media server and primary playback platform.

Its role is different from Sonarr and Radarr. Sonarr and Radarr manage the acquisition and organization of media, while Jellyfin manages the final media library and provides the interface used for playback.

Once Sonarr and Radarr have organized the files, Jellyfin scans the configured directories and identifies the content.

Jellyfin provides:
- Movies & TV shows cataloging
- Seasons and episodes management
- Metadata & artwork (posters, descriptions, cast, genres)
- Media streaming and transcoding
- Subtitle support

Instead of presenting the raw filesystem to the user, Jellyfin transforms it into a structured media library. The server can then be accessed through:
- Web browsers
- Smartphones & tablets
- Desktop computers
- Smart TVs & compatible Jellyfin streaming clients

---

# 13. Metadata Management

Jellyfin uses metadata providers to identify the media stored on the server.

For example, a file such as `The.Sopranos.S01E01.1080p.mkv` can be automatically matched and enriched with:
- The correct series, season, and episode
- Episode title & description
- Actor credits & character lists
- High-resolution posters and backgrounds
- Genre classifications and ratings

This transforms an ordinary collection of files into a proper media library while preserving a clean, simple filesystem underneath.

---

# 14. Subtitle Integration

OpenSubtitles was integrated with Jellyfin to simplify subtitle management.

The objective was to avoid having to manually search for subtitles for every episode or movie. Jellyfin can query the subtitle provider and download matching subtitles based on the media metadata. Italian subtitles were prioritized according to the configured language preferences.

This makes subtitles available directly during playback without requiring manual downloads.

---

# 15. Language and Quality Configuration

The media automation applications were configured with specific language and quality preferences.

For example, Radarr can be configured to prefer Italian-language releases rather than simply accepting the first release returned by an indexer. Quality profiles can also be used to define preferred characteristics:
- Resolution (e.g., 1080p, 4K)
- Codec (e.g., H.264, HEVC/H.265)
- Source quality (e.g., WEB-DL, BluRay)
- Audio language & multi-audio tracks
- Custom release profile scorings

The objective is not simply to download something, but to automatically select a release that satisfies strict quality and language criteria.

---

# 16. Live TV

In addition to the on-demand media library, Jellyfin was configured to support Live TV.

Jellyfin can integrate with an appropriate Live TV source such as a tuner, IPTV source, or compatible M3U playlist. This adds a second media function to the server:
- **Sonarr / Radarr**: On-demand media acquisition
- **Jellyfin**: Centralized interface for both the on-demand library and Live TV

This allows Jellyfin to act as the single media portal for the entire home infrastructure.

---

# 17. Final System Architecture

The final solution can be summarized by the responsibility of each component:

| Component | Responsibility |
| :--- | :--- |
| **Ubuntu Server** | Base operating system |
| **Docker** | Container execution and process isolation |
| **CasaOS** | Server and application management interface |
| **Prowlarr** | Indexer management and proxying |
| **Sonarr** | TV series tracking and automation |
| **Radarr** | Movie monitoring and automation |
| **qBittorrent** | Torrent download client |
| **Jellyfin** | Media library presentation and video streaming |
| **OpenSubtitles** | Automated subtitle retrieval |
| **Live TV source** | Live television feed integration |

### Complete Media Lifecycle

1. A movie or TV series is selected.
2. Sonarr or Radarr searches for suitable releases.
3. Prowlarr routes the search to configured indexers.
4. A suitable release is selected based on quality and language profiles.
5. qBittorrent performs the download.
6. Sonarr or Radarr detects the completed download.
7. The media file is imported and renamed into the correct library path.
8. Jellyfin scans the directory and retrieves metadata and artwork.
9. Subtitle services provide missing subtitles.
10. The media becomes available for instant playback.

---

# 18. Why Docker Was Used

Docker was one of the most critical architectural decisions in the project. Without Docker, every application would need to be installed directly on Ubuntu Server, which introduces several problems:
- Host filesystem bloat with unmanaged dependencies
- Potential conflicts between differing runtime versions
- High complexity when upgrading or removing services
- Difficult backup and migration processes

With Docker, each application is isolated in its own environment. If the physical host needs to be replaced, the Docker compose configurations and persistent data directories can simply be moved to the new machine and brought up immediately.

---

# 19. Network Architecture

The services communicate locally through the home network and via Docker internal networks. Each application exposes a specific port for its web UI or API.

Services are accessed via the server's private IP address (e.g., `192.168.1.208:8096` for Jellyfin). Using private IP addresses ensures the system is initially isolated from direct internet access, keeping the public attack surface at zero.

---

# 20. Cybersecurity Considerations

Because the server hosts multiple network-accessible services, cybersecurity is essential.

The primary guiding principle is **minimizing the attack surface**:
- Administrative interfaces (**CasaOS, Sonarr, Radarr, Prowlarr, qBittorrent**) must remain strictly restricted to the local network or accessible only through an encrypted VPN.
- **Jellyfin** is the only service that may realistically require external exposure for remote playback.

---

# 21. Firewall Configuration

The host runs **UFW (Uncomplicated Firewall)** under a strict default-deny policy:

```text
Default policy: Deny incoming
Required services: Allow only specific ports / subnets
```

Restricting inbound ports ensures that even if a container misconfigures its listening ports, unauthorized external machines cannot reach it.

---

# 22. SSH Hardening

SSH is a critical administrative vector and should follow standard hardening practices:
- Enforce public-key authentication
- Disable password-based logins
- Disable direct root login (`PermitRootLogin no`)
- Use a non-root administrative user with `sudo` privileges
- Restrict SSH access through UFW to known local subnets
- Monitor `/var/log/auth.log` for anomalous authentication attempts

---

# 23. Strong Authentication

Every web service must utilize a unique, high-entropy password. Reusing passwords across CasaOS, Jellyfin, the \*arr stack, and SSH must be strictly avoided.

Multi-factor authentication (MFA/2FA) should be enabled wherever supported. All API tokens generated by Sonarr, Radarr, and Prowlarr must be treated with the same confidentiality as private keys.

---

# 24. Secrets Management

Credentials, passwords, and API keys must never be committed to public Git repositories or insecure configuration files:

```bash
# BAD PRACTICE (in public repos):
API_KEY=xxxxxxxx
PASSWORD=xxxxxxxx
TOKEN=xxxxxxxx
```

Instead, use environment variable files (`.env`) excluded via `.gitignore`, Docker secrets, or restricted configuration files with `600` permissions.

---

# 25. Docker Security

Docker containers should run according to the principle of least privilege:
- Avoid the `--privileged` flag unless strictly required by hardware (e.g., GPU passthrough).
- Avoid mounting host root directories (`/` or `/etc`).
- Expose only the minimum required ports.
- Use explicit container image tags from trusted maintainers (e.g., LinuxServer.io) rather than unverified third parties.
- Periodically prune orphaned containers, dangling images, and unused volumes.

---

# 26. Container Image Security

Images should be periodically updated to patch vulnerabilities in application layers and underlying base OS packages.

For advanced implementations, static vulnerability scanners like **Trivy** can be used to scan images before updating, integrating a lightweight DevSecOps approach to container management.

---

# 27. Network Segmentation

A robust architecture isolates the media server onto a separate VLAN:

```text
Home Network
|
+-- Personal Devices (Trusted)
|
+-- IoT Devices (Untrusted)
|
+-- Server VLAN (Isolated)
```

Segregating the server from untrusted IoT gadgets mitigates lateral movement in the event of a compromised smart device on the local network.

---

# 28. Remote Access

Exposing services via direct router port-forwarding creates unnecessary risk. Remote access should be routed through secure channels:

```text
Internet ---> Reverse Proxy / VPN ---> Jellyfin / Services
```

Using a mesh VPN overlay like **Tailscale** or **WireGuard** allows seamless, encrypted remote access to internal web interfaces without publishing open ports to the global internet.

---

# 29. Reverse Proxy and HTTPS

If public access is required for Jellyfin, traffic should route through a Reverse Proxy (such as **Nginx Proxy Manager**, **Caddy**, or **Traefik**):
- Automated TLS certificate issuance and renewal via Let's Encrypt
- Enforced HTTPS encryption
- Header sanitation and rate limiting
- Domain/subdomain-based routing (e.g., `media.example.com`) instead of raw IP/port exposure

---

# 30. Monitoring and Logging

Centralized visibility helps detect failures and intrusions early:
- Failed authentication attempts and SSH logs
- Docker container health and restart loops
- Resource utilization (CPU, RAM, swap)
- Network bandwidth metrics
- SMART metrics for disk health

Tools like Netdata, Glances, or a Prometheus/Grafana stack can be added to provide dashboards and alerting.

---

# 31. Automatic Updates and Patch Management

Keeping both the OS and containers up to date is vital for system integrity:
- Unattended-upgrades for critical Ubuntu security updates
- Regular container updates using tools like Watchtower or manual compose pulls
- Pre-upgrade backups before running major application updates

---

# 32. Disk Health and Data Integrity

Given the intensive read/write operations of torrent clients and media libraries, monitoring disk health is essential.

Using `smartmontools` to check S.M.A.R.T. parameters:
- Reallocated sectors count
- Drive temperature trends
- Read/write error rates
- Power-on hours

Early detection of drive degradation prevents unexpected catastrophic data loss.

---

# 33. Backup Strategy

**RAID is not a backup.** A dedicated backup policy must protect unrecoverable assets:
- Personal documents, photos, and media configurations
- Docker volume configurations and databases

Adhere to the **3-2-1 backup strategy**:
- **3** copies of critical data
- **2** different storage media (e.g., internal pool + external USB drive)
- **1** copy located off-site (cloud storage or remote server)

---

# 34. Music Server Expansion

The ecosystem can expand into music streaming by integrating **Navidrome**:
- Docker-based deployment
- Mounts `/DATA/Media/Music`
- Integrates with Subsonic-compatible mobile apps (e.g., Symfonium, DSub)
- Low memory footprint and instant library indexing

---

# 35. Photo Backup Expansion

Deploying **Immich** provides a self-hosted alternative to proprietary cloud photo storage:
- Automatic background mobile backup
- Machine learning-based facial recognition and object tagging
- Native timeline and album browsing
- Multi-user support with isolated libraries

---

# 36. Additional Storage Solutions

As storage demands increase, the system can scale through:
- **MergerFS + SnapRAID**: Ideal for pooling mixed-size drives with parity protection for media workloads.
- **ZFS pools**: Enterprise-grade integrity, snapshots, and software RAID capabilities.
- **Dedicated NAS / DAS expansion**: Direct-attached storage enclosures over USB 3.0 / eSATA.

---

# 37. Remote Access with VPN

A dedicated VPN container or node (Tailscale/WireGuard) provides administrative access from anywhere:
- Access CasaOS, SSH, and the *arr stack remotely
- Zero open incoming ports needed on the home router
- End-to-end encrypted traffic tunneling

---

# 38. Possible Final Evolution

The architecture can scale into a complete private cloud infrastructure:
- **Jellyfin**: Video streaming
- **Navidrome**: Audio streaming
- **Immich**: Photo storage
- **Nextcloud**: Cloud storage & document collaboration
- **Pi-hole / AdGuard Home**: Network-wide DNS ad-blocking
- **Uptime Kuma**: Service uptime monitoring

---

# 39. Engineering Lessons

This build provides practical experience across multiple systems and infrastructure disciplines:

- **Linux Administration**: Filesystem management, permissions, systemd units, SSH security, shell automation.
- **Containerization**: Docker runtimes, volume bindings, network bridges, resource constraints.
- **Networking**: CIDR subnets, port mapping, internal DNS resolution, reverse proxy routing, VPN tunnels.
- **Automation**: Decoupled microservice coordination via REST APIs and webhooks.
- **DevSecOps & Hardening**: Attack surface minimization, secret management, least-privilege permission models, vulnerability remediation.

---

# 40. Final Result

The project demonstrates how decommissioned desktop hardware can be transformed into an enterprise-inspired, fully automated home media server.

Through strict separation of concerns, container isolation, declarative configurations, and defensive network posture, the system handles the entire lifecycle of digital media seamlessly from discovery to playback.
