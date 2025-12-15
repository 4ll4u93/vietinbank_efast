# 🏦 iOS Runtime Cryptography Tracing – Security Research Case Study

> 🔬 **Security Research & Authorized Testing Only**  
> This repository documents a **real-world mobile security research case study** conducted on the iOS application of **one of the four largest commercial banks in Vietnam**.

---

## 🧭 Research Context

This project originates from a security assessment of a **tier-1 Vietnamese banking mobile application**, used by millions of customers nationwide.

### 🔐 Android vs iOS Security Posture

| Platform | Observations |
|--------|--------------|
| 🤖 **Android** | Heavily protected. Multiple layers of anti-tampering, anti-hooking, and runtime integrity checks. All dynamic instrumentation attempts were blocked during research. |
| 🍎 **iOS** | Despite strong surface-level protections (TLS pinning, jailbreak detection, hook detection), the application exposes **multiple architectural weaknesses at runtime**. |

👉 **Key insight**:  
> The iOS version relies on **client-side cryptographic operations** that can be observed **before encryption** and **after decryption**, even when network traffic itself remains fully encrypted.

---

## 🎯 Research Objective

The goal of this research is **not** to bypass protections for exploitation, but to:

- 📖 Study **real-world banking app security design**
- 🧠 Understand **runtime cryptography workflows**
- 🔍 Identify **systemic weaknesses in iOS app architecture**
- 🛡️ Provide insights for **defensive improvement**

---

## 🧠 Core Research Findings (High Level)

### ⚠️ 1. Client-Side Encryption Visibility
Sensitive request payloads are constructed and encrypted **inside Objective-C runtime methods**, making them observable **before cryptographic transformation**.

### ⚠️ 2. Post-Decryption Response Exposure
Decrypted server responses are returned as structured objects **inside the app process**, enabling inspection **after decryption but before UI rendering**.

### ⚠️ 3. Over-Reliance on Hook Detection
The app attempts to block:
- Frida attachment
- Substrate-based hooks
- Dynamic instrumentation

However, **method-level runtime tracing remains possible** via carefully designed tweaks.

### ⚠️ 4. Asymmetric Security Maturity
Android security implementation is significantly more mature and hardened compared to iOS, indicating:
- Platform inconsistency
- Security design drift between teams

---

## 🛠️ Research Methodology

### 🧪 Phase 1 – Runtime Visibility
A Theos-based jailbreak tweak is used to:
- Hook selected Objective-C methods
- Log parameters and return values
- Avoid modifying control flow

### 🕵️ Phase 2 – Objective-C Hunter
A dynamic enumeration module:
- Scans loaded classes at runtime
- Filters by banking / crypto / security keywords
- Dumps method lists for manual analysis

### 🔎 Phase 3 – Cryptographic Flow Mapping
By correlating:
- Runtime logs
- Method names
- Input/output structures

The **full client-side crypto lifecycle** can be reconstructed **without decrypting network traffic**.

---

## 🧬 Example Runtime Observation (Sanitized)

```json
ENCRYPT INPUT:
{
  "username": "...",
  "password": "...",
  "deviceOS": "iOS",
  "requestId": "..."
}

DECRYPT OUTPUT:
{
  "status": {
    "code": "0",
    "message": "User does not exist"
  }
}
