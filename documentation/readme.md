# F&P Safe

**F&P Safe** is a local-first, privacy-focused vault designed to keep your passwords, documents, photos and videos secure in one place.

The goal is simple: your private data should remain private. Everything stored in F&P Safe is encrypted, and access to your vault requires your master password. No cloud account, no developer backdoor, and no unnecessary exposure of your personal files.

This project is currently under development.

## Features

- **Password storage** - Securely store and manage your passwords.
- **Document protection** - Keep your private documents encrypted and accessible only through F&P Safe.
- **Photo and video protection** - Store personal photos and videos without exposing their contents to the outside world.
- **Firefox autofill** - Automatically fill in saved login credentials using a dedicated browser extension. *(Currently in development.)*
- **Automatic locking** - Lock the vault after a configurable period of inactivity or when the application closes. *(Planned.)*
- **Local storage** - Keep your vault data on your own machine, without requiring cloud storage.
- **Cross-platform support** - Targeting Windows and Linux.
- **Double Layered encryption** - Having a second password for a particularly sensitive folder or file

## Project structure

```text
.
├── Cargo.lock
├── Cargo.toml
├── documentation
│   └── readme.md
├── readme.md
└── src
    ├── database
    │   └── mod.rs
    ├── encryption
    │   └── mod.rs
    ├── frontend
    ├── main.rs
    ├── security
    │   └── mod.rs
    └── vault
        └── mod.rs
```

### Module overview

| Module | Responsibility |
|---|---|
| `database` | Database operations and encrypted data storage. |
| `encryption` | Encryption and decryption functionality. |
| `frontend` | User interface and application interaction. |
| `security` | Authentication and security-related functionality. |
| `vault` | Vault management and file operations. |

## Technologies used

- **Rust** – Main programming language.
- **SQLite** – Local database for storing encrypted data and metadata.
- **XChaCha20-Poly1305** – Authenticated encryption for protecting stored data.
- **Tauri** – Desktop application framework.

Additional technologies and dependencies may be introduced as development progresses.

## Security philosophy

Security and privacy are the main priorities of F&P Safe.

The project aims to follow a few fundamental principles:

- **Encryption at rest:** Stored data remains encrypted while its not being locked at.
- **No backdoors:** There should be no developer-controlled master key or hidden recovery mechanism.
- **Local-first:** Private data stays on the user's machine.
- **Minimal exposure:** Files should only be decrypted when needed, with decrypted data kept in memory for as little time as reasonably possible.
- **Automatic locking:** The vault should lock after inactivity and when the application closes.

F&P Safe aims to make unauthorized access to a locked vault computationally infeasible when strong passwords and properly implemented cryptography are used.

No security system can guarantee absolute protection against a compromised device, malware or every possible attack.

## Supported platforms

The initial development targets are:

- Windows
- Linux

Linux packaging is planned to include `.deb`, `.rpm` and AppImage distributions.

## Development status

F&P Safe is currently under active development. Features, architecture and implementation details may change as the project evolves.

Some features listed above are planned and should not yet be considered production-ready.

## License

License information will be added once a project license has been selected.