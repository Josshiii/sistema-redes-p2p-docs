# Mimic Hub // Zero-Knowledge P2P Networking Utility
*Architecture & Topology Documentation*

![Version](https://img.shields.io/badge/Version-1.1.6-45F0D1?style=for-the-badge)
![Rust](https://img.shields.io/badge/Core-Rust-black?style=for-the-badge&logo=rust)
![Tauri](https://img.shields.io/badge/UI-Tauri-blue?style=for-the-badge&logo=tauri)
![License](https://img.shields.io/badge/License-Proprietary-FF3366?style=for-the-badge)

**Mimic Hub** is a lightweight, high-performance L4/L7 multiplexer engineered to bypass Carrier-Grade NAT (CGNAT) and strict firewalls. It establishes raw, E2E encrypted network tunnels for local multiplayer games and self-hosted environments without requiring port forwarding.

Unlike traditional VPNs (like ZeroTier or Hamachi) that create bloated virtual network adapters, or reverse proxies (like Ngrok) that suffer from heavy routing overhead, Mimic Hub operates entirely in user-space as an invisible bridge. 

---

## System Architecture

Mimic Hub is designed under a strict **Zero-Knowledge (ZK)** and **Zero-Trust** paradigm. The infrastructure is divided into three isolated layers:

### 1. The Native Client (Tauri + Rust)
The desktop application is built with Tauri, utilizing web technologies for the UI but deferring 100% of the networking logic to a highly optimized Rust core. This guarantees minimal memory footprint (~20MB) and near-native execution speed.
* **Local Proxy Interception:** The client binds to local ports (e.g., `127.0.0.1:25565`) and intercepts game traffic, wrapping it securely before it leaves the host machine.

### 2. The Cloud Relay (Vultr / Linux)
A centralized multiplexer built in Rust. It serves exclusively as a fallback mechanism. 
* **Mathematical Blindness:** The server routes packets but possesses zero capability to decrypt them. It relies on ephemeral Session IDs and volatile RAM atomic counters for rate-limiting, ensuring no persistent logs or IP tables are written to disk.

### 3. The Edge Analytics (Cloudflare)
All telemetry regarding acquisition and updates is handled at the edge. The system uses Cloudflare Workers and R2 to distribute updates (`latest.json`) dynamically, ensuring users are always on the same protocol version without polluting the core server with update requests.

### 🌐 Asymmetric Routing Protocol

Mimic Hub employs an aggressive fallback routing protocol to guarantee connection regardless of the host's ISP restrictions.

```mermaid
graph TD
    %% Styling for dark/hacker aesthetic natively supported by GitHub
    classDef edge fill:#111,stroke:#45F0D1,stroke-width:1px,color:#fff
    classDef core fill:#000,stroke:#FF3366,stroke-width:2px,color:#fff
    classDef relay fill:#111,stroke:#F5A623,stroke-width:1px,color:#fff
    classDef success fill:#052e16,stroke:#10B981,stroke-width:2px,color:#10B981

    A[Guest Instance]:::edge -->|1. Join Request| B(Cloudflare Edge)
    B -->|2. Verify License| C{Keygen.sh Validation}
    
    C -->|Invalid / Cooldown| X[Drop Connection]:::core
    C -->|Valid| D[Retrieve Ephemeral Token]:::edge
    
    D -->|3. Initiate P2P Handshake| E{NAT Firewall Check}
    
    E -->|Open / Full Cone NAT| F[Direct UDP Hole Punching]:::success
    E -->|Symmetric / CGNAT| G[Vultr Fallback Multiplexer]:::relay
    
    F -->|Raw Encrypted Packets| H((Host Local Port)):::success
    G -->|KCP/TCP Proxy Tunnel| H
