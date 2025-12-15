Markdown

# 🛡️ Vietinbank eFast Application Analysis & Bypass Tweak (iOS/Jailbreak)

This repository contains the analysis methodology and custom tweaks developed to overcome the sophisticated anti-debugging, anti-hooking, and SSL Pinning mechanisms implemented in the Vietinbank eFast mobile banking application on iOS.

The primary goal of this project was to successfully capture and log the application's clean JSON payloads (both requests and responses) before they are encrypted and after they are decrypted at the application layer.

## 🚀 The Challenge

The Vietinbank eFast application employs multiple layers of protection that necessitate an advanced reverse engineering approach:

1.  **Anti-Debugging/Anti-Hooking:** The application actively detects and terminates when standard dynamic analysis tools (like Frida or debuggers) are attached or when injection frameworks (like Cydia Substrate) are used.
2.  **SSL Pinning (Network Layer):** Traditional network interception using proxies (e.g., Burp Suite, Mitmproxy) is prevented due to strict certificate validation checks.
3.  **Application-Level Encryption (Data Layer):** Even after bypassing SSL Pinning, the payload data exchanged over the network remains encrypted using a proprietary or custom cryptographic scheme, preventing direct viewing of transactions.

## 💡 Methodology & Technical Steps

The analysis was executed in a three-phase escalation:

### Phase 1: Bypassing Anti-Hooking & Function Tracing

* **Technique:** Instead of relying on blocked dynamic tools, a custom Dylib Injection (or Tweak) was used for **Runtime Objective-C Analysis**.
* **Purpose:** To programmatically iterate through all loaded classes and methods (using tools like `tweak_function_trace.m`) to identify classes responsible for security and network handling.
* **Result:** Identification of the critical class, `SecurityPackage`, which contained suspicious methods related to data transformation.

### Phase 2: Kernel-Level SSL Pinning Bypass

* **Technique:** The standard SSL Pinning bypass was insufficient. A modified **SSL Kill Switch** was built and deployed to run at a low-level, potentially integrated with core system or kernel components.
* **Purpose:** To unconditionally disable the `SecureTransport` and `CFNetwork` certificate checking routines, ensuring all TLS traffic could be intercepted by an external proxy.
* **Result:** Successful man-in-the-middle (MITM) interception of the encrypted traffic, confirming only application-level encryption remained.

### Phase 3: Hooking Application-Layer Cryptography

* **Technique:** A final, targeted tweak (`teak_get_enc_dec.m`) was developed to hook the discovered methods in the `SecurityPackage` class.
* **Goal:** To extract the Objective-C object containing the data *just before* it is passed to the encryption function, and *just after* it is returned from the decryption function.
* **Target Methods (Hooked in `SecurityPackage`):**
    * `- (id)GET_ENC_STRING_NEW:(id)arg1`
    * `- (id)GET_DEC_STRING_NEW:(id)arg1`
* **Success:** The tweak successfully logged the clean JSON data to the system log (`oslog`).

## 📈 Analysis Flow Diagram

This diagram illustrates the successful data interception flow using the custom tweaks:

```mermaid
graph TD
    A[Vietinbank eFast App] -->|1. Clean JSON Payload| B(SecurityPackage Class);
    B -->|2. Hooked: GET_ENC_STRING_NEW:| C{Tweak: teak_get_enc_dec.m};
    C -->|3. LOG: Clean JSON (Pre-Enc)| D[System Log / oslog];
    B -->|4. Encrypt Data| E[Encrypted Payload];
    E -->|5. TLS Layer (Bypassed Pinning)| F[Network Proxy (Burp/Mitm)];
    F -->|6. Encrypted Response| G[App Receive];
    G -->|7. Hooked: GET_DEC_STRING_NEW:| H{Tweak: teak_get_enc_dec.m};
    H -->|8. LOG: Clean JSON (Post-Dec)| D;
    H -->|9. Decrypted Payload| A;
🛠️ Implementation Details
teak_get_enc_dec.m (The Data Extractor)
This is the core tweak that provides visibility into the application's data exchange.

Objective-C

%hook SecurityPackage

- (id)GET_ENC_STRING_NEW:(id)arg1 {
    // Capture and log data *before* it is encrypted (Request payload)
    NSString *cleanLog = convertObjToString(arg1);
    LOG(@"ENCRYPT INPUT: %@", cleanLog); 
    return %orig;
}

- (id)GET_DEC_STRING_NEW:(id)arg1 {
    id result = %orig; // Execute the decryption function
    // Capture and log data *after* it is decrypted (Response payload)
    NSString *cleanLog = convertObjToString(result);
    LOG(@"DECRYPT OUTPUT: %@", cleanLog);
    return result;
}

%end
Example Log Output (Proof of Concept):

Đoạn mã

<Notice> VietinbankeFast[16447]: [eFastTrace] ENCRYPT INPUT: {"mid":"1","version":"3.4.0.0","requestType":"login"...}
tweak_function_trace.m (The Hunter)
This file contains the initial code to dynamically dump methods of Objective-C classes, which was crucial for discovering the SecurityPackage methods.

⏭️ Future Work
With the ability to monitor clean requests and responses, future analysis can focus on:

Vulnerability Testing: Utilizing the clean payloads to perform business logic testing, IDOR checks, and parameter tampering attacks.

Crypto Re-Implementation: Using the data flow knowledge (and captured artifacts like RSA_KEY.png) to reverse engineer the full cryptographic scheme (key/IV derivation, cipher mode, padding) for re-implementation in a standalone tool (e.g., Burp Extender or Python script).