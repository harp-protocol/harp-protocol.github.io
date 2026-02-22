# HARP — Human Authorization & Review Protocol

**HARP** is an open, cryptographically verifiable protocol that ensures every AI agent action receives explicit human approval before execution.

> ⚠️ **Draft Specification** — HARP is under active development. Do not treat as a finalized standard.

## Overview

HARP introduces a secure, out-of-band authorization layer between AI coding agents and the tools they invoke. It ensures that no code is committed, no command is executed, and no file is modified without a cryptographically signed human decision.

### Key Principles

- **Zero-Knowledge Gateway** — The relay routes only encrypted ciphertext and metadata. It cannot read prompts, inspect code, or decrypt artifacts.
- **End-to-End Encryption** — All artifacts are encrypted from the desktop agent to the mobile approver. No intermediary sees plaintext.
- **Fail-Closed Enforcement** — If the gateway is unreachable or the signature is invalid, the action is denied by default.
- **Cryptographic Binding** — Every approval is tied to its specific artifact via SHA-256 hashing and Ed25519 signatures.

### Protocol Layers

| Layer | Purpose |
|-------|---------|
| **CORE** | Artifact hashing, decision signing, signature verification |
| **PROMPT** | Prompt classification, hash computation, metadata tagging |
| **SESSION** | Session lifecycle, snapshot hashing, state management |

## Website

The specification website is built with [Astro](https://astro.build) + [Starlight](https://starlight.astro.build) and deployed via GitHub Pages.

**Live site:** [harp-protocol.github.io](https://harp-protocol.github.io)

### Local Development

```bash
# Install dependencies
yarn install

# Start dev server
yarn dev

# Build for production
yarn build
```

### Project Structure

```
src/
├── config/          # Site configuration (theme, menu, sidebar)
├── content/docs/    # Documentation pages (MDX)
├── components/      # Astro components
├── styles/          # CSS (Tailwind v4)
└── assets/          # Images, logos, icons
```

## Deployment

The site deploys automatically to GitHub Pages on push to `main` via the workflow in `.github/workflows/deploy.yml`.

## License

This specification is open source. See [LICENSE](LICENSE) for details.
