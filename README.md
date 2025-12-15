# 🏦 iOS Banking App Runtime Cryptography Tracing (Security Research)

> 🔬 **Authorized Security Research Only**  
> This repository documents a **real-world security research case study** on the iOS application of **one of the four largest commercial banks in Vietnam**.

This project is **not an exploit**. It is a **runtime analysis framework** designed to study **client-side cryptographic behavior** in a highly sensitive mobile banking application.

---

## 🧭 Background & Motivation

During the security assessment, a significant difference was observed between Android and iOS implementations of the same banking application.

### 🔐 Platform Security Comparison

| Platform | Observations |
|--------|--------------|
| 🤖 Android | Highly hardened. Multiple layers of anti-tampering, anti-hooking, and runtime integrity checks. Dynamic instrumentation was effectively blocked. |
| 🍎 iOS | Despite TLS pinning, jailbreak detection, and hook detection, **runtime Objective-C logic remains observable**, exposing sensitive cryptographic workflows. |

**Key insight:**  
Even when **network traffic remains fully encrypted**, sensitive data can still be accessed **inside the app process**.

---

## 🎯 Research Objective

This research aims to:

- Observe plaintext request data **before encryption**
- Observe plaintext response data **after decryption**
- Discover cryptographic and security-related classes dynamically
- Analyze architectural weaknesses in real-world iOS banking apps
- Provide defensive insights for security improvement

---

## 🧱 High-Level Architecture

### Request Flow

```text
User Input
   │
   ▼
Request Object Builder
   │   ◄── Runtime Hook (Tracer)
   ▼
Client-Side Encryption
   │
   ▼
Encrypted HTTPS Traffic
```

### Response Flow

```text
Encrypted HTTPS Response
   │
   ▼
Client-Side Decryption
   │   ◄── Runtime Hook (Tracer)
   ▼
Plain Objective-C Object
   │
   ▼
UI Rendering
```

---

## 🧪 Module 1: Runtime Cryptography Tracer

### 📌 Purpose

The **Tracer** hooks specific Objective-C methods responsible for encrypting requests and decrypting responses.

It logs:
- Input parameters (before encryption)
- Return values (after decryption)

---

### 🧠 Core Hook Implementation

```objc
%hook SecurityPackage

- (id)GET_ENC_STRING_NEW:(id)arg1 {
    NSString *cleanLog = convertObjToString(arg1);
    LOG(@"ENCRYPT INPUT: %@", cleanLog);
    return %orig;
}

- (id)GET_DEC_STRING_NEW:(id)arg1 {
    id result = %orig;
    NSString *cleanLog = convertObjToString(result);
    LOG(@"DECRYPT OUTPUT: %@", cleanLog);
    return result;
}

%end
```

---

### 🔄 Safe Object Serialization

```objc
static NSString *convertObjToString(id obj) {
    if (!obj) return @"(null)";

    if ([obj isKindOfClass:[NSString class]]) {
        return (NSString *)obj;
    }

    if ([obj isKindOfClass:[NSDictionary class]] ||
        [obj isKindOfClass:[NSArray class]]) {
        NSData *jsonData =
            [NSJSONSerialization dataWithJSONObject:obj options:0 error:nil];
        return [[NSString alloc] initWithData:jsonData
                                     encoding:NSUTF8StringEncoding];
    }

    if ([obj isKindOfClass:[NSData class]]) {
        return [[NSString alloc] initWithData:(NSData *)obj
                                     encoding:NSUTF8StringEncoding];
    }

    return [NSString stringWithFormat:@"%@", obj];
}
```

---

### 📜 Example Runtime Logs

```text
[eFastTrace] ENCRYPT INPUT:
{"username":"test","password":"***","deviceOS":"iOS"}

[eFastTrace] DECRYPT OUTPUT:
{"status":{"code":"0","message":"User does not exist"}}
```

---

## 🕵️ Module 2: Objective-C Runtime Hunter

### 📌 Purpose

The **Hunter** dynamically enumerates Objective-C classes at runtime to discover encryption managers, security helpers, and internal SDK logic.

---

### 🔍 Class Scanning Logic

```objc
void HunterScan() {
    int numClasses = objc_getClassList(NULL, 0);
    Class *classes = (Class *)malloc(sizeof(Class) * numClasses);
    objc_getClassList(classes, numClasses);

    for (int i = 0; i < numClasses; i++) {
        NSString *name = NSStringFromClass(classes[i]);

        BOOL suspicious =
            [name hasPrefix:@"VTB"] ||
            [name containsString:@"Security"] ||
            [name containsString:@"Encrypt"] ||
            [name containsString:@"Crypto"];

        if (suspicious) {
            DumpMethods(classes[i]);
        }
    }
    free(classes);
}
```

---

### 🧬 Method Dumping

```objc
void DumpMethods(Class cls) {
    unsigned int count = 0;
    Method *methods = class_copyMethodList(cls, &count);

    for (int i = 0; i < count; i++) {
        LOG(@"func: - %@", NSStringFromSelector(method_getName(methods[i])));
    }

    free(methods);
}
```

---

### ⏱️ Safe Execution Timing

```objc
%hook UIViewController
- (void)viewDidAppear:(BOOL)animated {
    %orig;
    static dispatch_once_t once;
    dispatch_once(&once, ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 5*NSEC_PER_SEC),
                       dispatch_get_main_queue(), ^{
            HunterScan();
        });
    });
}
%end
```

---

## 🧱 Project Structure

```text
.
├── Makefile
├── Tweak.xm
├── control
├── packages/
```

---

## ⚙️ Build & Package (Rootless)

```bash
export THEOS=/var/mobile/theos
make package ROOTLESS=1
```

---

## 📲 Install on Device

```bash
dpkg -i packages/*.deb
```

Restart the target app or respring if required.

---

## 📜 Viewing Logs

```bash
oslog | grep eFastTrace
oslog | grep eFastHunter
```

---

## ⚠️ Ethical & Legal Disclaimer

- No authentication bypass
- No server compromise
- No cryptographic break
- No real user data extraction

This work highlights **design-level risks**, not exploitation techniques.

---

## 🧠 Final Takeaway

> If sensitive logic exists on the client, it can be observed — regardless of TLS or pinning.

This is a **systemic architectural concern** for high-risk mobile applications such as banking apps.

---

## 📬 Responsible Disclosure

If you are part of the affected organization, please follow your internal responsible disclosure process.


