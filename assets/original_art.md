# Original asset statement

All files under `assets/png` and `assets/wav` were generated specifically for
**You Have Got Pizza** by `assets/generate_assets.py`.

The visual design is intentionally generic and symbolic:

- a fictional academic Piazza background;
- a professor sprite with a small four-frame walking animation;
- hungry student sprites with four-frame walking/reaching animations;
- dough, tomato sauce, cheese, and basil ingredient icons;
- small UI icons, platform tiles, ladder tiles, and a plate;
- a splash screen using only the project title and original pizza imagery.

The audio design is also original:

- short mono PCM WAV sound effects synthesized from simple sine/harmonic tones;
- no melodies, samples, recordings, or game sounds from third-party sources;
- no imitation of BurgerTime music, branding, character designs, or sound effects.

The game is mechanically inspired by the broad arcade idea of moving on platforms
while assembling food, but the artwork, names, characters, sounds, setting, and
story are original and Piazza-themed.

Regenerate the full asset set with:

```sh
cd assets
python3 generate_assets.py
```

The generated PNG files are master assets for documentation and future bitmap
conversion. The current PRG32 cartridge renders lightweight rectangle sprites and
uses PRG32 beeps so that the C and RISC-V assembly sources remain readable in a
classroom setting.
