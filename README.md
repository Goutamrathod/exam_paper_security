# Secure Exam Exchange System (SEES)

A cryptographic file-transfer suite written in Python to secure confidential examination papers transmitted between a Faculty Member (Sender) and an Examination Coordinator (Recipient) over untrusted networks.

## Table of Contents

- [Overview](#overview)
- [Cryptographic Specifications](#cryptographic-specifications)
- [Project Structure](#project-structure)
- [System Requirements & Dependencies](#system-requirements--dependencies)
- [Setup & Environment Configuration](#setup--environment-configuration)
- [Execution & Operational Workflow](#execution--operational-workflow)
  - [1. Key Generation](#1-key-generation)
  - [2. Public Key Exchange](#2-public-key-exchange)
  - [3. Starting the Recipient Listener](#3-starting-the-recipient-listener)
  - [4. Encrypting & Transmitting Examination Papers](#4-encrypting--transmitting-examination-papers)
- [Security & Tamper Verification Testing](#security--tamper-verification-testing)
- [Troubleshooting](#troubleshooting)
- [Security Guidelines](#security-guidelines)

## Overview

The **Secure Exam Exchange System (SEES)** provides end-to-end security guarantees for transferring sensitive academic documents. It guarantees:

1. **Confidentiality:** Uses **AES-256-GCM** to encrypt examination contents, with the session key encapsulated using **RSA-OAEP** under the recipient's public key.
2. **Authenticity & Non-Repudiation:** Uses **RSA-PSS** digital signatures over the payload using the sender's private key.
3. **Integrity:** Authenticates both ciphertext and associated metadata using GCM authentication tags and RSA signatures.
4. **Freshness & Replay Prevention:** Employs unique 96-bit random nonces (`os.urandom(12)`) and fresh symmetric keys for every execution run.

## Cryptographic Specifications

| Function | Algorithm / Primitive | Key Length / Parameters |
| :--- | :--- | :--- |
| **Symmetric Encryption** | AES-GCM | 256-bit Key, 96-bit Nonce |
| **Key Encapsulation** | RSA-OAEP | 2048-bit / 4096-bit RSA, MGF1 (SHA-256) |
| **Digital Signature** | RSA-PSS | Salt Length = MAX, Hash = SHA-256 |
| **Integrity Check** | AES-GCM Auth Tag | 128-bit Authentication Tag |

## Project Structure

```text
exam_security_project/
├── secure_exam_exchange.py   # Core Cryptographic Engine & CLI tool
├── sender_client.py          # Faculty Client module for network transmission
├── receiver_server.py        # Coordinator Server daemon module
├── final_exam.txt            # Sample input examination paper
├── keys/                     # Local PEM key store
│   ├── sender_private.pem
│   ├── sender_public.pem
│   ├── recipient_private.pem
│   └── recipient_public.pem
├── received_files/           # Verified and decrypted output directory
├── README.md                 # Project documentation
└── .gitignore                # Git exclusions file
```

## System Requirements & Dependencies

- **Python Version:** Python 3.10 or higher (Tested on macOS / Linux / Windows)
- **Library Dependency:** Standard Python `cryptography` package

## Setup & Environment Configuration

### 1. Project Directory & Environment Setup

Clone or create your project directory and enter it:

```bash
mkdir -p exam_security_project
cd exam_security_project
```

### 2. Dependency Installation

Install the cryptographic library using `pip`:

```bash
python3 -m pip install cryptography
```

## Execution & Operational Workflow

### 1. Key Generation

Generate separate RSA key pairs for both the Sender (Faculty) and Recipient (Coordinator):

**On Laptop A (Sender / Faculty):**

```bash
python3 secure_exam_exchange.py genkeys keys/sender_private.pem keys/sender_public.pem
```

**On Laptop B (Recipient / Coordinator):**

```bash
python3 secure_exam_exchange.py genkeys keys/recipient_private.pem keys/recipient_public.pem
```

### 2. Public Key Exchange

Exchange public keys over an out-of-band trusted channel (e.g., in-person USB transfer or verified chat):

1. Copy `sender_public.pem` from **Laptop A** to **Laptop B** into its `keys/` directory as `sender_public_trusted.pem`.
2. Copy `recipient_public.pem` from **Laptop B** to **Laptop A** into its `keys/` directory as `recipient_public_trusted.pem`.

> **CRITICAL:** Private keys (`sender_private.pem` and `recipient_private.pem`) must **NEVER** leave their local machines.

### 3. Starting the Recipient Listener

On **Laptop B (Coordinator)**, run the receiver daemon:

```bash
python3 receiver_server.py --port 65432 --outdir ./received_files
```

### 4. Encrypting & Transmitting Examination Papers

On **Laptop A (Faculty)**, send the confidential file to the Coordinator's IP address:

```bash
python3 sender_client.py send <COORDINATOR_IP> final_exam.txt --port 65432
```

*Example:*

```bash
python3 sender_client.py send 192.168.1.35 final_exam.txt --port 65432
```

## Security & Tamper Verification Testing

To test the system's resilience against tampered inputs or active adversary attacks, run the client with the `--tamper` flag:

```bash
python3 sender_client.py send 192.168.1.35 final_exam.txt --tamper
```

### Expected Behavior

The receiver program detects bit-flips or signature mismatches prior to decryption, aborts file release, and displays a security error:

```text
[SECURITY ERROR] AUTHENTICATION FAILURE: Digital signature verification failed! Package tampered or untrusted sender.
```

## Troubleshooting

### Python Command Not Found (`pyenv` issue on macOS)

If `pyenv` fails to detect your environment directory:

```bash
cd ~
cd path/to/exam_security_project
pyenv global 3.10.13
```

Ensure **Terminal** has been granted **Full Disk Access** under `System Settings > Privacy & Security`.

### Connection Refused / Connection Timeout

- Confirm the receiver is running and listening on port `65432`.
- Verify network reachability (`ping <COORDINATOR_IP>`).
- Ensure local firewalls permit traffic over TCP port `65432`.

## Security Guidelines

- **In-Memory Decryption:** Plaintext examination papers are written to disk **only** after both signature verification and AES-GCM tag verification pass successfully.
- **Git Hygiene:** Add `keys/*.pem` and output `package.bin` files to `.gitignore` to prevent sensitive credentials from entering version control.