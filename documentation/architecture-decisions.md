#  Architectural Decisions

> **Status:** Working architecture / design notes\
> **Project stage:** Early development\
> **Last updated:** 15 November 2026
>
> This document records the architectural direction discussed so far. It
> is not a claim that every feature has already been implemented or
> security-reviewed. Items marked **Proposed** should be validated
> during implementation.

## 1. Project overview

EasyVault is a local-first, privacy-focused vault for storing passwords,
documents, photos and videos.

The core objective is to keep stored data confidential while the vault
is locked, without relying on cloud storage, developer-controlled master
keys or hidden recovery backdoors.

The application is intended to run on **Windows and Linux** from a
shared codebase.

## 2. Core security principles

-   **Local-first storage:** Vault data is stored on the user's own
    machine. Cloud storage is not required.
-   **Encryption at rest:** Sensitive data remains encrypted on disk,
    including while the vault is unlocked.
-   **Selective decryption:** Unlocking the vault does not decrypt the
    contents of every stored file.
-   **No developer backdoor:** There should be no developer-controlled
    master key or hidden mechanism for bypassing the user's password.
-   **Minimal plaintext exposure:** Decrypt file contents only when
    needed, keep plaintext in memory for as short a time as reasonably
    possible, and avoid writing plaintext temporary files.
-   **Automatic locking:** Lock after a configurable period of
    inactivity and when the application closes.
-   **Established cryptography:** Use established cryptographic
    libraries and constructions rather than implementing cryptographic
    algorithms from scratch.

These goals reduce exposure but cannot guarantee protection against a
compromised operating system, malware, keyloggers, memory capture while
unlocked, or every possible forensic technique.

## 3. Locking and decryption model

EasyVault has two primary states.

### 3.1 Locked

While locked:

-   The index, directory structure, filenames and metadata are
    encrypted.
-   File contents and file chunks are encrypted.
-   The application does not have usable vault encryption keys.
-   The master password is not stored by the application.

### 3.2 Unlocked

When the user enters the master password:

1.  EasyVault derives the required key material from the password.
2.  It authenticates and decrypts the initial index.
3.  It decrypts directory information and metadata as the user
    navigates.
4.  It displays the logical folder and file structure.

**Unlocking does not decrypt all file contents.** Actual documents,
images and videos remain encrypted on disk until the user opens them.

### 3.3 Opening and closing a file

When a file is opened:

1.  EasyVault locates the relevant encrypted entity and its chunks.
2.  It obtains the required file key.
3.  It authenticates and decrypts the required content into memory.
4.  It displays or streams the content.
5.  When the file is closed, EasyVault clears application-controlled
    plaintext buffers and temporary key material where possible.

For large media, especially video, decryption should be performed
progressively rather than loading the entire file into memory.

`drop()` alone does not guarantee secure memory erasure. Sensitive
buffers and keys should use suitable zeroization techniques, such as the
Rust `zeroize` ecosystem, while recognizing that copies may exist in the
UI, operating-system caches, swap, crash dumps or the graphics stack.

## 4. Storage layout

### 4.1 On-disk layout

The current proposed layout is a vault directory containing an encrypted
root index and a blob store:

``` text
EasyVault/
├── index.vault
└── blobs/
    ├── 8f3a91....blob
    ├── 21bc72....blob
    ├── a90213....blob
    ├── d71e45....blob
    └── ...
```

-   `index.vault` is the encrypted entry point to the vault.
-   `blobs/` contains encrypted entity data, including directory
    manifests and file content.
-   Blob filenames use random identifiers rather than original
    filenames, extensions or sequential names.
-   The original filenames, folder names and logical hierarchy are
    stored only inside encrypted data.

The operating system can still observe some information, such as the
vault's total size, file sizes, timestamps, access patterns and changes
to the storage directory. Encryption alone does not hide all metadata.

### 4.2 Role of the root index

`index.vault` should be a small authenticated root manifest, not a copy
of the entire folder hierarchy.

It should contain only the information needed to start navigating the
vault, such as:

-   Vault format/version information that can safely be public, or a
    protected equivalent.
