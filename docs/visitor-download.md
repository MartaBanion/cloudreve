# Anonymous Visitor Download Setup

Enable file download for unauthenticated (guest/visitor) users in Cloudreve.

## Overview

By default, Cloudreve requires users to log in before they can download shared files. This guide explains how to configure the **Anonymous** user group so that visitors can download files directly from share links without any login.

## How It Works

Cloudreve uses a **Boolset** (bitmask-based permission) system for user groups. The Anonymous group controls what unauthenticated visitors can do. By setting the correct permission bits, you grant download capabilities to anyone with a share link.

## Permission Bits

For the **User Group** permission scope, the relevant bits are:

| Bit | Description | Required? |
|-----|-------------|-----------|
| 0   | Is administrator | ❌ |
| 1   | Is anonymous user group | ✅ (must be set for guest users) |
| 2   | Can share files | ❌ |
| 3   | Can access WebDAV | ❌ |
| 4   | Can perform server-side batch download | Optional |
| 5   | Can execute archive compression tasks | Optional |
| 6   | Can enable WebDAV proxy | ❌ |
| 7   | Can download shares from others | ✅ **Required for download** |
| 8   | Can download shares for free | ✅ **Recommended** (bypasses credit/point cost) |
| 9   | Can perform remote downloads | ❌ |
| 10  | Can transfer storage policy | ❌ |
| 11  | Use redirected source link | Optional |

## Database Setup

Execute the following SQL to set appropriate permissions for the Anonymous group:

```sql
-- Enable: anonymous flag (bit 1) + download shares (bit 7) + free download (bit 8)
-- Byte 0: 0x82 (bits 1 and 7)
-- Byte 1: 0x01 (bit 8)
UPDATE groups
SET permissions = x'8201',
    updated_at  = datetime('now')
WHERE id = 3;
```

This translates to:
- `0x82` → `10000010` → Bits 1 (anonymous) + 7 (download shares) enabled
- `0x01` → `00000001` → Bit 8 (free download) enabled

## Site URL Configuration

Cloudreve must know its public-facing URL to generate correct share links.

### Via Database

```sql
UPDATE settings
SET value = 'http://your-public-ip:5212',
    updated_at = datetime('now')
WHERE name = 'siteURL';
```

### Via Config File (`conf.ini`)

```ini
[System]
SiteURL = http://your-public-ip:5212
```

> **Important:** Replace `your-public-ip` with your actual server IP or domain name. If the `SiteURL` is set to `localhost`, generated share links will not work for external users.

## User-Facing Steps

Once permissions and site URL are configured:

1. **Upload a file** to Cloudreve
2. **Right-click** the file → **Share**
3. In the share dialog:
   - ✅ Enable **Allow Download**
   - ❌ Disable **Extraction Code** (password) for direct access
   - Set expiration as needed
4. **Copy the share link** (format: `http://your-server:5212/s/xxxxx`)
5. Anyone with this link can **view and download** the file without logging in

## Verification

To verify the setup works:

1. Open the share link in an **incognito/private browser window**
2. You should see the file preview page
3. Click the download button — the file should download without prompting for login
4. Run the following SQL to check permissions:

```sql
SELECT id, name, hex(permissions) FROM groups;
```

Expected output:

```
1|Admin|FD1A01
2|User|8408
3|Anonymous|8201
```

## Notes

- The Anonymous group permission changes take effect **immediately** after database update (no server restart required if using SQLite in most cases, though a restart ensures consistency).
- Share links generated before enabling these permissions may not work — **re-generate** share links after making changes.
- Ensure your server firewall allows inbound connections on the Cloudreve port (default: `5212`).
- For production deployments, consider adding rate limiting to prevent abuse of public download links.
