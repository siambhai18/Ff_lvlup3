# TODO — Fix MajorLogin "Wrong wire type in tag" protobuf errors

## Diagnosis
- [x] Extract & inspect project (main-fix.py, CguTeam_pb2.py, token_cache.json)
- [x] Locate error source: `send_majorlogin()` final `ParseFromString`
- [x] Reproduce live MajorLogin with a valid cached account (works, 200 + protobuf @ offset 64)
- [x] Confirm server returns NON-protobuf bodies for errors (text `BR_*`, HTML 503)
- [x] Map each parse error to a response type (gzip / text / html / binary)
- [x] Verify alternate login endpoints also work

## Fix
- [x] Add robust helpers: gzip/deflate auto-decompress, text-error extraction, prefix-tolerant proto parse
- [x] Rewrite `send_majorlogin()` to report real server errors instead of crashing
- [x] Rewrite `send_getlogin()` the same way (+ derive Host from base_url)
- [x] Add endpoint fallback (loginbp.ggblueshark / ggpolarbear) when primary returns garbage
- [x] Syntax-check the patched file

## Deliver
- [x] Re-package the fixed project and explain the root cause + fix