-   A reference to the root directory entity.
-   The protected key material or key reference required to access the
    root directory.
-   Any minimal vault configuration required at unlock.

Directory contents should be stored in their own encrypted directory
entities. This avoids requiring the entire vault structure to be loaded
when the vault is opened.

The index must be authenticated and updated safely. Updates should be
designed so a crash or interrupted write does not leave the vault in an
inconsistent or unusable state.

## Hierarchical Entity Storage and Concurrent Chunk Processing

EasyVault uses a hierarchical storage model in which directories, files, and data blobs have distinct responsibilities.

### Storage hierarchy

The vault is structured around three types of components:

- **Directory entities:** Each directory contains an encrypted manifest listing its child directories and files.
- **File entities:** Each file has an encrypted manifest containing its metadata, such as its name and format, along with references to the blobs that store its contents.
- **Blobs:** Blobs contain encrypted file data. They have no knowledge of the original filename, file format, or logical directory structure. Each blob is identified by a random ID.

The logical structure looks like this:

```text
index.vault
    |
    v
Root Directory
    |
    +---- Directory A
    |         |
    |         +---- File A
    |         |       |
    |         |       +---- Blob 1
    |         |       +---- Blob 2
    |         |
    |         +---- File B
    |                 |
    |                 +---- Blob 3
    |
    +---- Directory B
              |
              +---- File C
                      |
                      +---- Blob 4
```

### Selective decryption

EasyVault should avoid decrypting the entire vault when it is unlocked.

Instead, it uses lazy decryption:

1. When the vault is unlocked, only the root index and the metadata required to access the root directory are decrypted.
2. When the user navigates into a directory, only that directory's manifest is decrypted.
3. File manifests are decrypted when their metadata is needed for display or interaction.
4. File contents remain encrypted until the user explicitly opens the file.
5. When a file is opened, only the required data blobs are read and decrypted.

This approach limits the amount of sensitive information held in memory at any given time.

### Blob-based file storage

Files are divided into encrypted blobs, particularly when they are large.

The file manifest stores the ordered references to these blobs. This allows EasyVault to reconstruct the original file without requiring each blob to understand the file it belongs to.

Blob references and their ordering must remain encrypted inside the file manifest.

The blob storage layer should remain independent of the logical directory structure. This separation allows the physical storage implementation to change without requiring changes to how users navigate their vault.

### Concurrent blob processing

When a file consists of multiple blobs, EasyVault can process them concurrently to improve performance.

For example, several encrypted blobs can be read and decrypted at the same time rather than processing every blob sequentially.

However, concurrency should be bounded. Processing every blob simultaneously could consume excessive memory and system resources, especially when opening large files.

The implementation should therefore:

- Use a limited number of concurrent workers.
- Read and decrypt blobs independently where possible.
- Preserve the original blob ordering when reconstructing the file.
- Stream decrypted data to the viewer when possible instead of loading the entire file into memory.
- Prioritize the blobs required for the current playback or reading position.
- Support seeking and partial access for large files, especially videos.

The initial number of concurrent workers should be determined through benchmarking rather than treated as a fixed design requirement.

### Integrity and security requirements

Every blob must be authenticated during decryption. The implementation must also prevent an attacker from silently reordering, duplicating, removing, or replacing blobs.

The cryptographic design should bind each blob to its corresponding file and chunk position using authenticated metadata, such as associated authenticated data (AAD).

Nonces must be unique for each encryption operation under the same key.

EasyVault should use a vetted authenticated streaming or chunked-encryption construction rather than implementing a custom cryptographic format.

### Design goals

This storage model aims to provide:

- **Separation of responsibilities:** Directories manage logical structure, files manage metadata and blob references, and blobs contain encrypted data.
- **Selective decryption:** Only the metadata needed for the current operation is decrypted.
- **Memory efficiency:** Large files can be processed in chunks without loading everything into memory.
- **Concurrent processing:** Multiple blobs can be processed in parallel with bounded resource usage.
- **Metadata protection:** Filenames, file formats, directory relationships, and blob references remain encrypted.
- **Extensibility:** The physical blob storage implementation can evolve independently of the logical vault structure.

## 5. Entity-based storage model

