# About

A tool to convert ncm file to wav.

网易云音乐的ncm文件转换工具，输出统一为 wav。

The inner audio (mp3/flac/wav/aac/ogg/aiff/...) is decoded and transcoded to wav.
If the inner codec cannot be decoded, the file is reported as failed (non-zero exit code).

注意：wav 输出不携带歌曲元数据（标题/艺术家/封面等），文件名保留原歌曲名。

# Usage

```bash
ncm_c -h

ncm_c *.ncm
# for unix-pro guys
find . -type f -name '*.ncm' -exec ncm_c {} +
# for `fd` user
fd -e ncm -X ncm_c

# decrtpt and gather
mkdir failed
fd -e ncm -X ncm_c --quiet 2> >(xargs -d'\n' mv -t failed) | xargs -d'\n' mv -t .
```

# Installation

```shell
cargo install ncm_c
```
or download the binary from [release page](https://github.com/kingwingfly/ncmc/releases)

# Others

The core file parser is published as a crate: [ncmc_lib](https://crates.io/crates/ncmc_lib).

# Acknowledgement

- [YTSakura233/ncm2mp3](https://github.com/YTSakura233/ncm2mp3)
- [taurusxin/ncmdump](https://github.com/taurusxin/ncmdump)
