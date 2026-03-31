# Tokenizer Backdoors: The Silent Supply Chain Attack Vector That Existing Tools Don't Catch

> **Author:** Shoumik Chandra | AI Security Researcher  
> **Disclosure Status:** Reported to Hugging Face Security Team ✅ | PoC Repositories Made Private ✅  
> **Research Phase:** Proof of Concept. Full technical details reserved for upcoming research paper

---

## TL;DR

A class of supply chain attack exists that allows a malicious actor to **silently inject system prompts, bypass model safety guardrails, and propagate the attack through the Hugging Face model ecosystem** entirely through the tokenizer, without touching model weights. Existing security tools like `picklescan` and `SafeTensors` provide **zero coverage** for this attack surface. This writeup documents the methodology, the detection gap, and proof-of-concept findings demonstrating real-world impact.

---

## Background

This research began with a simple question:

> *"If everyone is focused on model weights, what else in the model artifact is being ignored?"*

A modern model on Hugging Face is not just `model.safetensors`. It's a collection of files: config JSONs, special token maps, and critically a `tokenizer.json` and an optional `chat_template.jinja`. These files govern how user input is formatted and passed to the model. They are loaded **silently, automatically, and unconditionally** with a single `AutoTokenizer.from_pretrained()` call.

That's a large, completely unguarded attack surface.

---

## Research Phases

### Phase 1 — Mapping the Detection Gap

The first step was auditing what existing tools actually scan.

**`picklescan` behavior:**


<img width="917" height="503" alt="Screenshot (1783)" src="https://github.com/user-attachments/assets/d59afd8d-d6bd-49f2-a457-16b736e1bb60" />




The behavior is subtle and important: `picklescan` attempts to parse `tokenizer.json` as a pickle file, fails with a warning, and reports no threat. It was never designed to understand tokenizer semantics. It simply cannot detect a maliciously crafted `chat_template`.

**`SafeTensors` behavior:**


<img width="690" height="251" alt="Screenshot (1784)" src="https://github.com/user-attachments/assets/92b6e9d5-1110-4c11-96fd-f2a68c9f3520" />



SafeTensors is excellent at what it does, securing tensor weights, but has no awareness of tokenizer files whatsoever. This creates a systematic blind spot.

**Combined detection gap:**

<img width="558" height="202" alt="Screenshot (1785)" src="https://github.com/user-attachments/assets/debff90e-1c59-43bc-9b35-c37837c84df8" />


---


### Phase 2 — Establishing Scale: How Widespread Is Tokenizer Modification?

Before demonstrating the attack, I needed to understand how often tokenizers are legitimately modified in the wild because a malicious modification needs to be indistinguishable from normal finetuning behavior.

I ran a supply chain analysis across three base model families — `gpt2`, `distilgpt2`, and `EleutherAI/pythia-1b` — downloading and SHA-256 hashing tokenizer files from **138 publicly available derivative models**.

**Results:**


<img width="914" height="618" alt="Screenshot (1786)" src="https://github.com/user-attachments/assets/075806c0-f45e-4ff1-b41d-0724f8218dda" />



**89.1% of derivative models have a tokenizer that differs from their base.** This means a malicious tokenizer modification is effectively invisible by convention, it blends perfectly with the noise of legitimate fine-tuning. There is no anomaly to detect unless you examine *what* changed, not just *that* something changed.

---

### Phase 3 — The Attack: Silent System Prompt Injection via Chat Template

The attack exploits the `chat_template` field in `tokenizer_config.json` (and the separate `chat_template.jinja` file introduced in recent versions of `transformers`). This Jinja2 template controls exactly how `apply_chat_template()` formats messages before they reach the model. It runs **client side, in the user's environment, before any inference happens**.

**Attack vector:**

1. Start from a popular base model (e.g., `gpt2`)
2. Inject a malicious system prompt into the `chat_template`
3. Add custom special tokens to the tokenizer vocabulary
4. Save and upload as a seemingly legitimate finetuned model with an innocuous commit message

The user's workflow never changes. They run `AutoTokenizer.from_pretrained("attacker/model")` and their application is immediately compromised.

**What the model actually receives (the user only typed the final line):**


<img width="1770" height="581" alt="Screenshot (1787)" src="https://github.com/user-attachments/assets/26d5acd6-3846-48f4-9cc9-2e28b61b0430" />



The injected system prompt is prepended silently by the tokenizer template entirely invisible to the end user and to any upstream security scanner.

---

### Phase 4 — Stealth Variant: Base64 Obfuscation

To demonstrate that even basic string-matching detection can be bypassed, a stealth variant was implemented where the payload is **Base64-encoded inside the template**. The injected content no longer appears as readable text in `tokenizer.json` or `tokenizer_config.json`. A `grep` for known jailbreak phrases finds nothing.


<img width="1755" height="169" alt="Screenshot (1788)" src="https://github.com/user-attachments/assets/f1729f55-a0ba-4bf2-8df7-abc9dd99a920" />



The Base64 string decodes to the full malicious system prompt, but is opaque to any static analysis tool that does not execute or decode the template.

---

### Phase 5 — Supply Chain Propagation

This is why the attack is a **supply chain attack** and not a one-off malicious model. The poisoned tokenizer files are just JSON and Jinja2, they propagate naturally.

When anyone:
- Downloads the malicious model and fine-tunes on top of it
- Copies the tokenizer directory into their project
- Runs `tokenizer.save_pretrained()` and re-uploads

...the malicious chat template propagates to the next model, and the next, and the next. **The `model.safetensors` file is never touched. The weights are clean. Only the tokenizer carries the payload.**

Demonstrated propagation path:
```
attacker/gpt2-malicious                    ← payload INJECTED
        │
        └──► user/gpt2-finetuned           ← payload INHERITED (weights new)
                    │
                    └──► org/gpt2-production  ← payload STILL ACTIVE (3 hops)
```
My analysis found that 88.5% of GPT-2 derivative models on Hugging Face had a modified tokenizer vs. the base model, meaning a malicious payload is statistically indistinguishable from legitimate 
finetuning noise at every hop in this chain. There is no standard workflow in any ML pipeline today that inspects or verifies the chat_template field. The payload arrives silently, automatically, 
and indefinitely.

---

## Attack Properties Summary

| Property | Value |
|---|---|
| Modified files | `tokenizer_config.json`, `chat_template.jinja` only |
| Weight files changed | ❌ None |
| Detected by `picklescan` | ❌ No |
| Detected by `SafeTensors` | ❌ No |
| Visible to end user | ❌ No |
| Persists across fine-tuning | ✅ Yes |
| Works on any model family | ✅ Yes (GPT-2, Mistral, LLaMA, etc.) |
| Survives Base64 obfuscation | ✅ Yes |

---

## Immediate Mitigation

If you maintain CI/CD pipelines for ML workflows:

> **Never trust a tokenizer you did not hash-verify against an official source, regardless of what `picklescan` or `SafeTensors` reports.**

Concretely:
1. Pin tokenizer file hashes in your model registry at ingestion time
2. Re-verify hashes before any inference pipeline loads a tokenizer
3. Treat `tokenizer_config.json` and `chat_template.jinja` as executable code, not config

---


## Responsible Disclosure

- All proof-of-concept repositories have been made **private**
- This research has been **reported to the Hugging Face security team**
- No payload in this research was used to target real users or live systems
- Full technical implementation details, evasion analysis, and detection tooling are reserved for the forthcoming research paper

---


[GitHub](https://github.com/shoumik27) · [HuggingFace](https://huggingface.co/Shoumikwho)

> *For collaboration inquiries or responsible disclosure coordination, reach out directly.*
