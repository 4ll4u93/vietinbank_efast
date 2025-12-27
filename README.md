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

Step-by-step build & integration guide (Vietnamese)
-------------------------------------------------

Below is a practical write-up describing how to build the Theos tweak that embeds `/enc` and `/dec` endpoints and how to use it with a proxy for request/response translation. The instructions assume a typical Theos workflow.

Prerequisites
- macOS (recommended) with Xcode command-line tools (or a cross-build environment that supports Theos).
- Theos installed and configured. If you prefer rootless packaging on-device, set `THEOS=/var/theos` on device or your cross-build environment.
- A jailbroken iPhone for testing (or a device + rootless Theos tooling). Device must be reachable on the local network from your analyst machine.
- `dpkg` / `scp` / `ssh` access to the device for installing packages.
- `ldid` or codesigning tools if building for a device requiring signing.

Files to check in the repo
- `my_tweak/tweak_full.m` — contains the hooks, webserver startup (`startWebServer()`), and handlers for `/enc` and `/dec`.
- `Makefile` inside `my_tweak` (Theos Makefile) used to build the tweak.
- `mitm_proxy_doan.py` (or `mitmproxy1.py`) — example proxy addon to integrate with the tweak.

Step-by-step build and install
1) Prepare the tweak source
   - Confirm `my_tweak/tweak_full.m` implements:
     - Hook for `GET_ENC_STRING_NEW:` that captures `sharedSecurityInstance` and returns the original encryption result.
     - Hook for `GET_DEC_STRING_NEW:` that calls original decrypt and logs/returns the result.
     - `startWebServer()` which registers `/enc` and `/dec` using `GCDWebServer` and starts it on port `1997`.

2) Ensure dependencies are available
   - Add `GCDWebServer` source or import into the `my_tweak` vendor folder. The Theos target must be able to link or compile the GCDWebServer sources.
   - In your `Makefile`, ensure required frameworks are declared (example):

```
TWEAK_NAME = eFastTrace
eFastTrace_FILES = tweak_full.m
eFastTrace_FRAMEWORKS = Foundation Security
eFastTrace_PRIVATE_FRAMEWORKS = GCDWebServer
```

Note: The exact Makefile syntax depends on your Theos / Logos setup — add `GCDWebServer` sources under `vendor` and include them in the build if not available as a framework.

3) Set THEOS and build

```bash
export THEOS=/path/to/theos   # or /var/mobile/theos for on-device build
cd my_tweak
make clean
make package ROOTLESS=1      # use ROOTLESS=1 if building rootless package
```

4) Install on-device

```bash
scp packages/*.deb root@<device-ip>:/tmp/
ssh root@<device-ip> "dpkg -i /tmp/*.deb"
# Restart the target app or respring if needed
```

5) Verify webserver and hooks
- On the device, check logs for the webserver start message: `oslog | grep DOANNGUYEN` or `oslog | grep eFastTrace`.
- If `sharedSecurityInstance` is not captured, open the app and perform the action that instantiates `SecurityPackage` so hooks attach.

6) Configure your proxy machine
- Ensure your analyst machine (running mitmproxy/Burp) and the device are on the same network.
- Update `mitm_proxy_doan.py` or the example mitmproxy addon with the device IP (e.g., `IPHONE_URL = "http://192.168.5.193:1997"`).

7) Run mitmproxy and proxy the app traffic
- Start `mitmproxy` or `mitmdump` with your addon loaded. Example:

```bash
# run mitmproxy with your addon
mitmdump -s mitm_proxy_doan.py
```

- Confirm that outgoing requests for configured host (e.g., `efastmb.vietinbank.vn`) are forwarded to `/enc` and replaced with the encrypted bytes returned by the tweak.

Troubleshooting
- Webserver fails to bind: ensure `GCDWebServerOption_BindToLocalhost: @NO` is used intentionally; firewall or sandbox restrictions can block binding to non-localhost addresses.
- `sharedSecurityInstance` null: interact with the app UI until the class that implements `GET_ENC_STRING_NEW:` is loaded; then retry the HTTP call to `/enc`.
- `original_impl` or `original_dec_impl` null: verify the hook registration succeeded for `SecurityPackage` in logs at startup.
- Use `oslog` or device `syslog` to view messages: `oslog | grep DOANNGUYEN`.

Data flow model (diagram)
-------------------------

High-level sequence (proxy-assisted mode):

Analyst machine (Burp / mitmproxy)
     |
   send plaintext JSON
     |
   -> mitmproxy addon POST /enc -> `http://<iphone-ip>:1997/enc`
     |
   iPhone GCDWebServer `/enc` calls `original_impl(sharedSecurityInstance, sel, json)`
     |
   App's `GET_ENC_STRING_NEW:` returns encrypted string
     |
   iPhone returns encrypted string to mitmproxy
     |
   mitmproxy gzips (if required) and forwards to bank server

Bank Server
   |
   Encrypted response
   |
   -> mitmproxy receives encrypted bytes -> mitmproxy POST /dec -> `http://<iphone-ip>:1997/dec`
   |
   iPhone `/dec` handler calls `original_dec_impl(sharedSecurityInstance, sel, encrypted_string)`
   |
   Decrypted object returned to mitmproxy
   |
   mitmproxy replaces proxied response body with plaintext JSON for analyst view

ASCII flow diagram (simplified):

[Burp/mitmproxy] --(plaintext JSON)--> [mitmproxy addon] --(POST /enc)--> [iPhone GCDWebServer /enc]
    --(call original_impl)--> [App: GET_ENC_STRING_NEW] --(encrypted string)--> [iPhone] --(response)--> [mitmproxy]
    --(forward to bank)--> [Bank]

[Bank] --(encrypted response)--> [mitmproxy] --(POST /dec)--> [iPhone GCDWebServer /dec]
    --(call original_dec_impl)--> [App: GET_DEC_STRING_NEW] --(plaintext)--> [iPhone] --(response)--> [mitmproxy]
    --(replace response)--> [Burp/analyst view]

Security & ethics reminder
- Only run these steps in authorized testing environments.

Notes & next steps
- I can (optionally) add a sample `Makefile` snippet tailored to your `my_tweak` layout, or update `mitm_proxy_doan.py` to match the exact `/enc` and `/dec` request/response shapes used by your tweak.


