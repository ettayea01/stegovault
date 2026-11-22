# StegoVault

Hide a secret text message inside an ordinary **image** or **any file** — invisible,
optionally sealed with a password, and recoverable only with StegoVault.
Everything runs locally in the browser: no server, no uploads, works fully offline.

**Live:** https://ettayea01.github.io/stegovault/

**UI:** Moroccan Darija (Arabic script, RTL) and intentionally basic — one file
button per card; the app auto-detects images (LSB mode) vs other files (append mode).

## The two modes

### 1 · Image (PNG, LSB)
Pick any image (PNG, JPG, WebP…). The message is written into the
least-significant bits of the red, green and blue channels — changes no eye can
see — and the result is downloaded as a **PNG** (output is always PNG, because
JPEG compression would scramble those bits and destroy the message).

Capacity is about 3 bits per pixel: an 800 × 600 image hides ≈ 180,000 bytes
(≈ 179,991 plain characters, ≈ 179,947 with a password). The app shows the live
capacity of your image before you download, and refuses with a clear error if
the message does not fit.

### 2 · PDF / Any file (append)
Pick **any** file — PDF, JPG, ZIP, MP3, anything. The message is attached in a
small marked block at the very end of the file, after its normal end. Readers
and players ignore trailing bytes, so the carrier still opens and works
normally. The download keeps the original name plus a `-stego` suffix
(e.g. `report-stego.pdf`).

## How to use

**Hide:** drop in any carrier file (auto-detected) → type your message →
optionally set a password → **Hide & Download**.

**Reveal:** drop in the StegoVault file → enter the password if one was set →
**Reveal message** → read or copy it.

## Honest limits — read this before sending

- **Send the result as a Document / File, never as a photo.** WhatsApp,
  Facebook and most chat apps recompress photos and strip hidden data. A
  StegoVault image sent “as a photo” loses its message forever; sent as a
  document/file attachment, the bytes arrive untouched and the message survives.
- The append mode survives only while the carrier's bytes stay untouched:
  re-saving, converting or recompressing the file removes the hidden block.
- Image mode is not encryption by itself — anyone who knows (or guesses) the
  trick can read an unprotected message. **Use a password** for real secrecy.

## Tech

- Single self-contained `index.html` — zero external resources, works offline.
- Payload framing: `STG1` magic + flags + length, shared by both modes.
- Encryption (optional): **AES-256-GCM**, key derived with **PBKDF2-SHA-256**
  (100,000 iterations), fresh random 16-byte salt + 12-byte IV per message —
  the same scheme as EchoCipher. A wrong password fails cleanly instead of
  producing garbage.
- Core logic is exposed as `window.Stego` (framing, LSB embed/extract, append
  extract, encrypt/decrypt) and is covered by Node roundtrip tests: LSB
  (incl. Unicode/Arabic), capacity guard, append on PDF/JPEG-like bytes,
  not-found errors, and the AES password path.
