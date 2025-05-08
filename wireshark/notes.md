## Wireshark Notes: Decrypting HTTPS Traffic

### 1. My Notes

1. Install Wireshark:
   ```bash
   sudo apt install wireshark
   ```

2. Create a new group `wireshark` and add the user to it:
   ```bash
   sudo groupadd wireshark
   sudo usermod -aG wireshark $USER
   ```

3. To decrypt HTTPS data in Wireshark:
   - Set the `SSLKEYLOGFILE` environment variable:
     ```bash
     export SSLKEYLOGFILE=/home/premgowda/wireshark-ssl-keys/tlskey
     ```
   - This works for applications like `curl` and browsers like Chrome and Firefox.
   - Example for `curl`:
     ```bash
     curl https://example.com
     ```
     The TLS key will be logged to the specified file.

4. Configure Wireshark to use the key file:
   - Open Wireshark → Edit → Preferences → Protocols → TLS
   - Set "(Pre)-Master-Secret log filename" to `/home/premgowda/wireshark-ssl-keys/tlskey`.

5. For Chrome:
   - Set the `SSLKEYLOGFILE` environment variable in the terminal.
   - Start Chrome from the same terminal:
     ```bash
     google-chrome &
     ```

6. For applications that do not support `SSLKEYLOGFILE` (e.g., Go applications):
   - Use a MITM proxy like `mitmproxy` to capture and decrypt traffic.

### 2. Understanding `SSLKEYLOGFILE`

#### 🔐 What is `SSLKEYLOGFILE`?

`SSLKEYLOGFILE` is an **environment variable** supported by some TLS libraries (notably **NSS**, **OpenSSL**, and **libssl** variants) that logs **pre-master secrets** used during TLS handshakes. These secrets allow tools like Wireshark to decrypt HTTPS traffic in real time.

#### 🧠 Why It Works

When HTTPS is used, the client and server perform a **TLS handshake** to securely exchange encryption keys:

- During this handshake, a **pre-master secret** is generated.
- If `SSLKEYLOGFILE` is set, the client logs these secrets in a plain-text file.
- Wireshark uses these secrets to decrypt the TLS traffic as if it had access to the keys.

This method is non-intrusive and does not require tampering or proxying.

#### 📁 What’s Inside the Log File?

A line in the `SSLKEYLOGFILE` might look like:

```
CLIENT_RANDOM 7e8af13cbf7d4f4fb7f2d8ea6bbf92b0e2f3e2879ec15f3d2f16e46d1c95f28e 612fcf8f3bd09dd237d5dd2fa0e064ee229f9bcd2d4748cb5d9942b6bcbfcfbb
```

- `CLIENT_RANDOM`: the TLS handshake type
- The first hex string is the **random value** sent by the client
- The second is the **pre-master key**

Wireshark matches this info with the captured TLS handshake to decrypt the session.

#### ✅ When Does It Work?

**Works If:**
- The application uses **OpenSSL**, **libssl**, or **NSS**.
- You launch the application with `SSLKEYLOGFILE` set.
- You're capturing traffic from that application in Wireshark.
- You're using Wireshark 2.2+ (with TLS key log support).

**Doesn’t Work If:**
- The app uses a TLS library that **ignores `SSLKEYLOGFILE`** (e.g., custom Go TLS, Java’s native TLS, or Rust’s rustls).
- You launch the app **without the environment variable**.
- The app runs in a **container or as another user**, and you don’t inject the variable there.

### 3. Additional Wireshark Tips

- Use display filters effectively:
  - `http` or `http2`: Show decrypted HTTP/HTTPS traffic.
  - `tcp.port == 443`: Show HTTPS traffic.
  - `ssl`: Show SSL/TLS packets.
  - `http.request.method == "POST"`: Show POST requests.

- Right-click on a TLS packet → Follow → TLS Stream to see the decrypted conversation.

- Wireshark can export decrypted HTTPS content: File → Export Objects → HTTP.

- For continuous monitoring, use the ring buffer capture option to prevent filling your disk.

