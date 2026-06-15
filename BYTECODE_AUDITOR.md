---
name: bytecode-auditor
description: "Advanced EVM bytecode auditor for smart contracts on Base. Uses the A.P.O.P. framework to identify security vulnerabilities in unverified or closed-source contracts by analyzing raw bytecode."
track: "The Bytecode Whisperer"
---

# Bytecode Auditor Skill (A.P.O.P. Framework)

This skill enables AI agents to act as specialized Security Auditors for the Base L2 network, focusing on raw EVM bytecode analysis.

## Assets (Inputs)
- **Primary:** Raw EVM Bytecode (hex string).
- **Secondary:** Basescan API integration for fetching deployed bytecode.
- **Context:** ChainID 8453 (Base Mainnet).

## Process (Execution Logic)
1. **Decompilation Simulation:** Map raw bytecode to common Opcode patterns (PUSH, PUSH, MSTORE, etc.).
2. **Vulnerability Scan:**
   - **Reentrancy:** Look for `CALL` opcodes following state-changing operations without adequate gas limits or checks.
   - **Integer Overflow:** Analyze arithmetic operations (`ADD`, `MUL`, `SUB`) lacking safe-math patterns.
   - **Access Control:** Identify `CALLER` checks and verify they restrict sensitive functions.
   - **Self-Destruct:** Scan for `SELFDESTRUCT` opcodes and identify who can trigger them.
3. **Risk Scoring:** Assign a risk level (Low, Medium, High, Critical) based on the exploitability of found patterns.

## Output (Deliverables)
- **Security Report:** A detailed Markdown report listing all identified patterns, their associated risks, and potential mitigations.
- **JSON Metadata:** An agent-readable summary of the audit findings.

## Protocol (Interaction)
- **Role:** Senior Security Auditor at Gitbank.
- **Reasoning:** Use Chain-of-Thought (CoT) to explain *why* a specific opcode sequence is dangerous.
- **Verification:** cross-reference found patterns with known 2026 DeFi exploit signatures.

---

## Example Usage

**User:** "Audit this bytecode on Base: 0x60806040..."
**AI Agent:** 
1. Calls Basescan to verify if the bytecode matches a known contract.
2. Identifies a potential `delegatecall` to an untrusted address.
3. Generates a report: "CRITICAL: Potential proxy vulnerability detected. The contract allows arbitrary delegatecalls..."
