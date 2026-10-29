# Galaxy A07 + Galaxy Buds 3 Pro: what really happens to Dolby Atmos (E-AC-3 JOC) and 24-bit FLAC over Bluetooth

Tested with Samsung Music ( Installed from Galaxy Store) , Termux and Shizuku. Phone: Samsung Galaxy A07 (\[fill in One UI version\], \[fill in Android version\]). Earbuds: Galaxy Buds 3 Pro.

Note: Only render, file read, and output render test. No audio listening test includes.&#32;

## TL;DR

1. **Atmos files are decoded properly.** Samsung Music hands E-AC-3 JOC (Dolby Atmos) files to the phone's own licensed Dolby decoder, `c2.dolby.eac3.decoder`. It outputs a 12-channel (7.1.4) stream at 48 kHz, and the decoder's downmix mode is off.
2. **Bluetooth carries stereo only.** The 12 channels are turned into a headphone mix by Dolby's effect. The Atmos switch visibly turns that effect and Android's spatializer on and off in the logs.
3. **24-bit FLAC keeps 24 bits up to the codec.** Samsung Music decodes it to **32-bit float**, the mixer thread that feeds the Bluetooth output is 24-bit, and the Bluetooth codec is Samsung SSC at 24-bit, 48 kHz. Turning Dolby on or off did not change that path.
4. **Not bit-perfect.** SSC is lossy. Its real **output** bitrate is **not measured** here ( that's another lot work 🙂, if I do test, I'll update) . What goes into the encoder is about 2,304 kbit/s.&#32;

Everything below is what the phone reported, not a measurement of the sound itself.

## The question

