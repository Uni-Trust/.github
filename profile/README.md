# UniTrust - Decentralized Document Approval and Verification System

## Overview

UniTrust is a **permissioned consortium blockchain** system that provides a secure, transparent, and coercion-resistant solution for digital document (diploma) approval and verification. Instead of trusting a single centralized database, trust is distributed across multiple institutions (YOK—Higher Education Council, universities) using cryptographic proofs, multi-signature approvals, and a Panic Key anti-coercion mechanism.

**Course**: Hacettepe University Computer Engineering, BBM479 <br>
**Supervisor**: Doc. Dr. Adnan Özsoy <br>
**Members**: Ümit Sevinçler, Yusuf İpek, Veli Toprak Güngör <br>

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Traditional System vs. UniTrust](#traditional-system-vs-unitrust)
3. [System Architecture](#system-architecture)
4. [Key Design Decisions](#key-design-decisions)
5. [Features & Focus](#features--focus)
6. [Tech Stack](#tech-stack)

## Problem Statement

Current digital document management systems (university student info systems, e-Devlet) rely on a centralized trust model with critical vulnerabilities:

| Vulnerability | Consequence |
| --- | --- |
| Single Point of Attack | One compromised database server invalidates all trust |
| Insider Threats | Authorized personnel can alter records without detection |
| Mutable Audit Trails | Logs can be deleted or modified after the fact |
| No Error Grace Period | Wrong file uploaded? It becomes permanent immediately |

Recent fake diploma scandals in Turkey show these are not theoretical risks.

## Traditional System vs. UniTrust

### Traditional System
```
    ┌──────────────┐
    │  University  │
    │   Database   │
    │  (mutable)   │
    └──────────────┘
         │
         │ one click = save
         ▼
    ┌──────────────┐
    │ Single Admin │
    │ Control      │
    └──────────────┘
         │
         ▼
    ┌──────────────┐
    │ Logs (can be │
    │  deleted)    │
    └──────────────┘
```

**Problems:**
- Single point of failure
- Insider threats (admin can alter records)
- No permanent audit trail
- No error recovery
- No protection against coercion

### UniTrust System
```
           REST API (JWT)
  ┌──────────────┐ ◄──────────────► ┌──────────────────────────┐
  │   Frontend    │                  │   Backend (Go/Gin)        │
  │ (React + Web3)│                  │                           │
  │               │                  │  ┌─────────────────────┐  │
  │ ethers.js +   │                  │  │ IPFS Node Container │  │
  │ MetaMask ───┬─┼─ on-chain txs ──┼──► (localhost:5001)    │  │
  │ (sign txs)  │ │                  │  │                     │  │
  │             │ │                  │  │ Stores: PDFs,       │  │
  └─────────────┼─┘                  │  │ encrypted files     │  │
                │                    │  └─────────────────────┘  │
                │ read-only calls    │                           │
                ▼                    │  ┌─────────────────────┐  │
          ┌──────────────┐           │  │ Event Listener      │  │
          │ Smart        │           │  │ (updates drafts)    │  │
          │ Contract     │ ◄─────────┼─ │                     │  │
          │ (Multi-Sig)  │           │  └─────────────────────┘  │
          │              │           │                           │
          │ Stores:      │           │  ┌─────────────────────┐  │
          │ • IPFS CID   │           │  │ PostgreSQL          │  │
          │ • Approvals  │           │  │ (draft documents,   │  │
          │ • Signers    │           │  │  user roles)        │  │
          │ • Timestamps │           │  └─────────────────────┘  │
          └──────────────┘           └──────────────────────────┘
                │
                │ immutable on-chain
                │ (multi-validator consensus)
                ▼
          ┌──────────────┐
          │ Blockchain   │
          │ Network      │
          │ (PoA, Geth)  │
          └──────────────┘
```

**Solutions:**
- Distributed trust across multiple validators
- IPFS for decentralized document storage (content-addressed)
- Multi-signature approvals (K-of-N validators must sign)
- Immutable on-chain audit trail
- 48-hour grace period for document correction
- Panic Key mechanism for coercion resistance

### Side-by-side Comparison

| Aspect | Traditional System | UniTrust |
| --- | --- | --- |
| **Trust model** | Central server (single admin) | Distributed blockchain (multi-validator consensus) |
| **Document storage** | Single database (mutable) | IPFS + Blockchain (immutable hashes) |
| **Approval process** | One person clicks "save" | Multi-signature (minimum N validators must approve) |
| **Audit trail** | Mutable database logs (erasable) | Immutable on-chain records (verifiable) |
| **Error correction** | None (wrong file = permanent) | 48-hour timelock grace period |
| **Coercion resistance** | None | Panic Key mechanism (silent blocklist) |

## System Architecture

**Component Roles:**

| Component | Responsibility |
| --- | --- |
| **Frontend** | Web interface; users sign transactions with MetaMask; browser-based signature (keys never leave user's device) |
| **Backend** | Stateless service; exposes REST API; listens to blockchain events; manages draft documents in PostgreSQL; reads from IPFS; validates signatures |
| **Smart Contract** | Enforces multi-signature approvals; stores IPFS content hashes; records immutable validation history; maintains signer registry |
| **IPFS** | Decentralized document storage; returns content-addressed CIDs; documents pinned across nodes |
| **PostgreSQL** | Temporary storage for drafts; user/role management; not the source of truth (blockchain is) |
| **Blockchain (Geth)** | Permissioned PoA network; consensus among known validators; permanent record |

## Key Design Decisions

| Decision | Why it matters |
| --- | --- |
| **Permissioned PoA blockchain** (not public) | Known participants (YOK, universities); zero gas costs; energy-efficient; identity-controlled network; no anonymous participation. |
| **IPFS for documents, blockchain for proofs** | Off-chain storage keeps costs low; on-chain hashes ensure tamper-proof records; separates data from validation. |
| **MetaMask for browser-based signing** | Private keys remain in user's browser; never sent to backend; stronger security model; users retain control. |
| **Backend cannot write to contracts** | All state changes go through MetaMask; backend is read-only listener; forces transparency (all actions traceable on-chain). |
| **Multi-signature approval** | Minimum N validators must approve; no single authority can force approval; consensus model. |
| **48-hour grace period** | Documents can be corrected if issued by mistake; grace period is immutable (on-chain); after timeout, no changes allowed. |

## Features & Focus

| Feature | Description |
| --- | --- |
| **Multi-signature approvals** | Issuer (e.g., university registrar) creates; validator (e.g., YOK representative) confirms; both signatures required on-chain. |
| **Panic Key mechanism** | Coercion-resistant blocklist; issuer can silently mark a document as blocked; prevents forced approval; alerts authorities. |
| **Cryptographic verification** | Documents verified via IPFS hash + blockchain signatures; recipients can independently verify without contacting any server. |
| **Permanent audit trail** | Every approval, rejection, or correction is recorded immutably on-chain with timestamps and signer identities. |
| **Error correction (48-hour window)** | Document issued by mistake? Issuer can request retraction within 48 hours; after timeout, correction requires validator approval. |
| **Decentralized document storage** | IPFS ensures documents survive server failures; content-addressed (hash = file integrity). |
| **Role-based access control** | Issuer, validator, viewer roles; different permissions on-chain and in frontend; enforced by smart contract. |

## Quick Workflow

**How a document flows through UniTrust:**

```
Step 1: Issuer Creates              Step 2: Validator Approves        Step 3: Document on-chain
┌─────────────────────┐            ┌──────────────────────┐         ┌──────────────────────┐
│ University Registrar│            │ YOK Representative   │         │ Blockchain Network   │
│                     │            │                      │         │                      │
│ 1. Upload PDF       │ ─REST API──▶│ 2. Reads document    │         │ 3. Multi-sig approval│
│ 2. Select validators│ (draft)    │    from backend/IPFS │ ─web3──▶│ 4. IPFS CID stored   │
│ 3. Sign with        │            │ 3. Reviews content   │ (direct)│ 5. Immutable record  │
│    MetaMask         │ ─web3──────┼───────────────────────▶ Sign with MetaMask        │
│    (broadcasts tx)  │ (direct)   │ 4. Approves & signs  │ (direct)│ 6. Emit event        │
│                     │            │    with MetaMask     │         │                      │
│ Draft in PostgreSQL │            │                      │         │ Permanent & auditable
└─────────────────────┘            └──────────────────────┘         └──────────────────────┘
```

**Key Point:** All signing transactions (web3) go **directly to blockchain**, bypassing the backend. Backend only:
- Stores drafts (before signing)
- Listens to blockchain events
- Provides read-only API for document retrieval

**Verification (Anytime, Anywhere):**
- Recipient receives document + blockchain proof
- Can verify IPFS hash against smart contract on-chain
- No need to contact central authority
- Cryptographic verification ensures authenticity

**Coercion Protection (Panic Key):**
- Issuer senses coercion or pressure
- Silently activates Panic Key via MetaMask signature
- Document marked as blocked on-chain
- System alerts authorities
- No external sign of intervention (silent, hence "panic key")

## Tech Stack

| Layer | Technology | Notes |
| --- | --- | --- |
| Orchestration | Docker Compose | Multi-service local environment. |
| Database | PostgreSQL | Primary relational store. |
| Distributed storage | IPFS Kubo | Content-addressed document storage. |
| Blockchain node | Geth (go-ethereum) | Private permissioned PoA chain. |
| Contracts tooling | Hardhat + Node.js 20 | Solidity build/test/deploy tooling. |
| Backend service | Go (Gin, Gorm) | API and core services. |
| Frontend service | React + Vite (Node.js 20) | Web application. |
