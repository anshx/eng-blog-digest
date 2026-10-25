---
title: "Reflections on Trusting Trust"
source: https://dl.acm.org/doi/10.1145/358198.358210
author: Ken Thompson
company: Bell Labs
date_posted: 1984-08-01
date_digested: 2026-10-25
---

# Reflections on Trusting Trust

## What's new to learn

1. **Quines** — A program that outputs its own complete source code verbatim; the computational primitive that makes self-replication possible in any Turing-complete language.
2. **The compiler bootstrap attack** — A Trojan embedded only in a compiler *binary* perpetuates through every recompile of its own *source*, because the trojaned binary is what reads and processes the source — so cleaning the source does nothing.
3. **Transitivity of trust** — Source-code review cannot establish the safety of a binary; trust is a chain, and its strength equals its weakest link, which must eventually be grounded in something physically trusted (hardware, or a compiler you built by hand from silicon up).

## Prerequisites

- How a self-hosted ("bootstrapped") compiler works — a compiler written in the language it compiles; the Unix C compiler compiles itself
- The basic Unix login authentication path (password hashing, PAM)
- The difference between source code and the resulting compiled binary / executable on disk

## The core idea

Thompson presents his argument in three stages, each building on the last.

**Stage 1 — Quines.** Any program can be made to print its own source code. The construction: store the source of the "interesting" part of the program as a character array, then print that array twice — once as a C initializer literal (to reconstruct the array declaration), and once raw (to reconstruct the code itself). This is a self-referential fixed point, not a hack; it is a straightforward consequence of a Turing-complete language having both data and code representations.

**Stage 2 — A Trojan in the compiler.** The C compiler is just another program. Thompson modifies it to pattern-match on the source it is currently compiling: if it recognises the `login` program, it silently emits extra object code that accepts a secret backdoor password, bypassing all normal authentication. Clean source in; trojaned binary out. A reader auditing the `login` source sees nothing wrong.

**Stage 3 — The self-perpetuating Trojan.** Here is the turn. Thompson also modifies the compiler to recognise when it is compiling *the compiler itself*, and at that point silently re-inserts both modifications (the login backdoor and the self-replication rule). The modification is, again, a quine: the text of the Trojan is embedded in the binary so the compiler can reproduce it verbatim.

Bootstrap sequence:
1. Compile the doubly-modified C source with the old clean binary → obtain trojaned binary **B₁**.
2. Restore the C compiler source to its original clean state — no trace of any Trojan remains in any source file.
3. Distribute **B₁** as the "official" compiler. Anyone compiling the clean source with **B₁** gets a new binary **B₂** that contains both Trojans, because **B₁** pattern-matches "this looks like `login`" and "this looks like the compiler" and re-inserts them.
4. **B₃**, **B₄**, … all carry the Trojans. The chain is self-sustaining forever.

No source-code audit, no matter how thorough, will ever find anything: the source is genuinely clean.

Thompson's closing line: *"You can't trust code that you did not totally create yourself."*

## Mechanics

### The quine construction (Stage 1)

```c
/* Pseudocode faithful to Thompson's description */
char s[] = {
    '\t', '0', '\n', '}', ';', '\n',
    /* ... the ASCII values of main() below ... */
    '\0'
};

main() {
    int i;
    printf("char\ts[] = {\n");
    for (i = 0; s[i]; i++)
        printf("\t%d,\n", s[i]);          /* print the array initializer */
    printf(s);                             /* print the program body */
}
```

The character array `s[]` holds the text of the `main()` function.  When run, it first prints the array initializer (reconstructing `char s[] = {…};`) and then prints the contents of `s[]` (reconstructing `main()`).  The output is byte-for-byte the original source.

### The login Trojan (Stage 2, pseudocode)

```c
void compile(char *source) {
    if (pattern_match(source, login_source_pattern)) {
        emit_object_code(trojan_login_binary_template);
        return;
    }
    normal_compile(source);
}
```

`login_source_pattern` is a recogniser for the part of the login source that handles password checking.  `trojan_login_binary_template` emits the normal logic *plus* a branch that accepts the backdoor password.

### The self-replicating Trojan (Stage 3, pseudocode)

```c
void compile(char *source) {
    if (pattern_match(source, login_source_pattern)) {
        emit_object_code(trojan_login_binary_template);
        return;
    }
    if (pattern_match(source, compiler_source_pattern)) {
        emit_object_code(trojan_compiler_binary_template); /* re-inserts both trojans */
        return;
    }
    normal_compile(source);
}
```

`trojan_compiler_binary_template` is a quine of the entire modified `compile()` function — the binary carries its own blueprint, so every binary the trojaned compiler produces knows how to reproduce itself.