The proposed storage model treats every logical object as an **entity**.
The two initial entity types are:

1.  **File entity**
2.  **Directory entity**

Each entity has a random logical identifier and is represented by
encrypted data stored as a blob.

### 5.1 File entities

A file entity represents one logical file.

It contains, or references:

-   Encrypted file content or encrypted content chunks.
-   Encrypted metadata, such as the original name and file type.
-   The information needed to identify and authenticate its content.
-   A file encryption key protected by the relevant parent/key
    hierarchy.

The operating-system blob name must not reveal the original filename or
file type.

### 5.2 Directory entities

A directory entity represents a logical folder.

Its encrypted contents contain references to child entities, including
information needed to locate them and obtain their protected keys. It
does not need to contain the actual contents of child files.

A conceptual decrypted directory manifest might look like:

``` json
{
  "children": [
    {
      "entity_id": "random-entity-id-a",
      "protected_key": "wrapped-child-key-a"
    },
    {
      "entity_id": "random-entity-id-b",
      "protected_key": "wrapped-child-key-b"
    }
  ]
}
```

This is illustrative only. The actual on-disk directory blob must be
encrypted and authenticated. Raw child encryption keys should not be
stored unprotected inside a directory.

### 5.3 Browsing behavior

When the user opens a directory:

-   EasyVault decrypts that directory entity.
-   It reads the child references.
-   It loads and decrypts further directory entities as needed.
-   It can display filenames and folder names without decrypting the
    contents of files.

When the user opens a file, EasyVault obtains that file's key and
decrypts its content only then.

### 5.4 Separate logical identity and physical location

An entity's logical identity should be separate from its physical
storage location.

For example:

``` text
Entity ID:        random-entity-id
Storage location: blobs/ab/cdef....blob
Entity type:      directory
Protected key:    supplied by the parent/key hierarchy
```

This allows the physical storage layout to change later without changing
the logical folder structure.

## 6. Encryption and key management

The current cryptographic direction is:

-   **Password-based key derivation:** Argon2id, with a unique random
    salt and parameters calibrated for supported devices.
-   **Authenticated encryption:** XChaCha20-Poly1305.
-   **Large-file encryption:** An established authenticated streaming or
    chunked-encryption construction.
-   **Memory handling:** Zeroize application-controlled sensitive
    buffers and keys where practical.
-   **Randomness:** Use the operating system's cryptographically secure
    random-number generator through a suitable Rust library.

Cryptographic algorithms and formats should be selected from
established, maintained libraries. EasyVault should not invent its own
cipher, nonce construction, authentication scheme or streaming protocol.

### 6.1 Envelope encryption

The proposed key-management approach uses separate random
data-encryption keys (DEKs) for entities or files, protected by
higher-level keys.

A possible hierarchy is:

-   The master password is processed with Argon2id to derive a
    key-encryption key (KEK).
-   The KEK protects the root directory key.
-   Directory keys protect or wrap the keys of their child entities.
-   A file's random DEK encrypts that file's content.
-   A directory's random key encrypts its directory manifest.

The exact hierarchy and key-wrapping format still need a detailed
cryptographic design and review. The key principle is that directory
manifests must not expose raw child keys, and file content keys must be
independently random.

This approach can make master-password changes more efficient: re-wrap
the relevant keys instead of re-encrypting every file's content.

## 7. Chunked file storage and size privacy

Large files should be represented as multiple encrypted chunks rather
than requiring a single huge in-memory decryption operation.

### 7.1 Chunking

-   A large file is split into ordered chunks.
-   Each chunk is encrypted and authenticated.
-   A file manifest records the chunk order and information needed to
    retrieve and verify the chunks.
-   The manifest itself is encrypted as part of the file entity.
-   Chunks are internal parts of a file entity, not separate
    user-visible folder entities.

Chunking supports streaming, random access, lower memory usage and
independent authentication of portions of a file.

### 7.2 Chunk size

A fixed chunk size is being considered to reduce leakage from individual
chunk sizes. Candidate sizes discussed include 4 MiB, 8 MiB and 16 MiB.
The final size should be selected through performance testing.

