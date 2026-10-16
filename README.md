# metacopy

**English** | [Русский](README.ru.md)

A small macOS CLI that copies metadata from an original photo onto its edited version.

Why. I run a lot of iPhone photos through neural nets: upscaling, retouching, sometimes generating stuff on top. What comes out is almost always a PNG with zero tags. Drop that into Photos and it lands on today's date, no location, no camera, just wrong. I got tired of looking up the right exiftool flags and fixing orientation by hand every time, so I wrote this.

```
$ metacopy IMG_4821.HEIC upscaled.png
✓ IMG_4821.HEIC → upscaled.png
  Дата съёмки  2024-06-15 14:23:01 +03:00
  GPS          55.751244, 37.618423, высота 156 м
  Камера       Apple iPhone 15 Pro
  Объектив     iPhone 15 Pro back triple camera 6.765mm f/1.78
  MakerNotes   Apple, 41 тег
  Картинка     2048×1536 JPEG, Orientation=1, без превью
  Сохранено    /Users/me/Pictures/upscaled_meta.jpg
```

The CLI output is in Russian for now. The lines above are: date taken, GPS, camera, lens, MakerNotes, image info and where the result was saved.

## Install

You need exiftool:

```sh
brew install exiftool
```

Any Python 3.8+ works, including the one that ships with macOS (I ran it on 3.9 and 3.13). Nothing to pip install.

```sh
git clone https://github.com/macfluryy/metacopy.git
cd metacopy
ln -s "$PWD/metacopy" ~/.local/bin/metacopy
```

If `~/.local/bin` isn't on your PATH:

```sh
mkdir -p ~/.local/bin
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

`sudo ln -s "$PWD/metacopy" /usr/local/bin/` works too, whatever you prefer. Just don't delete the cloned folder afterwards, the symlink points into it.

Check that it works: `metacopy --version`

## Usage

```
metacopy <original> <edited> [-o where] [--format auto|jpg|heic|keep]
```

By default the result goes next to the edited image: `art.png` → `art_meta.jpg`. Your files are never touched, all the work happens on a temp copy.

```sh
metacopy IMG_1234.HEIC art.png                    # → art_meta.jpg next to it
metacopy IMG_1234.HEIC art.png -o ~/Desktop/      # into a folder
metacopy IMG_1234.HEIC art.png -o sunset.jpg      # with your own name
metacopy IMG_1234.HEIC art.png --format heic      # → art_meta.heic
metacopy IMG_1234.HEIC art.webp --format keep     # webp stays webp
```

Tip: type `metacopy ` and drag both files from Finder into the terminal, the paths get filled in for you.

Then just drop the result into Photos (or File → Import) and it shows up on the original's date and on the map.

### Flags

| flag | what it does |
|---|---|
| `-o PATH` | file or folder. Treated as a folder if it ends with `/` or already exists as one |
| `--format` | `auto` (default): PNG, WEBP and everything else become JPEG, JPEG and HEIC are left alone. `jpg` / `heic`: force that format. `keep`: no conversion at all |
| `--quality N` | quality when converting, 95 by default |
| `--copy-icc` | also copy the original's color profile, see below |
| `-f` | overwrite the result if it already exists. Without it the tool refuses |
| `--batch CSV` | process a bunch of pairs |
| `-v` | print every exiftool and sips command, for debugging |

### Batch

Make a CSV where each row is an `original,edited` pair. An optional third column sets where to save:

```csv
source,target,output
IMG_1234.HEIC,ai/IMG_1234_upscaled.png
IMG_1235.HEIC,ai/IMG_1235_upscaled.png
IMG_1240.HEIC,ai/portrait.webp,best/portrait.jpg
```

```sh
metacopy --batch pairs.csv -o done/
```

Worth knowing:

- relative paths are resolved from the folder the CSV is in, not from where you run the command
- the header row and lines starting with `#` are skipped
- `;` works as a delimiter too, that's what Excel saves with a Russian locale. Quote paths that contain commas
- if one row fails, the rest still go through. At the end you get `Готово: 4 из 5, ошибок: 1` (done: 4 of 5, errors: 1) and exit code 1

## How it works

It's basically an exiftool wrapper with the right flags, plus fixing whatever starts lying after the copy.

1. Copies the target into a temp dir. PNG/WEBP get converted to JPEG via `sips`, because Photos reads EXIF from PNG hit or miss. Transparency goes onto a white background
2. `exiftool -all= --ICC_Profile:all` wipes all old metadata except the color profile
3. `exiftool -TagsFromFile src -all:all -unsafe` copies everything from the original: date with timezone, GPS, camera, lens, Apple MakerNotes
4. Then the fixes:
   - `Orientation=1`. The iPhone stores pixels the way the sensor sees them and writes `Rotate 90 CW`, but the AI output is already rotated. Without this every portrait shot ends up sideways
   - `ExifImageWidth/Height` and the XMP sizes are set to the real image size
   - IFD1 and PreviewImage are removed, otherwise some apps show a thumbnail of the old photo
   - `ColorSpace` is set based on the profile that's actually in the file
5. `FileModifyDate` is set to the capture date, taking `OffsetTimeOriginal` into account, so nothing shifts on import
6. Moves the result into place, re-reads it and prints a report. If something didn't make it over you'll see `✗ не перенеслось` (not copied)

No `_original` backups are left behind: exiftool runs with `-overwrite_original`, but only on the temp copy.

Without `sips` (so, not on a Mac) conversion falls back to Pillow if it's installed. HEIC output needs `sips`.

### About the color profile

The original's ICC profile is **not** copied by default, on purpose. iPhones shoot in Display P3, and AI tools almost always output sRGB. Put a P3 profile on sRGB pixels and the colors go neon. So the image keeps its own profile.

`--copy-icc` is for the opposite case: the tool kept P3 pixels but dropped the profile, and the image looks washed out next to the original.

## Troubleshooting

**Image is sideways.** The report will say `! у целевого файла было Orientation=…` (the target had Orientation=…). It means the AI tool kept the rotation tag but didn't rotate the pixels. Rotate it in Preview and run again.

**`нет в исходнике` (missing in source) everywhere.** The original most likely lost its metadata on the way: Telegram and WhatsApp strip it. Get the original via AirDrop, or in Photos use File → Export → Export Unmodified Original.

**Unexpected error.** Normal errors (broken file, missing file, wrong format) are printed as a single line. If you hit something weird, run with `-v` and paste the output into an issue.

Exit codes: `0` ok, `1` error, `2` bad arguments, `130` interrupted with Ctrl+C.

## Known issues

- `--copy-icc` together with `--format heic` doesn't work if the image had no profile of its own: exiftool can't add ICC to that kind of HEIC. You'll get a warning and the colors stay as in the target
- tested on macOS 15 with exiftool 13.59. Haven't tried older versions, but nothing exotic is used
- if Photos on some macOS version still puts the image on the wrong date, open an issue with the output of `exiftool -a -G1 file`