I have offline Dolby Atmos music files (E-AC-3 JOC in M4A containers) and 24-bit FLAC files, and I listen on Galaxy Buds 3 Pro. Does the phone really decode Atmos, what does the Atmos switch do, and is 24-bit kept all the way to the earbuds? I could not find a clear answer, so I read what the audio system reports while music plays. ( over curiosity 🤔, it's just me & my free time)

One fact first: no Dolby bitstream travels over Bluetooth. The phone decodes the file and sends ordinary stereo audio through a Bluetooth codec. The interesting part is what happens before that.

## Tools

- MediaInfo (Android): what is inside a file
- Codec Info (Android): which decoders the phone has
- Termux + Shizuku (using its `rish` shell): runs `logcat` and `dumpsys` with shell permissions, no root
- Samsung Music, Samsung Video Player, ZArchiver

## Setup: Termux with Shizuku's rish

1. Start Shizuku. In its menu open **Use Shizuku in terminal apps** and tap **Export files**. The system picker would not let me use the Download folder, so I made a folder called `rish` in internal storage. It gives two files: `rish` and `rish_shizuku.dex`.
2. In Termux:

```
termux-setup-storage
cp ~/storage/shared/rish/rish ~/storage/shared/rish/rish_shizuku.dex $PREFIX/bin/
cd $PREFIX/bin
sed -i 's/PKG/com.termux/' rish
chmod +x rish
chmod 400 rish_shizuku.dex
rish -c id
```

If the last command prints `uid=2000(shell)`, it works. Shizuku may ask Termux for permission the first time.

Note: in different tests I used different Dolby musics, so don't get confused if one says 5.1 multichannel and other 7.2 😁

## Test 1: is the file real Atmos? (MediaInfo)

The Atmos test file ( Example- 2 min 48 s, 17.6 MiB, `.m4a`):

- Format: **E-AC-3 JOC**, commercial name Dolby Digital Plus with Dolby Atmos
- Bit rate 768 kb/s, constant, 6-channel bed (L R C LFE Ls Rs), 48 kHz
- **15 dynamic objects**
- Lossy, as E-AC-3 always is

So the file genuinely contains Atmos objects.

## Test 2: which player opens it?

- **Samsung Video Player** would not open the audio-only .m4a. But if u have zarchiver, tap on the file, then tap the arrow and select open with and choose Samsung's built in video player.
- **Samsung Music** opened and played it.

The format name does not matter to the decoder. It picks the codec from the codec tag inside the container (`ec-3`). The extension only affects which app lists or accepts the file.

## Test 3: which decoders does the phone have? (Codec Info)

Searching for `eac3` found one decoder, `c2.dolby.eac3.decoder`, listed three times ( three individual formats ) . For all three: codec provider is Device vendor / OEM, software-only is false, hardware acceleration false (normal for audio, it runs on the CPU), up to 32 instances.

| Entry | Input channels | Bitrate range | Sample rates |
| --- | --- | --- | --- |
| 1 | 1 to 6 | 32 kbps to 640 kbps | 32, 44.1, 48 kHz |
| 2 | 1 to 8 | 32 kbps to 6 Mbps | 32, 44.1, 48 kHz |
| 3 | 1 to 16 | 32 kbps to 6 Mbps | 48 kHz only |

I forgot to screenshot the format label (MIME type) for each entry in the app. Entry 3 looks like the JOC variant, but that is a guess. The logs below confirm the JOC decoding anyway.

## Test 4: Atmos file in Samsung Music, Atmos switch ON

Script used (start it, then start the song, wait 10 seconds, press Enter):

```bash
#!/data/data/com.termux/files/usr/bin/bash
OUT=~/dolbywatch; mkdir -p "$OUT"
rish -c "logcat -c"
rish -c "logcat -v time" 2>&1 | grep -iE --line-buffered "eac3|ec-3|joc|atmos|dolby|spatializ|c2\." > "$OUT/log.txt" &
echo "Now play the song in Samsung Music. After 10 seconds, press Enter here."
read -r
rish -c "dumpsys media.audio_flinger" > "$OUT/flinger.txt" 2>&1
rish -c "dumpsys audio" > "$OUT/audio.txt" 2>&1
kill %1 2>/dev/null
echo "=== CODEC LOG ==="; head -30 "$OUT/log.txt"
echo "=== AUDIO TRACKS ==="; grep -iE "channel mask|format|spatializ|dolby|atmos" "$OUT/flinger.txt" | head -40
echo "=== SPATIALIZER ==="; grep -iE "spatializ|atmos|dolby" "$OUT/audio.txt" | head -30
```

What the log showed:

- The player recognised the stream as `audio/eac3-joc`.
- `c2.dolby.eac3.decoder` was allocated and configured with `audio/eac3-joc`. A statistics line labelled it `DOLBY_SW_EAC3`.
- Dolby engine settings pushed to the decoder: `dax-state = 1`, output level -14 dB with Dolby Atmos, `active-device = 128` (0x80, the A2DP Bluetooth device code).
- The decoder reported `nPCMOutChannels: 12` and `jocForceDownmixMode 0`, so no forced downmix.
- The player's output format was channel mask `0x2d63f` at 48 kHz. That mask is 12 channels: front L/R/C, LFE, back L/R, side L/R and four height channels (7.1.4).
- The Android audio policy checked that the stream can be spatialized on the A2DP device and selected the spatializer output.
- `dumpsys audio` said `mHasSpatializerEffect: true` and `isSpatializerEnabled: true`. Dolby's effect library (DAX3, `Dap_Spatializer`, `libswspatializer.so`) was present in the audio mixer.
- The final mixer output to the HAL was plain stereo, 16-bit, as expected for Bluetooth.
- One oddity: the framework still printed `isSpatialized=false` for the track. Samsung uses it's own Dolby path, so I do not know what that flag means here.&#32;

## Test 5: Atmos switch OFF (second Atmos capture)

Same script. This capture started after playback had begun, so it has no decoder lines, but it recorded the moment Atmos was switched off:

- The track still arrived as 12 channels, mask `0x2d63f`, 48 kHz.
- `g_effect_dolby_enable=0`, `setProfileEnabled() : for dolby atmos, false`, mode off.
- `AS.SpatializerHelper: setFeatureEnabled(false)`, and `isSpatializerEnabled:false` (it was `true` with Atmos on).
- The log confirmed Atmos is supported on the Bluetooth path, that the stream can be spatialized as multichannel 5.1 on the A2DP device, and that Android's own spatializer has no head tracking. (That is Android's spatializer, not the Buds' 360 Audio.)

So the switch really reaches the audio system. It changes the headphone processing, not the decoding.

## Test 6: listening

With the Buds 3 Pro I could not hear a clear difference between Atmos on and off, or between the Dolby decode and a plain stereo version. This was informal, one listener, no blind test. Plausible reasons: Atmos on headphones is a simulation of direction, pop mixes may use the height channels gently, and Bluetooth compression and small drivers hide detail. Bluetooth only carries two channels, so the Buds' four drivers (two per ear) all play one stereo signal. The 8d musics we find on YouTube aren't Dolby or Spatial audio ✌️ ( not peace sign, I mean the other one ), those are just artificially edited stereo two channels volume .

Galaxy Buds 360 Audio is Samsung's own spatial effect, not Dolby. The 360 audio is mostly working for VR and SAMSUNG'S built in camera app's (record 360 audio) feature.

## Test 7: 24-bit FLAC

Script (start it first, then start a FRESH track, wait 10 seconds, press Enter):

```bash
#!/data/data/com.termux/files/usr/bin/bash
OUT=~/flacwatch; mkdir -p "$OUT"
rish -c "logcat -c"
rish -c "logcat -v time" 2>&1 | grep -iE --line-buffered "flac|pcm-encoding|a2dp|codec.*(config|type)|bits" > "$OUT/log.txt" &
echo "Now start the 24-bit FLAC in Samsung Music. After 10 seconds, press Enter here."
read -r
rish -c "dumpsys media.audio_flinger" > "$OUT/flinger.txt" 2>&1
rish -c "dumpsys bluetooth_manager" > "$OUT/bt.txt" 2>&1
kill %1 2>/dev/null
echo "=== DECODER LOG ==="; head -25 "$OUT/log.txt"
echo "=== TRACK TABLE ==="; grep -A6 "Id Active Client" "$OUT/flinger.txt" | head -50
echo "=== OUTPUT FORMAT ==="; grep -iE "HAL format|Processing format|Mixer channel|Sample rate" "$OUT/flinger.txt" | head -20
echo "=== BLUETOOTH CODEC ==="; grep -iE "codec|bits_per_sample|sample_rate" "$OUT/bt.txt" | head -30
```

The filter was too broad (Bluetooth chatter filled the decoder section), so I ran two follow-up commands with the FLAC playing:

```
rish -c "dumpsys media.audio_flinger" > ~/f.txt
grep -nE "Output thread|Thread name|HAL format|Processing format| yes " ~/f.txt
grep -nE "^Output thread|Output device|AUDIO_DEVICE_OUT_BLUETOOTH" ~/f.txt
```

Results:

| Stage | What the dump showed |
| --- | --- |
| Player | Samsung Music (the active track's client) decoded the FLAC itself. The track is in format code `0x5`, which is 32-bit float, stereo |
| Track sample rate | 48,000 Hz in the first capture, 44,100 Hz in the later ones. I did not record the files' native rates |
| Output thread | `AudioOut_6D`, a MIXER thread. HAL format and processing format are both `0x6`, **24-bit packed** |
| Where that thread goes | `Output devices: 0x80 (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP)`. It is the only 24-bit thread |
| Other threads | 16-bit. `AudioOut_D` goes to a Bluetooth SCO headset device (call audio). The rest had no output device at the time |
| Bluetooth codec in use | **SSC, 48 kHz, 24 bits per sample, stereo** |
| Other codecs offered for these Buds | AAC (44.1 kHz, 16-bit), SBC (44.1 kHz, 16-bit) |
| Dolby on vs off | **Same result both times**: the FLAC track sat in `AudioOut_6D`, 24-bit |

Format codes in these dumps are Android audio formats: `0x1` = 16-bit, `0x5` = float, `0x6` = 24-bit packed.

Chain, as far as the logs show: FLAC file > decoded by Samsung Music > 32-bit float > `AudioOut_6D` (24-bit, A2DP output) > SSC at 24-bit, 48 kHz > Buds 3 Pro.

SSC offers only 48 kHz here, so 44.1 kHz material is presumably resampled. I did not verify that directly. Dolby on or off does not change the thread format, but the logs cannot show how precisely Dolby's effect processes audio inside that thread. For FLAC you want to hear as mastered, I leave Dolby off, because it is a spatial effect that changes the sound, not because it costs bits.

## Test 8: how much data goes to the Buds?

Script (reads Bluetooth counters before and after 30 seconds of playback):

```bash
#!/data/data/com.termux/files/usr/bin/bash
OUT=~/btrate; mkdir -p "$OUT"
snap(){ rish -c "dumpsys bluetooth_manager" 2>&1 | grep -iE "byte|packet|frame|bitrate|bit rate|kbps|throughput|enqueue|underrun|sent"; }
echo "Start the FLAC in Samsung Music now. When it is playing, press Enter."
read -r
snap > "$OUT/a.txt"; date +%s > "$OUT/ta"
echo "Measuring for 30 seconds. Keep it playing..."
sleep 30
snap > "$OUT/b.txt"; date +%s > "$OUT/tb"
echo "=== SECONDS BETWEEN SNAPSHOTS ==="; echo $(( $(cat $OUT/tb) - $(cat $OUT/ta) ))
echo "=== LINES THAT CHANGED ==="; diff "$OUT/a.txt" "$OUT/b.txt" | head -60
```

There is no byte counter in this dump, so the real over-the-air rate was not obtained. What I got:

- The dequeue counter rose by 1,756 in 32 seconds: about 55 packets a second, one every 18 ms.
- 18 ms at 48 kHz is 864 frames. In 24-bit stereo that is 5,184 bytes, the same read size that appeared in an earlier Bluetooth log line. The two numbers agree, but that is an inference.
- Input to the encoder: 48,000 x 2 x 24 = **2,304 kbit/s** (about 288 KB/s), calculated.
- The only SSC encoder line in the log: `a2dp_ssc_encode_frames: nb_frame 1 sample 48000 level 1 limit false monitor true scmst false`. It confirms 48 kHz. I do not know what `level`, `limit` or `monitor` mean, and it prints no bitrate.

**SSC output bitrate: not measured.** The route that could measure it is the Bluetooth HCI snoop log from Developer options, read in Wireshark. I did not do that. ( another long ahh work 😭)

## What I did not prove

- The sound itself. These logs show what the phone sends, not what comes out of the Buds and I observed like a sound specialist.&#32;
- That Samsung Music renders the Atmos objects binaurally with full quality. The 12-channel decode, the active Dolby effect and the enabled spatializer are all visible, but the quality of the final mix is not.
- The SSC bitrate, and anything about how lossy it is in practice.
- How precisely Dolby's effect processes audio internally.
- The native sample rates of the FLAC files (the dumps show 48 kHz in one capture and 44.1 kHz in others).
- Other players. I did not test Auxio, Just Player, VLC or Poweramp.
- Other phones. This is one Galaxy A07 and one pair of Buds 3 Pro. Other models or One UI versions may route things differently.

## How to repeat this

1. Check your file in MediaInfo: you want `E-AC-3 JOC` for Atmos, or 24-bit for FLAC.
2. Set up Termux with `rish` (above).
3. Run the script, but start it BEFORE the song and start a fresh track, or you miss the decoder lines.
4. Compare Atmos on and off in the phone's sound settings.
5. To find which thread feeds Bluetooth, look for the `Output thread` whose `Output devices` line says `AUDIO_DEVICE_OUT_BLUETOOTH_A2DP`.

Corrections and results from other phones are welcome in the comments.
