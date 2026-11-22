# StegoVault

Hide a secret text message inside an ordinary **image** or **any file** — invisible,
optionally sealed with a password, and recoverable only with StegoVault.
Everything runs locally in the browser: no server, no uploads, works fully offline.

**Live:** https://ettayea01.github.io/stegovault/

**UI:** Moroccan Darija (Arabic script, RTL) and intentionally basic — one file
button per card; the app auto-detects images (LSB mode) vs other files (append mode).

## The three modes

### 1 · Image, sent as a file (PNG, LSB)
Pick any image (PNG, JPG, WebP…). The message is written into the
least-significant bits of the red, green and blue channels — changes no eye can
see — and the result is downloaded as a **PNG** (output is always PNG, because
JPEG compression would scramble those bits and destroy the message).

Capacity is about 3 bits per pixel: an 800 × 600 image hides ≈ 180,000 bytes
(≈ 179,991 plain characters, ≈ 179,947 with a password). The app shows the live
capacity of your image before you download, and refuses with a clear error if
the message does not fit.

### 2 · Image, sent as a photo (robust DCT/QIM) 🛡️
Pick an image and choose **«سيفطها كصورة»**. The message is written with
quantization-index modulation (QIM) into one mid-frequency DCT coefficient of
every 8×8 luminance block, each bit repeated 9× and recovered by majority
vote, with a CRC16 so a damaged message fails cleanly instead of coming out
garbled. The change is tiny (max 4 grey levels per pixel, PSNR 48–62 dB) and
**survives WhatsApp / Facebook Messenger photo send**: the platforms
recompress the photo to JPEG, and the QIM decision margin (Δ/4 = 8) is larger
than the JPEG quantization error at quality 60 and up. Before embedding, the
app shrinks the carrier to 1600px on its long side — WhatsApp's own maximum —
so the platform has nothing left to shrink.

Capacity is small by design (~1 bit per 8×8 block): a 1600×1600 photo hides
≈ 544 bytes, 1280×800 ≈ 211 bytes, 800×600 ≈ 93 bytes. Short secrets only —
a code, a link, a sentence. The app shows the live capacity and refuses with
a clear error if the message does not fit.

### 3 · PDF / Any file (append)
Pick **any** file — PDF, JPG, ZIP, MP3, anything. The message is attached in a
small marked block at the very end of the file, after its normal end. Readers
and players ignore trailing bytes, so the carrier still opens and works
normally. The download keeps the original name plus a `-stego` suffix
(e.g. `report-stego.pdf`).

Reveal is automatic: an image is tried as LSB first, then robust DCT, then
append — the receiver just drops the file in.

## How to use

**Hide:** drop in any carrier file (auto-detected) → type your message →
optionally set a password → **Hide & Download**.

**Reveal:** drop in the StegoVault file → enter the password if one was set →
**Reveal message** → read or copy it.

## Honest limits — read this before sending

- **«سيفطها كملف» (LSB): send the result as a Document / File, never as a
  photo.** WhatsApp, Facebook and most chat apps recompress photos and strip
  hidden data. A StegoVault image sent “as a photo” loses its message forever;
  sent as a document/file attachment, the bytes arrive untouched and the
  message survives.
- **«سيفطها كصورة» (robust DCT): built exactly for photo send** — it survives
  JPEG recompression down to quality 60 and being re-sent as a photo twice
  (verified in tests). It does **not** survive downscaling, cropping, a
  screenshot, or heavy filters: the app pre-shrinks the carrier to 1600px so
  WhatsApp/Messenger won't shrink it further, but if the platform still
  resizes it (very large original, "HD off" quirks), the message is lost.
  Keep the message short and use a photo of 800px or more on each side.
- The append mode survives only while the carrier's bytes stay untouched:
  re-saving, converting or recompressing the file removes the hidden block.
- Image mode is not encryption by itself — anyone who knows (or guesses) the
  trick can read an unprotected message. **Use a password** for real secrecy.

## Tests

- Old paths: framing, LSB (incl. Arabic/emoji), capacity guard, append on
  PDF-like bytes, not-found errors, AES password path — **10/10 Node tests**.
- Robust DCT: real photo (1600×1600) + synthetic photo (1280×800) + mostly
  flat image (800×600) + small image (400×300), short Arabic + long + tiny
  messages, JPEG q70/q75/q85 after embedding — **all recovered** where the
  message fit the capacity; random (encrypted-like) payload also recovered
  after **double JPEG recompression (q75→q72) and at q60**. Downscaling the
  image to 75% destroys the message (expected — the 8×8 grid moves), which is
  why the app pre-shrinks the carrier to 1600px.
- Harness (outside the published repo, so it never ships):
  `~/workspace/stegovault-tests/` — `regression.js` (old paths),
  `test_all.py` + `test_extra.py` (JPEG simulations via PIL).
  Run: `node regression.js` and `python3 test_all.py` from that folder.

## Tech

- Single self-contained `index.html` — zero external resources, works offline.
- Payload framing: `STG1` magic + flags + length, shared by all modes.
- Encryption (optional): **AES-256-GCM**, key derived with **PBKDF2-SHA-256**
  (100,000 iterations), fresh random 16-byte salt + 12-byte IV per message —
  the same scheme as EchoCipher. A wrong password fails cleanly instead of
  producing garbage.
- Core logic is exposed as `window.Stego` (framing, LSB embed/extract, robust
  DCT embed/extract, append extract, encrypt/decrypt) and is covered by Node
  roundtrip tests: LSB (incl. Unicode/Arabic), capacity guard, append on
  PDF/JPEG-like bytes, not-found errors, the AES password path, and the
  robust DCT channel through simulated WhatsApp/Facebook JPEG recompression.
