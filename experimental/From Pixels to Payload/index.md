---
layout: default
title: From Pixels to Payload
---

# **From Pixels to Payload**

---
### 🎯 **Goal**

This writeup documents a personal learning experiment aimed at understanding how binary payloads can be stealthily embedded in images and executed entirely in memory with minimal forensic footprint.

The core idea is to **explore the combination of steganography and in-memory execution**, with the long-term goal of building a custom DLL that acts as a stealthy payload loader via **DLL hijacking**. This would allow code execution inside a trusted process without dropping files or triggering UAC. Or atleast thats the plan 

To begin, I prototyped the payload embedding stage in Python to experiment with:

- **LSB steganography** for hiding payloads in PNG images
    
- **Base64 encoding / decoding** as a flexible wrapper
    
- **In-memory shellcode execution** using Python’s `ctypes` (for testing purposes)

Although the extractor and runtime code are currently implemented in Python, the ultimate goal is to **rebuild that part in C++**, both for performance and for integration into a real DLL hijacking scenario.

This project is ongoing and even if some parts don't fully achieve the intended stealth or reliability, the process of testing, analyzing, and improving is the actual objective.  
It's as much about **understanding trade-offs** and limitations as it is about reaching a "perfect" PoC.

---
### 📦 Components

- `embed.py` → Encodes a shellcode or any binary payload into the LSBs of a PNG image
    
- `extract.py` → Recovers the payload, decodes it, and executes it in memory

---
### ⚙️ Technical Workflow

**Embedding Phase**  
![Embedding Flowchart](_screenshots/flowChartEmbed.drawio.png)

**Extraction Phase**  
![Extraction Flowchart](_screenshots/extract.drawio.png)


---
💡 Why LSB + Memory Execution?


---
🔐 Tested Payloads


---
🧪 Output Example