### Why cleaning the source does not help

The compiler's source describes what code *should* be emitted.  But the *binary* is what actually does the emitting.  When you feed clean source to the trojaned binary:

```
clean source  →  [trojaned binary reads it]  →  trojaned output binary
```

The trojaned binary's pattern-match fires on the clean source — it doesn't care that the source is now clean; it fires on *structure* (this looks like login, this looks like the compiler).  The clean source triggers the Trojan exactly as the modified source did.

## Where it breaks

**Diverse Double-Compilation (Wheeler, 2009).** Compile the suspect compiler C_s using a trusted compiler T_trusted to get binary B₁.  Also compile T_trusted from its own clean source using T_trusted to get T_trusted_clean.  Then compile C_s using T_trusted_clean and compare the result to B₁.  If C_s has a self-replicating Trojan that recognises T_trusted, the two compilation paths diverge.  The defence rests on the assumption that two independent, separately audited compilers are vanishingly unlikely to carry the *same* Trojan.

**Reproducible Builds.** Make compilation deterministic and hermetic (same source + same toolchain → identical binary, bit for bit, across independent machines).  Publish build manifests.  Once a binary is reproducible, auditors can verify that the released binary matches the published source on multiple independent machines.  The Reproducible Builds project has achieved this for Debian, NixOS, and many other distributions.

**Stage-0 bootstrap from scratch.** Tools like `bootstrappable-builds.org` stage-0 start from a 357-byte seed binary and bootstrap a full C compiler without ever trusting a pre-existing binary.  This breaks the trust chain entirely at the cost of enormous up-front effort.

**What no defence fully eliminates.** All defences push trust to a lower layer: to the hardware, to the physical supply chain, to the people who manufactured the chip.  Thompson's deeper point is that there is no algorithm that terminates this regression; at some point you are trusting atoms, and that trust is social, not computational.

## Why it works

The self-perpetuating Trojan works because of a **fixed-point property of the compilation function**.

Let `compile(B, S)` mean "use binary B to compile source S".  Thompson constructs a binary B_trojan such that:

```
compile(B_trojan, C_source) ≅ B_trojan    (on the compiler-source input)
compile(B_trojan, login_source) ≅ login_with_backdoor
```

The compiler binary is a fixed point of itself under recompilation.  This is a concrete instance of **Kleene's Second Recursion Theorem** (1938): for any total computable function f, there exists a program e such that e and f(e) are extensionally equal.  The quine construction *builds* that fixed point explicitly.

A corollary is that **source-code review is not a complete verification method for binary safety**.  This is exactly what **Rice's Theorem** (1953) says in the abstract: any non-trivial semantic property of a program is undecidable.  You cannot write a program that correctly answers "does this binary contain a Trojan?" for all possible inputs.

The practical modern manifestation is **PKI trust chains**.  A TLS certificate chain: leaf cert ← intermediate CA ← root CA.  The root CA is trusted not because you can prove it is trustworthy, but because it was installed in your OS/browser by the vendor at purchase time — a physical-world trust event.  Thompson's argument is that executable trust works the same way: the trust chain must bottom out in something you accepted via a physical, out-of-band channel.

**Hardware security modules** (TPM, AMD SEV, Apple Secure Enclave, AWS Nitro) are exactly this answer at scale: a chain of measurements (hash of firmware → hash of bootloader → hash of kernel) anchored to a key burned into silicon at the factory, trusted by physical procurement.  Secure Boot is Thompson's trust chain made operational in every PC shipped after 2012.

**The 2024 XZ Utils backdoor** (CVE-2024-3094) is Thompson's scenario in the wild, with a social-engineering twist: instead of modifying the compiler, the attacker injected malicious code into the *release tarball* (the binary distribution artefact) but not into the *git repository* (the source).  The gap between "what you can audit in git" and "what you actually downloaded and compiled" is precisely the source/binary gap Thompson described.

## Going deeper

1. **"Countering Trusting Trust through Diverse Double-Compilation" — David Wheeler (2009)** — The formal proof that DDC detects any Trojan that would need to recognise the trusted compiler, with a rigorous adversary model. Available free at https://dwheeler.com/trusting-trust/
2. **"Wolves in the Repository: A Software Engineering Analysis of the XZ Utils Supply Chain Attack" (arXiv 2504.17473, 2025)** — A 2025 post-mortem of the only known real deployment of a Thompson-style attack, showing exactly where the tarball/source gap was exploited and which SDLC controls would and would not have caught it.
3. **Bootstrappable Builds (bootstrappable-builds.org)** — The technical effort to build an entire software stack from a minimal, hand-auditable seed binary, directly operationalising Thompson's prescription of "start from something you built yourself."