A final partial chunk can be padded to the fixed size to conceal the
exact file length more effectively.

### 7.3 Metadata leakage limits

Fixed-size chunks and padding can make exact file sizes harder to infer,
but they do not hide everything.

An observer may still infer information from:

-   The total number of chunks.
-   Total vault storage consumption.
-   The number of blobs.
-   File access and modification patterns.
-   Changes in storage size over time.

Hiding access patterns and total storage usage would require more
advanced techniques and is not part of the initial implementation scope.

### 7.4 Chunk encryption requirements

Chunk encryption must use a carefully designed, authenticated format. It
must protect against tampering, reordering, duplication, truncation and
nonce reuse. Use a vetted streaming or chunked-encryption construction
rather than designing an ad hoc scheme.

## 8. Optional additional password protection

An optional second password for particularly sensitive files or
directories is being considered.

The intended behavior:

-   Normal entities are accessible after the vault is unlocked.
-   An additionally protected entity requires its own password before
    its content/key can be accessed.
-   The additional password is independent of the vault master password.
-   The extra password protects the relevant entity key through a
    password-derived key and key wrapping; it should not merely be an
    interface-level password prompt.
-   The application should not silently retain access to protected child
    keys after the protected entity is locked.

A directory-level password could protect access to the directory's child
references and keys. A file-level password could protect an individual
file's key.

The user experience still needs a decision: whether an additionally
protected entity remains unlocked until it is closed, for a short
configurable period, or until the vault itself is locked.

Loss of an additional password may mean permanent loss of access if no
recovery mechanism is deliberately designed. No hidden recovery backdoor
is intended.

## 9. Folder containers and import behavior

The primary storage model is individual file entities and directory
entities, with large files internally chunked.

An optional **encrypted directory container** or indexed container mode
has also been discussed. In that mode, a directory could be packaged as
one encrypted object with an encrypted internal manifest.

This is not required for the initial version. It adds complexity around
updating one file inside a container, random access, manifest management
and partial decryption.

The safer initial direction is to import folders into EasyVault's entity
model and store their files as individual file entities. Container
support can be evaluated later.

EasyVault should avoid encrypting arbitrary source folders in place as
its primary workflow. Importing and encrypting a copy is easier to make
reliable and reduces the risk of interrupted in-place encryption leaving
partial plaintext or inconsistent data.

## 10. Automatic locking

EasyVault should lock automatically in these situations:

-   After a configurable period without user activity.
-   When the user closes the application window.
-   When the user explicitly chooses Lock.

A default inactivity timeout of approximately 25 minutes was discussed,
with a configurable range as a possible setting.

### 10.1 Activity detection

System-wide idle detection is preferable to monitoring only activity
inside the EasyVault window. Otherwise, using another application could
cause EasyVault to incorrectly conclude that the user is idle.

System-wide idle detection requires platform-specific integration:

-   Windows: use an appropriate Windows idle-time API.
-   Linux: use suitable desktop/session facilities, accounting for
    differences between desktop environments and display/session
    systems.

The platform integration should be isolated from the shared vault and
encryption core.

### 10.2 Lock behavior

Locking should do more than hide the window. It should:

-   Stop access to decrypted vault data.
-   Clear application-controlled encryption keys and sensitive buffers
    where possible.
-   Close previews and revoke access to decrypted content.
-   Return the application to its locked state.

Closing the application should trigger the lock/cleanup routine before
normal shutdown where possible. A crash or forced termination cannot
guarantee that every memory buffer is securely erased.

## 11. Cross-platform targets

The application is intended to support:

-   Windows
-   Linux

The goal is one shared Rust core and shared user interface, with small
platform-specific modules where required.

Expected shared components include:

-   Encryption and key management.
-   Entity and vault logic.
-   Database/index handling.
-   Blob and chunk storage.
-   User interface.
-   Most file operations.

Platform-specific work is expected for:

-   System-wide idle detection.
-   Packaging and installers.
-   Native messaging host installation and registration for browser
    integration.
-   OS-specific dependencies and integration details.

Tauri uses the native webview stack for each platform. Linux builds may
require WebKitGTK and related system dependencies, while Windows uses
WebView2.

