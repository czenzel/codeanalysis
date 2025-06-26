# gosom/google‑maps‑scraper

Repository: https://github.com/gosom/google‑maps‑scraper

## 1. Arbitrary file-write via **create\_new\_category**

**Location in source**: lines **653–660**

```python
    # line 653
    filename = f"{next_num}_keywords_{category_name}.txt"
    # line 654
    filepath = os.path.join('keywords', filename)
    …
    # line 660
    with open(filepath, 'w', encoding='utf-8') as f:
        f.write("")  # creates the file
```

**Why it’s exploitable**

* `category_name` comes straight from the user.
* If you enter an absolute path (e.g. `/tmp/hacked`), then

  ```python
  os.path.join('keywords', '/tmp/hacked.txt')  
  # => '/tmp/hacked.txt'
  ```

  and the app will happily overwrite or create `/tmp/hacked.txt`, as long as it has permissions.

## 2. Arbitrary file-write via **create\_new\_location**

**Location in source**: lines **806–813**

```python
    # line 806
    filename = f"{next_num}_location_{location_name}.txt"
    # line 807
    filepath = os.path.join('location', filename)
    …
    # line 813
    with open(filepath, 'w', encoding='utf-8') as f:
        f.write(f"{zoom}\n{lat}\n{lon}")
```

**Why it’s exploitable**

* Same pattern: an attacker can set `location_name` to an absolute path (e.g. `/etc/cron.d/evil`), causing the scraper to write to `/etc/cron.d/evil.txt`.

## 3. Server-Side Request Forgery (SSRF) via **execute\_jobs**

**Location in source**: lines **505–506**

```python
    # line 505
    host = self.host_var.get()
    # line 506
    api_url = f"{host}/api/v1/jobs"
    response = requests.post(api_url, json=payload)
```

**Why it’s exploitable**

* The “Host” field is entirely user-controlled.
* You can point it at internal-only services (e.g. AWS metadata at `http://169.254.169.254`) and the scraper will blindly send POST/GET there, potentially leaking sensitive data.

### How to perform the file-write exploit

1. **Launch** the GUI:

   ```bash
   python scraper_gui.py
   ```
2. **Create a malicious category**

   * Switch to the **“Categorías y Keywords”** tab.
   * In **“Agregar Nueva Categoría”**, set **Nombre** = `/tmp/pwn`, then click **Crear Categoría**.
   * Verify on disk:

     ```bash
     ls -l /tmp/pwn.txt
     ```
3. **Overwrite a system file** (if you have permission)

   * Go to the **“Localizaciones”** tab.
   * In **“Agregar Nueva Localización”** set **Nombre** = `/etc/cron.d/evil`, **Zoom/Lat/Lon** to any values, then **Crear Localización**.
   * Check:

     ```bash
     cat /etc/cron.d/evil.txt
     ```
   * You’ve just planted `zoom\nlat\nlon` into `/etc/cron.d/evil.txt`.

### How to perform the SSRF exploit

1. **In the “Configuración” tab**, change **Host** from `http://localhost:8080` to:

   ```
   http://169.254.169.254/latest/meta-data
   ```
2. **Select at least one** category and location (you can even create dummy ones as above).
3. **Click “Run”**.
4. The scraper will `POST http://169.254.169.254/latest/meta-data/api/v1/jobs` and then `GET http://169.254.169.254/latest/meta-data/api/v1/jobs/<id>`—potentially exposing whatever that metadata endpoint returns.

## Recommendations

1. **Validate & sanitize** `category_name` and `location_name`:

   * Reject any input with leading `/`, `\`, or `..` segments.
   * Allow only a strict whitelist of characters (e.g. `^[A-Za-z0-9_-]+$`).
2. **Build file paths safely**:

   * After `join`, call `os.path.abspath` and verify it still lives under your intended directory.
3. **Whitelist “host”** values:

   * Allow only known, internal/external domains or IP ranges.
   * Enforce HTTPS and validate certificates.
4. **Consider least-privilege**:

   * Run the GUI without write permissions to sensitive system directories.

Applying these fixes will block both the directory-traversal file writes and the SSRF attacks.
