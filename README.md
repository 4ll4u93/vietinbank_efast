# iOS Runtime Cryptography Tracer — Detailed README

WARNING: Authorized security research only. Do not use this repository to attack systems you do not own or have explicit permission to test.

This repository contains a runtime analysis framework and supporting Theos tweak that were used during a security assessment of an iOS banking application. The tools are designed to observe client-side cryptographic workflows inside the app process (request encryption and response decryption). They are intended for defensive research and responsible disclosure.

Contents of this README:
- Overview and goals
- Architecture (how request/response tracing works)
- Tweak internals (what hooks are installed and why)
- Local helper web service endpoints (`/enc` and `/dec`)
- Building and packaging (Theos)
- Usage with proxy tools (mitmproxy / Burp) — example scripts
- Safety, limitations, and mitigation guidance
- Responsible disclosure and legal notes

Overview
--------
Purpose: capture plaintext data before client-side encryption and after client-side decryption by hooking Objective-C methods at runtime inside the target app process. This allows researchers to:

- Inspect request payloads prior to encryption
- Inspect response payloads after decryption
- Enumerate suspicious Objective-C classes and methods to locate crypto and security logic
- Provide defensive recommendations based on observed client behavior

High-level flow
----------------

Request path (instrumented):

User input -> Request object builder -> [HOOK: tracer reads plaintext] -> Client-side encryption -> Encrypted HTTPS traffic

Response path (instrumented):

Encrypted HTTPS response -> Client-side decryption -> [HOOK: tracer reads decrypted object] -> UI rendering

Key components
--------------

- Theos tweak (Objective-C / C code) — installs hooks into encryption/decryption APIs and networking hooks (e.g., `GET_ENC_STRING_NEW:`, `GET_DEC_STRING_NEW:`).
- Local web server embedded inside the tweak (GCDWebServer) that exposes two endpoints used by external tooling:
  - POST /enc  — call the app's encryption method with supplied JSON and return the encrypted string
  - POST /dec  — call the app's decryption method with supplied encrypted string and return the decrypted object
- Proxy-side helper script (Python for mitmproxy) — routes outgoing JSON to the iPhone service for encryption and routes incoming encrypted responses to the iPhone service for decryption.

Important tweak behaviors (summary)
----------------------------------

- Hooks key functions in Security / BoringSSL / SecureTransport / NSURLSession stacks to disable certificate checks (legacy SSL Kill Switch functionality present but separate from the tracer parts).
- Hooks application-specific class `SecurityPackage` (if present) to intercept `GET_ENC_STRING_NEW:` and `GET_DEC_STRING_NEW:`. When the encrypt hook runs it captures plaintext JSON into a global buffer; the code also captures a `sharedSecurityInstance` reference so the web server can call the original encryption/decryption functions.
- Hooks `-[NSMutableURLRequest setHTTPBody:]` to optionally patch outgoing request bodies: for local Burp/mitmproxy workflows it can replace the encrypted body with previously-captured plaintext so the analyst can view JSON in Burp.
- Runs an embedded GCDWebServer on port 1997 bound to all interfaces (configurable in code).

Web server API
--------------

Both endpoints expect and return JSON. Example payloads:

- POST /enc
  - Request JSON: arbitrary object (NSDictionary/NSArray). The tweak will call the app encryption method with that object and return the resulting encrypted string.
  - Response JSON: {"status":"success","result": <encrypted-string>}

- POST /dec
  - Request JSON: {"data": "<encrypted-string>"}
  - Response JSON: {"status":"success","result": <decrypted-object | null>}

Implementation notes
--------------------

- The tweak captures a reference to the object instance that implements the encryption/decryption methods so the web server can call the original Objective-C method implementations directly.
- For request-patching workflows the tweak uses a global NSData buffer (`global_PlaintextBody`) that is filled when the encryption hook receives a JSON object; the `setHTTPBody:` hook will replace outgoing encrypted bytes with this plaintext when the request target matches configured host patterns.
- The decrypt hook logs input (encrypted string) and the decrypted output (dictionary or string) and returns the original `id` result.

Building and packaging
----------------------

Requirements:
- A Theos development environment for iOS tweaks (Theos vX, SDKs as required)
- Xcode toolchain on macOS or a compatible cross-build environment

Typical build commands (rootless packaging):

```bash
export THEOS=/var/mobile/theos
make package ROOTLESS=1
```