### 11.1 Packaging targets

Initial packaging targets under consideration:

-   Windows installer (`.msi` or another suitable installer format).
-   Linux `.deb` package.
-   Linux AppImage.

RPM packaging may be added for Fedora, RHEL-family distributions and
openSUSE. Packaging formats do not require separate encryption
implementations or separate vault formats.

Builds should be tested on both Windows and Linux from early in
development rather than treating Linux as a late port.

## 12. Browser autofill

Firefox password autofill is planned through a browser extension and a
local native messaging host.

The extension will need to communicate with the EasyVault application
through a controlled local interface. Credentials should only be
supplied to matching sites and should not be exposed to unrelated
origins or frames.

The precise approval flow, trusted-site behavior, native messaging
protocol and cross-platform installation process remain to be designed.

The browser integration is a separate security boundary: credentials
necessarily become available to the browser briefly when autofill
occurs.

## 13. Current technology choices

The current project direction includes:

-   **Rust:** Main programming language.
-   **SQLite:** Local database technology for index/metadata and related
    bookkeeping, with sensitive values encrypted before storage.
-   **XChaCha20-Poly1305:** Authenticated encryption choice under
    consideration for entity data and/or chunks.
-   **Argon2id:** Password-based key derivation.
-   **Tauri:** Desktop application framework.
-   **Firefox extension:** Planned browser autofill integration.

SQLite itself does not encrypt data automatically. Sensitive metadata
and file contents must be encrypted by EasyVault before being written to
the database. The implementation must also consider SQLite journals, WAL
files, temporary files, backups and crash recovery so that plaintext is
never accidentally written to them.

The current project layout is:

``` text
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

This layout may evolve as the Tauri application, browser extension and
platform-specific code are added.

## 14. Decisions still to be finalized

The following items have been discussed but are not yet final
implementation decisions:

-   Exact entity and directory-manifest binary format.
-   Exact key hierarchy and key-wrapping construction.
-   Exact chunk format, chunk size and streaming-encryption library.
-   Whether chunks are individual blob files or packed into larger
    storage files.
-   Whether the root index and directory manifests use the same
    serialization format.
-   Crash-safe and atomic update/recovery strategy for index and
    directory changes.
-   Backup, restore and vault migration format.
-   Whether additional-password protection applies to files,
    directories, or both in the first release.
-   Whether protected entities remain unlocked until closed or use a
    timeout.
-   Whether a recovery key is offered. The current privacy goal is no
    developer-controlled recovery backdoor.
-   Exact inactivity timeout settings and behavior when the machine
    sleeps, locks or changes user session.
-   Linux desktop environments and packaging targets to support
    initially.
-   Browser extension/native messaging security design.
-   Whether indexed directory containers are worth supporting after the
    initial entity-based implementation.

## 15. Suggested implementation order

A sensible development sequence is:

1.  Define and document the on-disk format and threat model.
2.  Implement password-based key derivation and authenticated key
    verification.
3.  Implement root index creation, authentication and safe updates.
4.  Implement encrypted directory entities and entity references.
5.  Implement encrypted file entities.
6.  Add chunked authenticated encryption and streaming reads.
7.  Add locking, memory handling and inactivity detection.
8.  Add the desktop UI and cross-platform packaging.
9.  Add optional per-file/per-directory password protection.
10. Add Firefox autofill.
11. Evaluate advanced size and access-pattern privacy features.

Security-sensitive formats should be reviewed and tested before being
treated as stable. In particular, test wrong passwords,
modified/truncated/reordered chunks, interrupted writes, corrupted
indexes, missing blobs, disk-full conditions, backup/restore and
cross-platform vault compatibility.

------------------------------------------------------------------------

**Summary:** EasyVault is planned as a local-first encrypted vault built
around encrypted entities. The root index leads to encrypted directory
entities; directories reference child entities; file entities contain
encrypted file data, potentially split into authenticated fixed-size
chunks. Unlocking reveals the logical structure, while individual file
contents remain encrypted until opened. The design targets Windows and
Linux and prioritizes minimizing plaintext exposure and metadata leakage
without claiming absolute protection against a compromised device.
