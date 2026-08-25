# Secure Boot & Encrypted OTA at Scale

Module 3-09 built a bootloader that verifies a firmware image with a CRC —
enough to catch corruption, nothing to catch tampering. This module adds
what a CRC can never provide: cryptographic proof that an image was
produced by the legitimate vendor, plus the fleet-scale problem of updating
thousands of devices without a single compromised or bricked unit taking
down the rollout.

## Why CRC isn't enough, precisely

A CRC is designed to detect *accidental* corruption — bit flips from a noisy
flash write, a truncated transfer. It is not designed to resist an
*adversary*: anyone can recompute a CRC over a maliciously modified image
and write the new value into the header, exactly as module 3-09's own
`validate_image` would accept. Authenticity requires something the attacker
cannot compute without a secret the device already trusts.

## Signature verification, conceptually

```c
/* header extended with a signature instead of (or in addition to) a CRC */
typedef struct {
    uint32_t magic;
    uint32_t size;
    uint8_t  signature[64];      /* e.g. Ed25519 or ECDSA signature over the image hash */
    uint32_t version;
} signed_image_header_t;

/* the bootloader ships with the VENDOR'S PUBLIC key baked in at build time —
   never a private key, never anything writable at runtime */
extern const uint8_t vendor_public_key[32];

int verify_signed_image(const uint8_t *image, uint32_t total_size) {
    signed_image_header_t hdr;
    memcpy(&hdr, image, sizeof(hdr));
    if (hdr.magic != IMAGE_MAGIC) return -1;
    if (hdr.size != total_size) return -2;

    uint8_t hash[32];
    sha256(image + sizeof(hdr), total_size - sizeof(hdr), hash);

    /* only someone holding the VENDOR'S PRIVATE key could have produced a
       signature that verifies against the public key baked into this
       bootloader — this is the property a CRC can never provide */
    if (!ed25519_verify(hdr.signature, hash, sizeof(hash), vendor_public_key)) {
        return -3;
    }
    return 0;
}
```

The bootloader only ever holds the **public** key — verification, not
signing, happens on-device. Signing happens once, offline, in the vendor's
build/release pipeline, on infrastructure that never touches the field.

## The trust chain has to be anchored somewhere immutable

Signature verification only means something if the code performing the
verification, and the public key it checks against, cannot themselves be
tampered with. This is why **secure boot** on production silicon anchors
the first stage in **one-time-programmable (OTP) fuses or ROM** — a
bootloader that itself lives in ordinary, rewritable flash and merely
*checks* signatures provides no real guarantee, because an attacker with
write access to that flash can simply replace the checking code along with
the image. Real secure-boot silicon (STM32's TrustZone/secure-boot fuses,
similar mechanisms on other vendors) burns a hash of the trusted public key
into fuses that cannot be rewritten, so the first stage of verification is
rooted in something no software-only attack can alter.

## Encrypted OTA: confidentiality is a separate property from authenticity

Signing proves an image came from the vendor and wasn't altered; it says
nothing about whether an attacker sniffing the update *transport* can read
the firmware contents (a real concern for a product whose firmware contains
IP worth protecting, or credentials). Encrypting the OTA payload — typically
AES-256 with a key provisioned per-device or per-fleet-batch during
manufacturing — adds confidentiality on top of, not instead of, signature
verification: decrypt first, then verify the signature of the decrypted
image, so a corrupted or malicious ciphertext is caught the same way either
route in.

## Fleet-scale rollout: canary batches and staged rollback

Signing a bad image is still possible — a build with a real regression can
be legitimately signed by the vendor and still brick devices. At fleet
scale, the mitigation is procedural, not cryptographic:

```
1. Deploy to a small canary batch (1-5% of fleet), watch health telemetry
2. Automatic halt if canary error rate exceeds a threshold
3. Staged rollout: 10% -> 50% -> 100%, each stage gated on telemetry
4. Every device keeps the previous known-good image (module 3-09's dual-bank
   scheme) so a bad rollout is a rollback, not a truck roll
```

The dual-bank rollback mechanism from module 3-09 is what makes staged
rollout survivable — without a rollback path, "halt the rollout" still
leaves every device that already updated running the bad image.

## Modeling the verify-then-trust decision in portable C

The logic that decides "should this device install this image" (not the
real cryptography) is testable with a fake verifier:

```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>

typedef struct { int (*verify)(const uint8_t *img, uint32_t len); } verifier_t;

/* returns 1 if the update should be installed */
int should_install(verifier_t *v, const uint8_t *img, uint32_t len, int rollback_ok) {
    if (v->verify(img, len) != 0) return 0;     /* signature check failed: never install */
    (void)rollback_ok;                           /* real logic would also check version
                                                     monotonicity to prevent downgrade attacks */
    return 1;
}

int fake_verify_pass(const uint8_t *img, uint32_t len) { (void)img; (void)len; return 0; }
int fake_verify_fail(const uint8_t *img, uint32_t len) { (void)img; (void)len; return -3; }

int main(void) {
    verifier_t good = { .verify = fake_verify_pass };
    verifier_t bad  = { .verify = fake_verify_fail };
    uint8_t dummy[8] = {0};

    assert(should_install(&good, dummy, sizeof(dummy), 1) == 1);
    assert(should_install(&bad, dummy, sizeof(dummy), 1) == 0);

    printf("verify-then-trust decision model OK\n");
    return 0;
}
```

## Traps in secure boot and OTA at scale

- **Trusting a public key stored in ordinary flash**: if the key itself is
  writable by anything that can write firmware, an attacker replaces the key
  and their own signature, and verification "passes" against a
  self-signed malicious image.
- **No downgrade protection**: an attacker who can install *any* validly
  signed image, including an old one with a known vulnerability, can
  effectively undo a security fix — version-number monotonicity checks in
  the bootloader are part of a complete secure-boot design, not an optional
  extra.
- **Encrypting without also signing**: confidentiality alone doesn't stop a
  device from installing a decrypted-but-tampered image if nothing checks
  authenticity after decryption.
- **No staged rollout**: pushing 100% of a fleet at once turns any bad
  build — signed or not — into a fleet-wide outage instead of a caught
  canary.

## Cheat sheet

| Concept | Detail |
|---|---|
| CRC | Detects accidental corruption only — trivially recomputable by an attacker |
| Signature (Ed25519/ECDSA) | Proves the image came from whoever holds the private key |
| Public key placement | Must be immutable (OTP fuses/ROM) — a rewritable key defeats the whole scheme |
| Secure boot root of trust | First-stage verification anchored in hardware, not just software checking software |
| Encryption | Confidentiality, separate from and additional to signature-based authenticity |
| Downgrade protection | Version monotonicity check prevents re-installing an old, vulnerable, validly signed image |
| Canary/staged rollout | Procedural mitigation for a legitimately signed but regressive build |
| Verification here | Verify-then-trust decision logic compiled/run with `gcc`; real crypto primitives reviewed, not implemented/executed here |

## Exercise

Extend `should_install` to enforce downgrade protection: add a
`current_version`/`candidate_version` comparison alongside the (faked)
signature check, and reject an update whose version is not strictly greater
than the currently installed version even if the signature verifies. Write
assertions covering: valid signature + higher version (install), valid
signature + equal/lower version (reject), invalid signature regardless of
version (reject). Compile and run with `gcc`, and in a comment explain a
legitimate scenario where a vendor might need to bypass downgrade
protection on purpose (e.g. an emergency recall of a bad "higher" version)
and what extra safeguard that bypass path itself would need.
