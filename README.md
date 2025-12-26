# SCL:V1 Specification — Official Protocol Repository
Status: Informational (Non-Normative)
This repository is the **single authoritative source** for the SCL:V1 protocol.

Everything here defines:
- The language
- The grammar
- Canonical JSON rules
- Engine behavior
- Validation rules
- Runtime semantics
- Version and freeze policy
- Determinism guarantees

All SDKs, engines, tools, CLIs, servers, and third-party integrations **must comply with this repository exactly**.

---

## 1. Purpose

SCL:V1 provides a **deterministic**, **portable**, and **implementation-independent** instruction format for AI workflows.

This repository ensures:
- No ambiguity  
- No drift  
- No implementation-defined behavior  
- Complete reproducibility  

---

## 2. Repository Structure

### **/spec/**
Core specification documents:

- `SPEC_V1.md` — Full language specification  
- `GRAMMAR_FROZEN_V1.md` — Frozen grammar definition  
- `CANONICAL_JSON_V1.md` — Canonical JSON rules  
- `SPEC_V1_FREEZE_COMPLETE.md` — Official V1 freeze declaration  
- `SPEC_V1_FREEZE_WITH_C_CHECKLIST.md` — Engine parity confirmation  

---

### **/contracts/**
Authoritative protocol interface definitions:

- `ENGINE_V1_CONTRACT.md`  
- `PARSER_CONTRACT.md`  
- `VALIDATOR_CONTRACT_V1.md`  
- `RUNTIME_CONTRACT_V1.md`  

These define behavioral guarantees for engine builders and SDK authors.

---

### **/cli/**
Official CLI behavior and rules:

- `SCL_CLI_SPEC_V1.md`  
- `SCL_CLI_REFERENCE_V1.md`  

The CLI must never alter engine output.

---

### **/reports/**
Formal verifications and yield-stable documentation:

- Canonical JSON fix reports  
- Formatter/linter determinism reports  
- Engine normalization reports  

---

### **/matrix/**
Cross-implementation verification:

- Integration matrix  
- Global parity matrix  

These ensure every SDK and tool matches the core engine exactly.

---

## 3. Determinism Guarantees

SCL:V1 requires:

1. **Canonical JSON must be identical across all platforms.**  
2. **Errors must match byte-for-byte.**  
3. **The runtime must not rely on any external nondeterministic factors.**  
4. **All implementations must produce identical SHA-256 hashes for the same input.**

No exceptions.

---

## 4. Versioning & Freeze Policy

SCL:V1 is **frozen**.

Post-freeze changes allowed:
- Clarifications  
- Typo fixes  
- Non-semantic improvements  

NOT allowed:
- Grammar changes  
- Behavior changes  
- Canonical JSON changes  
- Output formatting changes  
- New semantics without V2  

Future versions (V2+) must be introduced separately with explicit version metadata.

---

## 5. Contribution Rules

Contributors may submit:
- Documentation clarifications  
- Improved examples  
- Error correction in text  
- Test alignment reports  

Contributors may NOT submit:
- Changes to V1 semantics  
- Behavioral changes  
- Anything that breaks determinism  

All semantic changes must target V2 or higher.

---

## 6. Who Should Use This Repository?

- Engine implementers  
- SDK maintainers  
- External vendors integrating SCL  
- Researchers validating determinism  
- Contributors building tooling  

If you build anything that *reads, writes, executes, formats, validates,* or *extends* SCL, this is your source of truth.

---

## 7. Status

SCL:V1 is complete and frozen.  
The C engine is fully aligned.  
All SDKs are being aligned using this repo.

This repository forms the **bedrock of the SCL ecosystem**.
Normative behavior is defined exclusively by the versioned specification files under /spec/. This README is informational.
