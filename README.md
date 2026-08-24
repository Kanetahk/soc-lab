# SOC Lab

Personal SOC Laboratory (Security Operations Center) Splunk, threat detection,
and response automation.

## Known limitations
This host (Intel i3-2328M, Sandy Bridge mobile) does not have AES-NI
enabled, Intel disabled this instruction set on some mobile i3 SKUs of
that generation. Splunk versions from 9.4/10.x onward require AES-NI/AVX
in the KVStore (MongoDB) binary, with no way to work around it via
environment variable. The image is pinned to `splunk/splunk:9.3`, the
last version line still compatible with this hardware.