Install produced package on device:

```bash
dpkg -i packages/*.deb
```

Restart the target app or respring if required.

Using with mitmproxy / Burp
---------------------------

Typical researcher workflow:

1. Install the tweak package on a jailbroken instrumented iPhone and start the target banking app.
2. Run the proxy (mitmproxy or Burp) on the analyst machine and ensure it can forward requests to the iPhone's local HTTP service (the tweak runs a webserver on port 1997).
3. Use a short mitmproxy addon script to route plaintext request JSONs to `http://<iphone-ip>:1997/enc` and replace the request body with the returned encrypted string (optionally gzip it). For responses, route the encrypted bytes to `http://<iphone-ip>:1997/dec` and replace the proxied response body with the returned decrypted JSON.

Example mitmproxy handler (outline — adapt IP and headers):

```python
from mitmproxy import http
import requests, json, gzip

IPHONE_URL = "http://192.168.5.193:1997"

def request(flow: http.HTTPFlow):
    if "efastmb.vietinbank.vn" in flow.request.pretty_host:
        # remove Content-Encoding header if present
        flow.request.headers.pop("Content-Encoding", None)
        plaintext = flow.request.get_text()
        payload = json.loads(plaintext)
        res = requests.post(f"{IPHONE_URL}/enc", json=payload, timeout=10)
        encrypted = res.json().get("result")
        flow.request.content = gzip.compress(encrypted.encode("utf-8"))
        flow.request.headers["Content-Encoding"] = "gzip"

def response(flow: http.HTTPFlow):
    if "efastmb.vietinbank.vn" in flow.request.pretty_host:
        content = flow.response.content
        # detect gzip and decompress; unwrap quoted JSON strings when needed
        # send to /dec and replace flow.response.text with returned JSON
```

The repository contains a working mitmproxy script (see `mitm_proxy_doan.py` or `mitmproxy1.py`) to use as a starting point. Adapt the iPhone IP, host checks, headers, and encoding rules to your environment.

Logs and debugging
------------------

- The tweak writes logs with a distinct tag (e.g., `DOANNGUYEN` or `eFastTrace`). Use `oslog` or device syslog to inspect messages:

```bash
oslog | grep eFastTrace
oslog | grep DOANNGUYEN
```

- GCDWebServer logs incoming requests and will report errors if `sharedSecurityInstance` or the original IMP is not yet captured. Interact with the app UI to ensure the instance is created before calling the web endpoints.

Safety, limitations, and ethics
------------------------------

- This tool inspects application-internal state and therefore requires the tester to have explicit authorization to test the application.
- The tweak disables various SSL/TLS verification functions — only use these components in controlled, authorized research contexts. The presence of disabling hooks in this repository is for historical and research reasons and not required to use the encryption/decryption tracer.
- Some apps perform additional integrity checks, anti-hooking, or obfuscation; this technique will not always succeed and can crash the app if misapplied.
- The webserver binds to all interfaces by default; on shared networks ensure firewalling or use local-only binds for safety.

Mitigations for app developers
-----------------------------

To reduce the risk of client-side secrets disclosure, consider:

- Move sensitive crypto operations to a secure backend where possible.
- Avoid performing high-value key operations purely in application logic.
- Harden the app against runtime instrumentation (anti-debug, integrity checks), but note that motivated attackers can bypass client-side protections.
- Use hardware-backed key stores (Secure Enclave) and design protocols so that raw plaintext sensitive data is not available in memory post-encryption.

Responsible disclosure
---------------------

If you are a vendor or operator with concerns about this research, follow your internal responsible disclosure process or contact the researcher listed in the project (if any). Do not use the materials here for unauthorized access.

Files of interest in this repository
-----------------------------------

- `my_tweak/tweak_full.m` — The Theos tweak that installs the hooks and runs the bundled GCDWebServer.
- `mitm_proxy_doan.py`, `mitmproxy1.py`, `mitmproxy2.py` — Example proxy scripts for automating request/response forwarding to the iPhone service.
- `README.md` — (this file) high-level documentation and usage guidance.

Contact and further work
------------------------

If you want a custom walkthrough, CI packaging, or help adapting the mitmproxy scripts to your environment, open an issue or request a follow-up.

License & Disclaimer
--------------------

This repository is provided for authorized security research and education. Use at your own risk. No warranty is provided. The authors are not responsible for misuse.

