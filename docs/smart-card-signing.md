# Signing with smart cards and tokens

> **Last reviewed:** 2026-11-02 · **Last updated:** 2026-11-02 · **Change:** SafeNet eToken module; the module that sees a card is preferred; expired and revoked card certificates are refused before the PIN; the PIN prompt masks ICP-Brasil CPF numbers

PdfCraft signs with keys on smart cards and USB tokens (national eID cards, corporate badges,
qualified-signature tokens) the way Acrobat, LibreOffice and Firefox do:

1. The card vendor's middleware provides a **PKCS #11 module**, a shared library with a public,
   standard interface (OASIS PKCS #11). PdfCraft loads it through `cryptoki` (Apache-2.0).
2. Certificates are listed without a PIN. Signing logs in with the PIN, typed once and zeroized,
   or none on a PIN-pad reader. **The private key never leaves the card.**
3. The signature is written as PAdES (ETSI EN 319 142), the same as for software keys.

Nothing from vendors' closed software is decompiled or reimplemented, and the card's PIN
lockout works as with any other application.

## Where

- **Desktop:** Sign with a Digital ID ▸ Configure New Digital ID ▸ *Use a smart card or token*.
  The module field is prefilled with the first known middleware module that sees a signing
  certificate, else the first installed (`pkcs11::preferred_module`).
  When a card holds an authentication and a signing certificate, only the signing one
  (keyUsage nonRepudiation) is offered.
- **CLI / agent tools:** `sign_card_ids` lists the cards' certificates; `sign_document` takes the
  `pkcs11:` reference. The module comes only from `pdfcraft-cli --card-module PATH` (else the first
  known one installed that sees a card) and the PIN only from a terminal prompt in
  `pdfcraft-cli run`; the MCP server can sign only with PIN-pad readers. No tool argument carries
  a PIN or a module path.
- **Before the PIN:** a card certificate outside its validity dates is refused; with `--network`
  its revocation (and that of the CAs above it) is asked too, and a revoked one is refused. Tokens
  often keep a replaced, revoked certificate next to its successor. `sign_card_ids` reports each
  certificate's status. The PIN prompt masks an ICP-Brasil CPF in the certificate name
  (`NAME:***.456.789-**`).
- **Check a card without the app:** `cargo run -p pdfcraft-sign --example pkcs11_probe -- [--sign] [--tsa URL]`.

## Long-term validation (opt-in)

Many jurisdictions require signatures that stay verifiable after the certificate expires
(PAdES B-LT / B-LTA). Preferences ▸ Signatures takes a **timestamp server** URL and an
**Include revocation information (LTV)** switch, both off by default. With them, signing goes
online on a background thread: an RFC 3161 signature timestamp, the chain's missing issuers
(AIA), an OCSP response or else a CRL for every certificate (only answers that verify and cover
the signing time are kept), a `/DSS`, and with a timestamp server a closing document timestamp:
**B-LTA**. Missing revocation information is reported, not fatal; a revoked certificate stops the
signing. The CLI does the same with `timestamp_server` / `ltv`, only when started with `--network`.

`pdfcraft-sign` never opens a socket (`ltv::Fetch` is injected); HTTP lives in
`pdfcraft-platform::http`.

## Known modules (Windows, 64-bit)

| Middleware | Module |
|---|---|
| Smart Card Middleware (NXP JCOP 3 "iGOV" eID cards; observed) | `C:\Program Files\Smart Card Middleware\bin\idoPKCS.dll` |
| Athena IDProtect | `C:\Windows\System32\asepkcs.dll` |
| Thales/Gemalto IDPrime (SafeNet Authentication Client 10.8 R2+) | `C:\Program Files\SafeNet\Authentication\SAC\x64\IDPrimePKCS1164.dll` |
| SafeNet eToken (SafeNet Authentication Client; ICP-Brasil A3 tokens; observed) | `C:\Windows\System32\eTPKCS11.dll` |
| OpenSC | `C:\Program Files\OpenSC Project\OpenSC\pkcs11\opensc-pkcs11.dll` |

Windows also copies card certificates into the user's certificate store, but under the legacy
CryptoAPI provider, which the Windows-store backend (CNG only) skips; PKCS #11 reads the card in
the reader directly.

## Not yet

PIN-pad and multi-slot UI; P-521, Brainpool and Ed25519 keys on tokens; named timestamp servers
with credentials; trusting a national root hierarchy other than by hand (Signatures panel ▸
Add to trusted certificates).
