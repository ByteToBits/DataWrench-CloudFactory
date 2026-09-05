# SECS-II message JSON Field Reference

## Header fields

### **doc**
Human-readable description. Not part of the actual SECS-II wire protocol, it is for pure documentation metadata so the message's purpose is clear without memorizing the **SVID or TIACK**.

### **stream**
The SECS-II **Stream number**, the top-level category a message belongs to.

| Stream | Category |
|---|---|
| 0 | Reserved (no stream) |
| 1 | Equipment status |
| 2 | Equipment control and diagnostics |
| 3 | Material status |
| 4 | Material control |
| 5 | Alarm management / exception handling |
| 6 | Data collection / event reporting |
| 7 | Process program (recipe) management |
| 8 | Control program transfer |
| 9 | System errors |
| 10 | Terminal services (operator messages) |
| 11 | Withdrawn (formerly host file services) |
| 12 | Wafer mapping |
| 13 | Data set transfers |
| 14 | Object services |
| 15 | Recipe management |
| 16 | Processing management (job control) |
| 17 | Equipment control and diagnostics, enhanced |
| 18 | Subsystem control and data |
| 19 | Recipe and parameter management |
| 21 | Item transfer |
| 64–127 | User-defined |

### **function**
The **Function number** within that stream, identifies the specific message. 

Strict rule: Requests are always odd-numbered, Responses are always the request's function + 1.

- **S1F1** (request) → **S1F2** (response)
- **S2F31** (request) → **S2F32** (response)

### **reply**
**true** if the message expects a reply (a primary/request message), **false**
if it's the reply itself (a secondary/response message).

## Body item fields

### **format**
The SECS-II data type of that value.

| Code | Meaning |
|---|---|
| **A** | ASCII string |
| **B** | Binary (raw bytes) |
| **BOOLEAN** | True/false |
| **U1** / **U2** / **U4** / **U8** | Unsigned integer, 1/2/4/8 bytes |
| **I1** / **I2** / **I4** / **I8** | Signed integer, 1/2/4/8 bytes |
| **F4** / **F8** | Floating point, 4/8 bytes |
| **L** | List — a container holding other typed items, this is what makes messages nest (example **S1F12** list-of-lists, one sub-list per status variable) |

### **value**
The actual data, typed according to format**.

## Example

**{"format": "U4", "value": 4}** is a 4-byte unsigned integer. The SECS-II equivalent of what your own FOUP RFID spec calls **UINT 32** in its "Data Type" column, just using SECS-II's own naming convention.
