# PeerRadar

A serverless, peer-to-peer terminal chat app built entirely on raw sockets,
with zero third-party dependencies. Every running instance broadcasts
itself over UDP and listens for others nearby, like a radar sweep. There
is no central server and no single point of failure. Discover a peer,
send them an invite, and once accepted, chat live over a direct TCP
connection.

Beyond basic messaging, PeerRadar supports multi-chat (hold several live
conversations at once without losing any of them), group chat (create a
room with an auto-generated join code, or join one with a code someone
shares with you), and file transfer (send images or files directly
between peers, safely encoded so binary data never corrupts the
protocol). The entire interface runs in the terminal, using arrow-key
navigation and live typing, with no GUI framework involved.

Built for the **Zero Dependency Hackathon**, **Track C (Web & Network)**.
Every piece, including discovery, connections, the wire protocol, the
terminal UI, and even Base64 encoding, is hand-written using nothing but
POSIX sockets and the C++ standard library, with every substitution for
a typical third-party package documented and justified in `STDLIB.md`.

---

## Platform setup

This project uses raw POSIX sockets and `termios`, which are not
natively supported on Windows. Requires Linux, macOS, or WSL2 on
Windows.

### Windows, via WSL2 (required, roughly 20-30 minutes one time)
1. Open PowerShell **as Administrator** and run:
   ```
   wsl --install
   ```
2. Restart your computer when prompted.
3. On reboot, Ubuntu finishes installing and asks you to create a
   Linux username and password.
4. Open the "Ubuntu" app (or type `wsl` in any terminal) to get a
   real Linux shell.
5. Install build tools inside WSL:
   ```
   sudo apt update
   sudo apt install build-essential g++ make -y
   ```
6. In VS Code, install the **WSL** extension (by Microsoft), then run
   **"WSL: Connect to WSL"** from the Command Palette. The bottom-left
   corner should now read "WSL: Ubuntu."

Your Windows files stay untouched. WSL is a separate, sandboxed Linux
environment and can be fully removed later with `wsl --unregister Ubuntu`
if you ever want to undo it.

### macOS
```
xcode-select --install
```

### Linux
Most distros already have `g++`/`make`. If not, on Debian/Ubuntu:
```
sudo apt update
sudo apt install build-essential g++ make -y
```

---

## Why this fits Track C

Track C explicitly names "chat over raw TCP" as an example project, and
requires four things:

