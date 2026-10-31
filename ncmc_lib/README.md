# About

The lib for the tool to convert ncm file to wav.

网易云音乐的ncm文件转换工具`ncm_c`的库。

# Usage

```rust no_run
use ncmc_lib::NcmFile;

let ncm = NcmFile::open("path/to/your.ncm").unwrap();
// save as the native format (mp3/flac/...) with id3 tags
ncm.save().unwrap();
```

```rust no_run
use ncmc_lib::NcmFile;

let ncm = NcmFile::open("path/to/your.ncm").unwrap();
// decode and transcode the inner audio to wav (no tags), requires the `transcode` feature
ncm.save_as_wav().unwrap();
```

# features

- `cover_download`: provide `with_cover` method to download cover image from internet if not contained in ncm file.
- `transcode` (default): provide `save_as_wav`/`save_as_wav_to` to decode the inner audio (mp3/flac/wav/aac/ogg/aiff/...) and transcode it to wav.

# Acknowledgement

- [YTSakura233/ncm2mp3](https://github.com/YTSakura233/ncm2mp3)
- [taurusxin/ncmdump](https://github.com/taurusxin/ncmdump)
