# Lester's Last Call

**Competition:** Seize da Flaq  
**Category:** Digital Forensics / Cryptography  
**Handle:** Effectivepawn  
**Status:** Solved

The challenge came with a WAV file named `lesters_last_call.wav`. The description said Lester had left a voicemail before disappearing, and hinted that the code was hidden in the tones, the picture, and the fine print.

I treated those as three separate leads: inspect the file metadata, extract the DTMF sequence, then use the spectrogram to recover the values needed by the decoder.

## 1. Start with the file

I checked the file type and metadata first:

```bash
file lesters_last_call.wav
exiftool lesters_last_call.wav
```

Most of the metadata looked ordinary. The `Comment` field was the useful part. It contained the decoding instructions:

- Each group of three digits represents a number from `000` to `255`.
- The first phone call is a decoy; use only the last call.
- The license plate and starting heat value are hidden in the static.
- The numbers must be decoded with a custom shuffled lookup table.

That gave me the order of work. I needed the last DTMF sequence, plus the plate and heat value from the spectrogram.

## 2. Extract the DTMF sequence

I converted the WAV to mono, 22.05 kHz, signed 16-bit PCM and piped it into `multimon-ng` for DTMF detection:

```bash
ffmpeg -hide_banner -loglevel error \
  -i lesters_last_call.wav \
  -ac 1 -ar 22050 -f s16le - |
  multimon-ng -t raw -a DTMF - 2>&1 |
  tee /tmp/lester_dtmf.log
```

The detector reported two calls:

```text
Call 1: 105 digits
Call 2: 201 digits
```

The note in the WAV comment made the choice clear: the first call was not Lester's. I grouped its digits into three-digit values anyway as a quick sanity check. It decoded to:

```text
sdf{wr0ng_numb3r_l4m4r_h4ng_up_n0w}
```

That was the decoy. The second call contained 201 digits, which gave me 67 three-digit values for the actual decoder.

## 3. Recover the plate and heat value

The metadata said these two values were hidden in the static, so I generated a spectrogram with SoX:

```bash
sox lesters_last_call.wav -n spectrogram \
  -o images/full.png -x 5000 -y 1600
```

The text was difficult to read at first, so I made two versions with different contrast settings:

```bash
sox lesters_last_call.wav -n spectrogram \
  -o images/contrast.png -x 5000 -y 1600 -z 80

sox lesters_last_call.wav -n spectrogram \
  -o images/alternate.png -x 5000 -y 2000 -z 120
```

Inspecting the spectrograms gave me the two values used by the decoder:

```text
PLATE = 46EEK572
HEAT  = 3
```

The plate mattered because it determines the shuffle seed. With the wrong plate, the lookup table would be different and the output would not decode correctly.

## 4. Rebuild the lookup table

The plate is converted into a 32-bit seed by processing each character's ASCII value:

```python
S = 0

for c in plate:
    S = (S * 31 + ord(c)) % (2**32)
```

The challenge then shuffles all byte values from `0` to `255`. It uses a linear congruential generator (LCG) to choose the swap position at each step:

```python
T = list(range(256))

for k in range(255, 0, -1):
    S = (1103515245 * S + 12345) % (2**31)
    j = S % (k + 1)
    T[k], T[j] = T[j], T[k]
```

For decoding, I needed the inverse of that shuffled table. In other words, for each value in `T`, I stored the position where it appeared:

```python
inverse = [0] * 256

for position, value in enumerate(T):
    inverse[value] = position
```

That inverse lookup turns each transformed byte back into the character's ASCII value.

## 5. Decode the second call

The decoder keeps two changing values:

- `H` starts at the recovered heat value, `3`.
- `B` starts at `0`.

For every three-digit number `N`, it first reverses the heat and bounty offsets:

```python
V = (N - 7 * H - B) % 256
```

Then `inverse[V]` gives the position in the shuffled table. That position is the ASCII value of the next character.

Both state values change after every decoded character, so the order of the numbers matters:

```python
B = (B + 2 * P) % 256
H = ((H + P) % 5) + 1
```

I put the metadata values, DTMF groups, and decoding steps into one script:

```python
plate = "46EEK572"
heat = 3
bounty = 0

numbers = [
    int(x) for x in """
    248 041 031 108 145 028 166 081 080 250
    207 131 046 182 213 166 123 041 033 207
    161 001 136 076 074 188 150 174 038 247
    078 252 131 162 061 175 228 027 162 098
    204 177 016 040 026 131 217 241 230 147
    185 015 176 093 149 147 143 120 163 054
    209 098 013 211 191 131 097
    """.split()
]

# Derive the shuffle seed from the license plate.
S = 0
for c in plate:
    S = (S * 31 + ord(c)) % (2**32)

# Recreate the challenge's shuffled byte table.
T = list(range(256))
for k in range(255, 0, -1):
    S = (1103515245 * S + 12345) % (2**31)
    j = S % (k + 1)
    T[k], T[j] = T[j], T[k]

# Build the reverse lookup: byte value -> position in T.
inverse = [0] * 256
for position, value in enumerate(T):
    inverse[value] = position

# Decode the second call, updating the state after each character.
result = bytearray()

for N in numbers:
    V = (N - 7 * heat - bounty) % 256
    P = inverse[V]
    result.append(P)

    bounty = (bounty + 2 * P) % 256
    heat = ((heat + P) % 5) + 1

print(result.decode("ascii"))
```

The script produced the flag below. The supplied 67 values decode to 67 ASCII characters.

## 6. Flag

```text
sdf{l3st3r_st1ll_pr3f3rs_th3_ph0n3_tr1ck_wh3n_h34t_h1ts_f1v3_st4rs}
```

The decoded message reads:

> Lester still prefers the phone trick when heat hits five stars.

## Takeaway

The audio had more than one thing going on. The DTMF detector gave me the numbers, the spectrogram supplied the plate and starting heat value, and the WAV comment explained how those pieces fit together. None of those clues was enough on its own; the message only came together once I combined all three.