| Requirement | How PeerRadar satisfies it |
|---|---|
| Handles concurrent connections without a framework | Multiple simultaneous chats (1:1 and group), each backed by its own thread and open TCP connection, with no networking library |
| Speaks the protocol correctly | A self-defined wire format for discovery, invites, chat, and group messages, documented below |
| Uses only stdlib networking | UDP broadcast for discovery, raw TCP sockets for chat, both POSIX, zero third-party networking libraries |
| Documents its concurrency model honestly | See [Concurrency model](#concurrency-model) below |

---

## Architecture

Every running instance of the program is identical. There is no central
server. Two peers talking to each other looks like this:

<img width="2720" height="1920" alt="peerradar_architecture" src="https://github.com/user-attachments/assets/10879238-2213-4ceb-9367-9bd714473cbd" />


| Module | Responsibility |
|---|---|
| Discovery | Broadcasts this peer's id and tagline over UDP, listens for other peers, drops entries that go quiet |
| Connection | Opens and accepts TCP connections for invites and chat |
| Protocol layer | Defines and parses wire messages: invite, accept, decline, chat, group join, group message, file transfer |
| Chat state | Tracks open 1:1 chats, group membership, unread counts |
| Terminal UI | Raw-mode keyboard input via `termios`, arrow-key menus, chat rendering |
| Group | Coordinator-based group membership and message fan-out |
| File transfer / Base64 | Chunked file sending over an open connection, with binary-safe encoding |

---

## Interaction flow

1. **Identity.** Enter an id and short tagline on startup.
2. **Discovery.** See a live list of nearby peers, refreshed continuously via UDP broadcast.
3. **Invite.** Arrow-select a peer, press enter to send a chat invite.
4. **Accept or decline.** The receiving peer sees an arrow-select prompt.
5. **Chat.** On accept, a live TCP-backed chat opens. Type to send, `/back` to return to the peer list, `/quit` to exit the whole program.
6. **Multi-chat.** Open chats with several peers at once. Switching away from one doesn't close it, and unread messages are tracked.
7. **Group chat.** Create a group (auto-generates a short join code) or join one by entering a code. Messages fan out to every member.
8. **File transfer.** Send a file to whoever you're chatting with; it's chunked, Base64-encoded, and reassembled on arrival.

---

## Concurrency model

**Thread-per-connection.** Every open chat connection runs its own
background thread listening for incoming messages, while the main
thread drives whatever menu or chat view you're actively looking at.
Shared state (the chat log, the peer connection table) is protected
with a mutex.

This was chosen over a single-threaded event loop (`select`/`poll`/
`epoll`) for simplicity: at the scale of a handful of simultaneous
chats, thread-per-connection is easy to reason about and debug. An
event loop would be more efficient at large connection counts, which
is not this project's use case.

**Known trade-off, stated honestly:** incoming invites are handled one
at a time. A second invite arriving while you're deciding on the first
will simply wait its turn rather than interrupting you. This was a
deliberate scope decision for a 72-hour build.

---

## Zero-dependency substitutions

See `STDLIB.md` for the full list with rationale. Highlights:

| Instead of | We use |
|---|---|
| A networking framework (Boost.Asio, libuv) | Raw POSIX sockets |
| A service-discovery library (Bonjour/mDNS bindings) | Hand-rolled UDP broadcast |
| A TUI library (ncurses, blessed) | Raw `termios` and manual ANSI escape codes |
| A JSON library (nlohmann/json) | A small hand-written wire message format |
| Python's `base64` / npm's `base64-js` | A hand-written Base64 encoder and decoder |
| An async library (libevent, libuv) | `std::thread` and `std::mutex` from the standard library |

---

## Project layout

```
peer-radar-chat/
|-- README.md            this file
|-- STDLIB.md              every stdlib-for-package substitution, with rationale
|-- Makefile                one command to a runnable artifact
|-- LICENSE                  MIT
|-- .zero-dep.toml            track letter and one-line pitch
|-- deps-proof.txt             output proving zero third-party runtime deps
|-- src/
|   |-- main.cpp                wires every module into the running program
|   |-- discovery.h/.cpp          UDP broadcast peer discovery
|   |-- connection.h/.cpp         TCP connect, accept, send, receive
|   |-- protocol.h/.cpp           wire message format, encode and decode
|   |-- chat_state.h/.cpp         open chats, groups, unread tracking
|   |-- terminal_ui.h/.cpp        termios raw mode, arrow menus, rendering
|   |-- group.h/.cpp              group membership and fan-out messaging
|   |-- file_transfer.h/.cpp      chunked file send and receive
|   |-- base64.h/.cpp             binary-safe encoding for file transfer
|-- tests/
    |-- stage1_discovery_test.cpp
    |-- stage_protocol_test.cpp
    |-- stage2_handshake_test.cpp
    |-- stage3_chatstate_test.cpp
    |-- stage_terminalui_test.cpp
    |-- stage5_group_test.cpp
    |-- stage_base64_test.cpp
    |-- stage7_filetransfer_test.cpp
```

---

## Build and run

```
make
./peer_radar
```

Run two or more instances, in separate terminals or on separate
machines on the same network, to see discovery and chat in action.

Run the automated module tests:
```
make test
```

Generate the dependency proof:
```
make deps-proof
```

---

## Submission requirements checklist

- [ ] Public GitHub repo, OSI-approved license (MIT, included)
- [x] Builds with a single command (`make`), no multi-step setup
- [x] Empty or absent dependency manifest, verifiable in seconds
- [x] `deps-proof.txt`, command output proving zero third-party runtime deps
- [x] `README.md`, what it does, how to run it, honest current limits
- [x] `STDLIB.md`, every stdlib-for-package substitution, with rationale
- [x] `tests/`, edge cases genuinely tested, not just the happy path
- [ ] Five-minute demo video, tool working, empty manifest shown on screen
- [x] Concurrency model documented honestly
- [x] `.zero-dep.toml`, track letter and one-line pitch

## Bonus challenges, which ones we're targeting

| Challenge | Realistic for us? | Notes |
|---|---|---|
| STDLIB Log (+3) | Yes | Ten or more real substitutions, documented in `STDLIB.md` |
| Package Killer (+3) | Yes | Hand-rolled Base64 is a clean, defensible replacement for a package almost everyone reaches for |
| Single File (+5) | Not planned | Deliberately modular for code quality, which is worth five times more in scoring |
| Reproducible Build (+5) | Stretch, if time remains | Achievable with static linking and fixed compiler flags |

## Out-of-scope self-check

- Not a single-function toy. Full discovery, invite and accept, chat, multi-chat, group chat, and file transfer.
- No shelling out to externally installed tools. Everything is our own POSIX socket and termios code.
- No vendored source. Nothing copied in from another library.
- No custom cipher. No cryptography is invented; none is in scope for this project.
- README and STDLIB.md exist, and every design decision can be explained by the person who built it.
- Laptop-friendly. No special hardware, no GUI framework, no proprietary toolchain.
- No external running service required. Fully peer-to-peer, nothing to stand up beforehand.

## Known limitations

- The wire protocol's `|` delimiter means an id, tagline, or message
  containing `|` will break parsing. Not escaped, for time reasons.
- Group chat relies on a single coordinator (whoever created the
  group). If the coordinator disconnects, the group does not currently
  elect a new one.
- No encryption. All traffic is sent as plain text or Base64 on the
  local network, which is fine for a LAN demo and not something we
  would call production-secure.

## Track

**Track C, Web & Network**

## Team

*(fill in before submission)*

## License

MIT. See `LICENSE`.
