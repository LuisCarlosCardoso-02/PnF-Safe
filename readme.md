<div align="center">

# 🔐 P&F Safe

**A secure, encrypted vault for your most sensitive data — built in Rust.**

![Rust](https://img.shields.io/badge/Rust-2021-orange?logo=rust&logoColor=white)
![Status](https://img.shields.io/badge/status-in%20development-yellow)
![License](https://img.shields.io/badge/license-see%20LICENSE-blue)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen)

[Features](#-features) •
[Architecture](#-architecture) •
[Getting Started](#-getting-started) •
[Project Structure](#-project-structure) •
[Roadmap](#-roadmap) •
[Contributing](#-contributing)

</div>

---

## 📖 About

**P&F Safe** is a secure vault application written in Rust. It is designed around a simple idea: your data should be encrypted before it ever touches the disk, and only you should be able to unlock it.

> ⚠️ **Status:** This project is in early development. APIs, file formats, and storage layouts may change without notice. Do not rely on it for critical data yet.

## ✨ Features

- 🔒 **Strong encryption** — data is encrypted at rest through a dedicated encryption module
- 🗄️ **Vault storage** — secrets are organised and managed inside a protected vault
- 🛡️ **Security layer** — centralised handling of authentication and access control
- 💾 **Persistent database** — encrypted data is stored and retrieved through a dedicated database layer
- 🖥️ **Frontend** — a user interface to interact with your vault
- 🧪 **Tested** — encryption/decryption round-trip tests included

## 🏗️ Architecture

P&F Safe is split into clearly separated modules, each with a single responsibility:

```
┌─────────────┐
│  Frontend   │  User interface
└──────┬──────┘
       │
┌──────▼──────┐
│ Application │  Orchestrates everything
└──────┬──────┘
       │
┌──────▼──────┐     ┌────────────┐
│    Vault    │────▶│  Security  │  Auth & access control
└──────┬──────┘     └────────────┘
       │
┌──────▼──────┐     ┌────────────┐
│ Encryption  │     │  Database  │  Persistence
└─────────────┘     └────────────┘
```

For the reasoning behind key design choices, see
[`documentation/architecture-decisions.md`](documentation/architecture-decisions.md).

## 🚀 Getting Started

### Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) (latest stable) and Cargo
- [Node.js](https://nodejs.org/) and npm (for the frontend)

### Installation

```bash
# Clone the repository
git clone https://github.com/<your-username>/PnF_Safe.git
cd PnF_Safe

# Build the project
cargo build --release
```

### Running

```bash
cargo run
```

### Running the frontend

```bash
cd src/frontend
npm install
npm start
```

### Running the tests

```bash
cargo test
```

## 📁 Project Structure

```
PnF_Safe
├── Cargo.toml                 # Rust package manifest
├── documentation/
│   ├── architecture-decisions.md
│   └── readme.md
└── src/
    ├── main.rs                # Entry point
    ├── application/           # Application logic and orchestration
    ├── vault/                 # Vault management
    ├── encryption/            # Encryption and decryption
    ├── security/              # Authentication and access control
    ├── database/              # Data persistence
    ├── frontend/              # User interface
    └── tests/                 # Encryption/decryption tests
```

## 🔐 Security

Security is the core goal of this project, but it has **not been independently audited**.

- Never commit secrets, keys, or vault files to version control
- Use a strong master password
- If you discover a vulnerability, please report it privately rather than opening a public issue

## 🗺️ Roadmap

- [x] Project structure and module layout
- [ ] Core encryption and decryption
- [ ] Vault creation and unlocking
- [ ] Encrypted database storage
- [ ] Authentication and access control
- [ ] Frontend interface
- [ ] Comprehensive test coverage
- [ ] Security review

## 🤝 Contributing

Contributions, ideas, and feedback are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m "Add amazing feature"`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

Distributed under the license found in the [LICENSE](LICENSE) file.

## 👤 Author

**Luis** — [GitHub](https://github.com/LuisCarlosCardoso-02)

---

<div align="center">

Made with 🦀 and a healthy dose of paranoia.

</div>