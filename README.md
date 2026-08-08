\# Mimic Hub // Zero-Knowledge P2P Networking Utility



!\[Version](https://img.shields.io/badge/Version-1.1.6-45F0D1?style=for-the-badge)

!\[Rust](https://img.shields.io/badge/Core-Rust-black?style=for-the-badge\&logo=rust)

!\[Tauri](https://img.shields.io/badge/UI-Tauri-blue?style=for-the-badge\&logo=tauri)

!\[License](https://img.shields.io/badge/License-Proprietary-FF3366?style=for-the-badge)



\*\*Mimic Hub\*\* is a lightweight, high-performance L4/L7 multiplexer engineered to bypass Carrier-Grade NAT (CGNAT) and strict firewalls. It establishes raw, E2E encrypted network tunnels for local multiplayer games and self-hosted environments without requiring port forwarding.



Unlike traditional VPNs (like ZeroTier or Hamachi) that create bloated virtual network adapters, or reverse proxies (like Ngrok) that suffer from heavy routing overhead, Mimic Hub operates entirely in user-space as an invisible bridge. 



\---



\## System Architecture



Mimic Hub is designed under a strict \*\*Zero-Knowledge (ZK)\*\* and \*\*Zero-Trust\*\* paradigm. The infrastructure is divided into three isolated layers:



\### 1. The Native Client (Tauri + Rust)

The desktop application is built with Tauri, utilizing web technologies for the UI but deferring 100% of the networking logic to a highly optimized Rust core. This guarantees minimal memory footprint (\~20MB) and near-native execution speed.

\*   \*\*Local Proxy Interception:\*\* The client binds to local ports (e.g., `127.0.0.1:25565`) and intercepts game traffic, wrapping it securely before it leaves the host machine.



\### 2. The Cloud Relay (Vultr / Linux)

A centralized multiplexer built in Rust. It serves exclusively as a fallback mechanism. 

\*   \*\*Mathematical Blindness:\*\* The server routes packets but possesses zero capability to decrypt them. It relies on ephemeral Session IDs and volatile RAM atomic counters for rate-limiting, ensuring no persistent logs or IP tables are written to disk.



\### 3. The Edge Analytics (Cloudflare)

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


\---



\## Latency Mitigation Engineering



Routing game traffic (especially UDP-based classic shooters or heavy TCP simulators like Foundry VTT) across the internet introduces severe latency spikes. We solved this through a three-pronged engineering approach:



\### I. Aggressive UDP Hole Punching (Direct P2P)

By default, the client attempts to establish a direct Peer-to-Peer connection using UDP Hole Punching. If successful, latency drops to physical network minimums (0 added ms). The cloud relay is only engaged if symmetric NATs prevent direct handshakes.



\### II. KCP Protocol Implementation over UDP

TCP guarantees delivery but introduces "Head-of-line blocking," which is fatal for fast-paced game states. Raw UDP is fast but drops packets, disconnecting players. 

\*   \*\*The Solution:\*\* We implemented \*\*KCP\*\* (A Fast and Reliable ARQ Protocol) over UDP. KCP sacrifices roughly 10% to 20% of bandwidth efficiency in exchange for reducing average latency by 30% to 40% compared to TCP, providing reliable delivery without the blocking delays.



\### III. Cryptographic Sliding Window (Replay Attack Mitigation)

Applying ChaCha20Poly1305 AEAD encryption to every packet secures the payload, but malicious actors or network loops can resend valid packets (Replay Attacks). In a naive implementation, the CPU would waste cycles attempting to decrypt these duplicate packets, causing local DoS and severe "spike lag."

\*   \*\*The Solution:\*\* We engineered a bit-masking \*\*Sliding Window\*\* filter. Before a packet reaches the decryption phase, its sequence number is evaluated in an $O(1)$ constant-time operation. If the packet is out of the acceptable latency window or has already been seen, it is silently dropped. This protects the decryption thread from CPU starvation.



\---



\## Cryptography \& Security Model



Mimic Hub does not rely on "security by obscurity" or centralized authentication servers.



1\.  \*\*Symmetric E2E Encryption:\*\* All payloads are encrypted using \*\*ChaCha20Poly1305\*\*. 

2\.  \*\*No Key Escrow:\*\* Keys are generated locally. The room password is hashed locally via \*\*SHA-256\*\*, and the resulting hash serves as the symmetric key for the tunnel. The cloud relay never receives the password or the derived key.

3\.  \*\*No Virtual Network Adapters:\*\* Mimic Hub operates solely at the application layer. It does not install root certificates, TAP adapters, or modify Windows registry routing tables. When you close the app, your network topology remains pristine.



\---



\## Getting Started



\*(Note: Production builds are currently managed via our proprietary release pipeline. To request a beta key or access the compiled binaries, please visit our community channels.)\*



\### System Requirements

\*   \*\*OS:\*\* Windows 10 / 11 (64-bit)

\*   \*\*Disk Space:\*\* < 50 MB

\*   \*\*Permissions:\*\* Standard User (No Administrator privileges required for Guest connections).



\---



\## EULA \& Liability



Mimic Hub is an infrastructure tool. We provide the encrypted pipe; you provide the water. We strictly prohibit the use of this software to bypass DRM, pirate software, or distribute illegal content. Due to our Zero-Knowledge architecture, we cannot monitor payloads, but we cooperate fully with infrastructure bans if network abuse is detected.



Read our full \[Privacy Policy \& Terms of Service](https://mimic-hub.net/privacy).


## Data Volatility & Memory Topology

Mimic Hub does not utilize traditional relational databases (SQL) or persistent NoSQL document stores for routing telemetry. To mathematically guarantee our Zero-Knowledge policy, all connection states exist strictly as ephemeral memory allocations in Rust.

### 1. In-Memory State Handling (The Relay Core)
The Vultr multiplexer utilizes heavily optimized, thread-safe `DashMaps` and `Atomic` counters. Data lifecycle is strictly tied to the active session:

*   **Session Topology:** `Map<RoomCode, Arc<RelayRoom>>`
*   **Payload Constraints:** We store the `HWID` (Hardware ID - hashed) and local atomic counters (`AtomicU64`) to measure bandwidth consumption for server stability.
*   **Volatile Death:** The instant a Host terminates the connection, or the UDP Hole Punching timeout is reached, the `Drop` trait in Rust deallocates the `RelayRoom` struct. No IPs, payloads, or session durations are ever flushed to disk storage. 

### 2. Separation of Concerns: Identity vs. Infrastructure
To maintain our infrastructure blind to Personally Identifiable Information (PII), we physically decouple identity from routing:

*   **Financial Data:** Handled exclusively by **LemonSqueezy** (Merchant of Record). Mimic Hub servers never touch credit cards or billing addresses.
*   **Cryptographic Licensing:** Handled by **Keygen.sh** via asymmetric signatures.
*   **The Bridge (Ephemeral Tokens):** When a user launches Mimic Hub, the client validates the license with Keygen.sh and receives an Ephemeral Session Token. The Vultr relay only reads this temporary token to authorize bandwidth access. Vultr does not know your email, your name, or your Discord tag.

### 3. Edge Analytics (Cloudflare)
Aggregate, anonymized telemetry (e.g., "Total Bandwidth Transferred Globally") is processed at the edge via Cloudflare Workers strictly for financial threshold monitoring (preventing DDoS billing spikes). This data is completely divorced from individual user sessions.

### Zero-Knowledge Memory Lifecycle

The relay server operates blindly. Payloads are encrypted client-side, and routing states exist only in volatile RAM.

```mermaid
sequenceDiagram
    participant Host
    participant VultrRelay as Vultr Core (RAM)
    participant Guest

    Host->>VultrRelay: Bind Room Code (TCP/UDP)
    Note over VultrRelay: Allocate Ephemeral DashMap
    Guest->>VultrRelay: Submit Session Token
    VultrRelay-->>Guest: Validate Token (No PII Stored)
    
    rect rgb(0, 40, 20)
    Note over Host, Guest: ChaCha20Poly1305 E2E Encrypted Tunnel
    Guest->>VultrRelay: Encrypted Payload [Nonce + Ciphertext]
    VultrRelay->>Host: Forward Blindly (O(1) Routing)
    end
    
    Host->>VultrRelay: Terminate Session / Close App
    Note over VultrRelay: Atomic Drop() <br> RAM Allocated Struct Destroyed
    VultrRelay-->>Guest: Connection Severed
