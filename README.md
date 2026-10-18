# Snapchat App Teardown (v14.28.0.54, Android)

Static reverse-engineering of the Snapchat Android app (`com.snapchat.android`, v14.28.0.54).
Verified genuine (Snap Inc signing cert `2F1CAAFC…59D95A58`). No app installed/run; no protobuf
payloads, crypto internals, or ranking math extracted — this is an architecture, backend, gRPC
service, and engagement-mechanics map from public artifacts.

## Contents
- **[snapchat-14.28-teardown.md](snapchat-14.28-teardown.md)** — full teardown: architecture, multi-cloud
  backend, 97-service/231-method gRPC catalog, Fidelius E2E encryption, Janus auth, Hermod/CAID
  attestation, Spotlight/Valis ranking engines, Snap Map location stack, Minerva GenAI, and the
  streak/charm/snap-score engagement mechanics.

## Method
Signature verification · apktool manifest/resource decode · dex string/URL/gRPC extraction ·
REA inventory · JADX decompile. Honest limits in the report.
