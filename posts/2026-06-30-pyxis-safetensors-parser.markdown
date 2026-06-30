---
title: "Pyxisの開発: safetensorsパーサー実装"
date: 2026-06-30 14:40:07 +0900
---

[前回の記事](/blog/2026/06/23/1/)でPyxisを軽く紹介した。LLM推論エンジンを自前実装するプロジェクトで、今回はその最初のステップであるsafetensorsパーサーの実装について書く。

---

## safetensorsとは

LLMの重みファイルのフォーマットはいくつかあるが、HuggingFaceが策定した[safetensors](https://github.com/huggingface/safetensors)が量子化されていないモデルの配布フォーマットとして現在のデファクトになっている。名前の通り、従来のPickleベースのフォーマット（PyTorchの`.pt`ファイル）に比べて安全で、任意コードの実行ができない。

バイナリフォーマットの構造はシンプルで、こうなっている。

```
[ 8 bytes : header size (u64 LE) ]
[ N bytes : JSON header          ]
[ tensor data                    ]
```

最初の8バイトにヘッダーのサイズがリトルエンディアンのu64で入っていて、その後ろにそのサイズ分のJSONが続く。JSONの中に各テンソルのメタデータ（名前、dtype、shape、データのオフセット）が入っている。テンソルの実データはJSONの後ろにそのまま並んでいる。

```json
{
  "__metadata__": {"format": "pt"},
  "model.embed_tokens.weight": {
    "dtype": "BF16",
    "shape": [151936, 2048],
    "data_offsets": [0, 622329856]
  }
}
```

`dtype`はテンソルの要素の数値型。主なものはこんな感じ。

| dtype | ビット数 | 説明 |
|---|---|---|
| `F32` | 32bit | 単精度浮動小数点。指数部8bit、仮数部23bit。精度が高いが重い |
| `F16` | 16bit | 半精度浮動小数点。指数部5bit、仮数部10bit |
| `BF16` | 16bit | Brain Float 16。Google Brainが考案（現在はGoogle DeepMindに統合）。指数部8bit、仮数部7bit。F32と指数部の幅が同じなので表現できる数値のスケールがF32と同じ（最大値約3.38×10^38、最小値約1.17×10^-38）。精度はF32より低いがスケールを保ったままメモリを半分に抑えられるため、LLMの学習・推論では広く使われる |
| `I8` | 8bit | 8ビット整数。量子化済みモデルで使われる |

BF16はF32の指数部をそのままに仮数部を23bitから7bitに切り詰めた、いわばF32の軽量版。メモリ使用量はF32の半分になりながら、数値のスケール（桁の範囲）はF32と同じに保てる。LLMの学習では勾配の値が幅広い範囲に散らばるため、スケールを保ったBF16はF32からの置き換えとして相性がよく、最近のモデルはBF16で配布されているものが多い。F16も同じ16bitだが指数部が5bitと狭く、スケールはBF16より大幅に小さい。

`shape`はテンソルの次元数と各次元のサイズで、たとえば`[151936, 2048]`なら151936行×2048列の行列を表す。

---

## 実装

パーサー本体はこんな感じ。

```rust
pub fn parse_header(path: &Path) -> io::Result<HashMap<String, TensorInfo>> {
    let mut file = File::open(path)?;

    let mut size_buf = [0u8; 8];
    file.read_exact(&mut size_buf)?;
    let header_size = u64::from_le_bytes(size_buf) as usize;

    let mut header_buf = vec![0u8; header_size];
    file.read_exact(&mut header_buf)?;

    let header_json: HashMap<String, serde_json::Value> =
        serde_json::from_slice(&header_buf)?;

    let mut tensors = HashMap::new();
    for (key, value) in header_json {
        if key == "__metadata__" {
            continue;
        }
        let info: TensorInfo = serde_json::from_value(value)?;
        tensors.insert(key, info);
    }

    Ok(tensors)
}
```

ファイルを開いて最初の8バイトを読んでヘッダーサイズを得る、ヘッダーサイズ分のバイトを読んでJSONとしてパースする、`__metadata__`キーをスキップして各テンソルのメタデータを取り出す、という流れ。

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct TensorInfo {
    pub dtype: String,
    pub shape: Vec<usize>,
    pub data_offsets: [usize; 2],
}
```

実際にQwen3-1.7BのモデルファイルにこのCLIを通してみると、こんな出力が得られる。

```
Tensor                                Dtype           Shape               Offsets
---------------------------------------------------------------------------------
lm_head.weight                         BF16  [151936, 2048]          0..622329856
model.embed_tokens.weight              BF16  [151936, 2048]          0..622329856
model.layers.0.input_layernorm.weight  BF16          [2048]  622329856..622333952
...
Total: 311 tensors
```

テンソルの名前、dtype、shape、データのオフセット範囲が一覧で確認できる。311個のテンソルが含まれていることがわかる。
