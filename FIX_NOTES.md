# Fix: "Wrong wire type in tag / Truncated message / Field number 0 is illegal / Unexpected end-group tag"

## What was happening
`send_majorlogin()` (and `send_getlogin()`) took the raw HTTP body and fed it
**directly** into `CguTeam_pb2.MajorLoginRes().ParseFromString(...)`.

That only works for a *successful* login. Garena's login servers answer a
**successful** MajorLogin with a protobuf, but they answer **every error** with a
plain, non-protobuf body:

| Situation | HTTP | Body |
|---|---|---|
| Wrong/expired guest token | 400 | `BR_GOP_TOKEN_AUTH_FAILED` (text) |
| Bad open_id | 400 | `BR_PLATFORM_INVALID_OPENID` (text) |
| Bad/empty payload | 400 | `BR_PLATFORM_INVALID_PLATFORM` (text) |
| Wrong `ReleaseVersion` | 503 | `<html>...503 Service Unavailable...</html>` |
| gzip body not auto-decoded | 200 | `1f 8b 08 ...` |

When those bytes are parsed as protobuf you get exactly the errors in the log,
and the error message changes every time because it depends on the first byte of
the text:

* `BR_...`  -> `0x42` = field 8, wire-type 2, length 82 -> **Truncated message**
* `<html>`  -> `0x3c` = wire-type 4 (end-group) -> **Unexpected end-group tag**
* gzip `1f` -> wire-type 7 -> **Wrong wire type in tag**
* bytes < 0x08 -> field 0 -> **Field number 0 is illegal**

So the bot was not "broken" by the accounts per se — it was **crashing while
trying to decode the server's error message**.

## What the fix does
New helpers in `main-fix.py`:

* `_gunzip_if_needed()` – transparently gunzip/zlib-decompress the body first.
* `_extract_text_error()` – pull out the real text error (`BR_...`, HTML, ...).
* `_parse_proto_with_prefix()` – parse a protobuf that has the 64-byte signature
  prefix, trying offset 0, offset 64 and a bounded scan, and **never raises**.
* `_friendly_login_error()` – map known `BR_*` codes to a human explanation.

`send_majorlogin()` now:
1. reports the true HTTP status + server message instead of a protobuf crash,
2. auto-decompresses the body,
3. parses the protobuf with prefix tolerance,
4. **falls back** to `loginbp.ggblueshark.com` / `loginbp.ggpolarbear.com` if the
   version-config URL returns something unusable.

`send_getlogin()` uses the same helpers and now derives the `Host` header from
the URL instead of hard-coding `clientbp.ppmainecoonghj.com`.

## Before vs after
Before:
```
[-] [MajorLogin] Exception: Wrong wire type in tag.
[-] [LOGIN] MajorLogin rejected for UID 6678947720
```
After:
```
[!] [MajorLogin] https://loginbp.ppmainecoonghj.com -> HTTP 400: BR_GOP_TOKEN_AUTH_FAILED
[-] [MajorLogin] Login rejected — BR_GOP_TOKEN_AUTH_FAILED — Guest login token rejected —
    the UID/password is wrong, expired or the guest was deleted.
```

## Important
If the new log now shows **`BR_GOP_TOKEN_AUTH_FAILED`**, the code is working
correctly and the real problem is the **account itself** (the guest UID/password
is wrong, expired, or the guest was deleted). Regenerate the guest account or
use a fresh token. If it shows a **fallback recovery** line, the login now
succeeds through the alternate endpoint.
