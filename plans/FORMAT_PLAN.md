# ncmc 转换 NCM → WAV（唯一输出格式）

## Context

当前 ncmc 只解密 NCM 并按文件内部原生格式（`meta.format`）保存，不做转码。**需求（已简化）：不管 NCM 内部是什么格式，一律转换为 WAV 输出。**

**用户已确认的设计决定：**
1. 输出格式只有 wav，无 CLI 格式参数、无原生/目标格式匹配确认流程
2. 解码端需支持原生格式 **wav / mp3 / flac / aac / ogg / aiff**；无法转换（不支持的编码等，如 ogg-opus）→ **报错**（走现有失败路径：stderr + 非零退出码）
3. WAV 输出**跳过元数据**（hound 不支持写元数据，WAV 对标签/封面支持差）
4. 测试样本：仓库根目录 `威尼斯船歌.ncm`（mp3 原生）；`test/` 下 2 个 git 跟踪的 ncm 文件

## 技术选型（API 已通过 docs.rs 验证）

| 用途 | crate | 说明 |
|---|---|---|
| 解码音频 → PCM | `symphonia 0.5`，默认 feature 全开（全部编解码器）+ 显式 `features = ["aiff"]` | 纯 Rust，按内容探测格式，无 C 依赖 |
| 编码 WAV | `hound 3.6` | 纯 Rust |

统一中间表示：交错 `Vec<i32>` PCM + sample_rate / channels / bits_per_sample。不再需要 mp3lame-encoder / flacenc / cmake。

编解码支持说明（symphonia 0.5.5 feature 已核实，默认 feature 即全部编码器）：
- 重点格式：mp3、flac、wav、aac（ADTS + `isomp4` 附带 m4a）、ogg（vorbis）、aiff
- 顺便支持：symphonia 默认 feature 自带的其他编码/容器 —— alac（m4a）、adpcm、pcm 变体等，零额外成本
- 仍无解码器的（如 ogg-opus）→ 走报错路径（符合"无法转换则报错"）
- 解码输出为 F32 的源（如 aac）→ 写 32-bit float WAV（hound `SampleFormat::Float`）；整型源按 `spec.bits_per_sample().max(16)` 写 Int WAV

## 转换规则

| 原生格式 | 处理 |
|---|---|
| mp3 / flac / aac / ogg / aiff / wav 及 symphonia 可解码的其他格式（alac、adpcm 等） | symphonia 解码 → hound 写 wav（统一路径，按内容探测、不特判） |
| 无解码器（如 ogg-opus）/ 探测或解码失败 | `NcmError::Transcode` → 现有失败语义（stderr 报告 + 退出码非 0） |

## 修改的文件

### 1. `ncmc_lib/Cargo.toml`
- 新增 optional 依赖：`symphonia`、`hound`
- 新增 feature：`transcode = ["dep:symphonia", "dep:hound"]`，`default = ["transcode"]`
- dev-dependencies: `tempfile`
- 版本 0.3.1 → 0.4.0（新增公共 API）

### 2. `ncmc_lib/src/transcode.rs`（新文件，核心转换逻辑）
```rust
pub(crate) struct Pcm {
    sample_rate: u32,
    channels: usize,
    bits_per_sample: u32,   // 来自 symphonia spec；mp3=16，flac 常为 16/24，wav 16/24
    samples: Vec<i32>,      // 交错
}
pub(crate) fn decode<R: Read>(reader: R) -> Result<Pcm>
// 全量读入 Cursor（NcmFile 只 impl Read，symphonia 需要 Read+Seek）
// → MediaSourceStream → probe（按内容识别，不依赖 meta.format）
// → 取第一个可解码音频轨 → 解码循环 → SampleBuffer<i32>::copy_interleaved_ref
// 不支持的 codec → NcmError::Transcode；流末尾的 DecodeError/UnexpectedEof 视为正常结束
pub(crate) fn pcm_to_wav<W: Write>(pcm: &Pcm, w: W) -> Result<()>
// hound WavWriter，SampleFormat::Int，bits_per_sample 取 pcm.bits
```

### 3. `ncmc_lib/src/lib.rs`
- `mod transcode;`（`#[cfg(feature = "transcode")]`）
- `NcmFile` 新方法：
  ```rust
  /// 解密并解码为 wav，保存在原文件旁（无元数据）
  pub fn save_as_wav(self) -> Result<PathBuf>       // path.with_extension("wav")
  pub fn save_as_wav_to(self, path: impl AsRef<Path>) -> Result<PathBuf>
  ```
  实现：`save_without_meta_to` 之后不写 ID3，改为 decode + pcm_to_wav（自 `&mut self` 读解密流）
- 现有 `save/save_to/save_without_meta*`（原生格式输出）保持不变，库用户向后兼容

### 4. `ncmc_lib/src/error.rs`
- 新变体：`#[error("Transcode error: {0}")] Transcode(String)`

### 5. `ncm_c/src/main.rs`
- 仅一处行为改动：`f.save()` → `f.save_as_wav()`
- 更新 clap `about` / 使用说明文案（输出统一为 wav）
- 线程池、`--quiet`/`--no-internet`/`-j` 逻辑全部不动

### 6. 文档与版本
- `README.md`（根）与 `ncmc_lib/README.md`：说明一律输出 wav、wav 无内嵌元数据、原生格式支持 mp3/flac/wav/aac/ogg/aiff（及 symphonia 支持的其他格式）
- `CHANGELOG.md` 新条目
- `ncm_c` 0.2.10 → 0.3.0（默认输出从原生格式改为 wav，破坏性变更）；两处 Cargo.toml description 同步

## 实现顺序（TDD）

1. `transcode.rs` 单元测试先行（合成正弦波 PCM）：pcm→wav→hound 读回，采样率/声道/样本一致
2. 实现 `pcm_to_wav`、`decode` 使测试通过
3. `save_as_wav*` + error 变体；集成测试 `ncmc_lib/tests/wav.rs`：对 `威尼斯船歌.ncm` 与 `test/*.ncm` 各跑 `save_as_wav`，用 hound 校验输出（可读、时长 > 0、双声道/44100 合理）；aac/ogg/aiff 无现成 ncm 样本，解码走与 mp3/flac 完全相同的 `decode` 路径，由 symphonia 支持矩阵保证（不支持的编码按需求报错）
4. main.rs 改调用 + 文案；cargo fmt / clippy -D warnings
5. 文档、CHANGELOG、版本号

## 验证

```bash
cargo fmt && cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
# 端到端：
cargo run -p ncm_c -- 威尼斯船歌.ncm    # mp3 原生 → 威尼斯船歌.wav
cargo run -p ncm_c -- test/*.ncm        # 另两个样本 → .wav
file *.wav                              # RIFF/WAVE 头 + PCM 数据
afplay 威尼斯船歌.wav                     # 试听确认解码正确
# 失败路径：损坏/非音频 ncm → stderr 报错、退出码非 0（现有语义）
```
时长可与 `~/Downloads/门德尔松：威尼斯船歌（Op.30 No.6）.wav` 参考对照。
