# meeting-assistant-transcription-models

Model binaries for **Meeting Assistant**, kept out of the app repository on purpose.

The app checkout stays small and private; a Whisper bundle is ~100 MB of data that
changes for different reasons than code does. So the app ships a *pointer* to this
repo and downloads what it needs, verifying every byte before it trusts any of it.

```
MeetingAssistant (local git only)          this repo (GitHub)
  assets/asr_models.json   ──────────────►   Release: asr-tiny-1
  asset_root = ".../releases/download/asr-tiny-1/"   tiny-encoder.int8.onnx
  lib/domain/transcription/model_installer.dart      tiny-decoder.int8.onnx
                                                     tiny-tokens.txt
```

## Why GitHub Releases and not git files / LFS / Pages

* `git` refuses any file over 100 MB without LFS, and the decoder is 90 MB — right
  at the edge where one upgrade to `base` (153 MB) breaks the repository.
* Free LFS storage is 1 GB and free LFS *transfer* is 1 GB/month, i.e. about ten
  app installs a month before GitHub starts refusing downloads.
  Release assets cost nothing extra.
* Releases are append-only: an old app build keeps working after a new model ships,
  which matters because a phone mid-meeting cannot be forced to upgrade.

## Layout

| Path | What it is |
|---|---|
| `manifest.json` | Human-readable catalog of every published bundle (mirrors the app's bundled copy) |
| *Releases* | The `.onnx` / `tokens.txt` assets themselves. Nothing else lives here. |

Release tags are `<model_key>-v<version>` with underscores replaced, e.g.
`asr-tiny-1` for `whisper-tiny` version 1. The tag **is** the API contract: the
app's `asset_root` names it, so a tag is never re-pointed or deleted once any
shipped app build refers to it. Updates are a new tag, always.

## Currently published

| Tag | Bundle | Size | Languages |
|---|---|---|---|
| [`asr-tiny-1`](../../releases/tag/asr-tiny-1) | sherpa-onnx Whisper **tiny**, int8 | 103,609,903 B (~99 MB) | en, hi |

## Integrity

Every file has a sha256 in `manifest.json`. The app hashes the download and refuses
the file if it disagrees. That is not paranoia: a truncated ONNX model loads without
complaining and transcribes fluent nonsense, and a user has no way to tell that apart
from a real transcript. The download is written to `<name>.part` and renamed only
after the hash matches, so a cut-off download can never look installed.

## Publishing a new bundle

```sh
# 1. Stage outside the app checkout. Names must match the manifest exactly.
mkdir -p ../whisper-base-v1 && cd ../whisper-base-v1
BASE=https://huggingface.co/csukuangfj/sherpa-onnx-whisper-base/resolve/main
for f in base-encoder.int8.onnx base-decoder.int8.onnx base-tokens.txt; do
  curl -L -C - -o "$f" "$BASE/$f"
done
sha256sum *.onnx *.txt          # these values go into the manifest verbatim

# 2. Create the release and upload (gh CLI), or use the web UI
gh release create asr-base-1 \
  base-encoder.int8.onnx base-decoder.int8.onnx base-tokens.txt \
  --title "Offline ASR: whisper-base v1" \
  --notes "int8 bundle. sha256 pinned in the app manifest."

# 3. Point the app at it: set asset_root in
#    MeetingAssist_AdvAPI/assets/asr_models.json  and in the
#    transcription_models table, then set active = true.
```

Only one `active` model per `model_key` may exist; the app takes the newest version
and installs it under `<app support>/asr/<model_key>/v<version>/`.

## Provenance

Exported Whisper models from [`csukuangfj` (k2-fsa / sherpa-onnx)](https://huggingface.co/csukuangfj),
int8-quantised variants. OpenAI Whisper is MIT-licensed; redistribution here is for
the same purpose it was published for — running it. Nothing in this repo is trained
or owned by us, and no audio, text or user data is ever stored in it.
