---
title: "Pyxisの開発: テンソルデータの読み込み"
date: 2026-06-30 22:01:50 +0900
---

[前回の記事](/blog/2026/06/30/1/)でsafetensorsのヘッダーをパースしてテンソルのメタデータ（名前、dtype、shape、オフセット）を取り出すところまで書いた。今回はその次のステップで、実際のテンソルデータをバイト列からf32の数値配列として読み込む部分を実装した。対応するPRは[#16](https://github.com/mizzy/pyxis/pull/16)。

---

## SafeTensors構造体の導入

前回は`parse_header`関数でヘッダーを読んでメタデータを返すだけだったので、ファイルをすぐ閉じてよかった。今回はテンソルデータの本体も読みに行くので、ファイルへのアクセスを保持し続ける必要がある。そこで`SafeTensors`構造体を導入して、ファイルアクセスとメタデータをまとめて持つようにした。

```rust
pub struct SafeTensors {
    mmap: memmap2::Mmap,
    data_offset: usize,
    tensors: HashMap<String, TensorInfo>,
}
```

`mmap`がファイルの中身へのアクセス。`data_offset`はヘッダーの終わり（= テンソルデータの先頭）の位置で、8バイトのサイズフィールド + ヘッダーのJSONバイト数。`tensors`が前回の`parse_header`で取得していたテンソルのメタデータ（dtype、shape、オフセット）。

前回の`parse_header`のロジックは`SafeTensors::load()`メソッドに移動していて、ヘッダーをパースしてメタデータを取り出す処理自体は同じ。

ファイルアクセスには`memmap2`クレートを使ってメモリマップしている。

```rust
let file = File::open(path)?;
let mmap = unsafe { memmap2::Mmap::map(&file)? };
```

メモリマップはファイルの中身をプロセスの仮想アドレス空間にマッピングする仕組みで、OSのページキャッシュを介してアクセスされる。ファイル全体を一気にメモリに読まなくても、アクセスしたページだけが物理メモリに載る。モデルファイルは数GBになるので、全部メモリに読み込むのは現実的ではないが、メモリマップなら必要な部分だけが透過的に読み込まれる。

---

## Dtype enum

前回は`dtype`を`String`で持っていたけど、テンソルデータを読むにはdtypeごとにバイト列の解釈を変える必要がある。文字列のまま`match`するのは型安全でないので、`Dtype` enumに変えた。

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Deserialize)]
#[serde(rename_all = "UPPERCASE")]
pub enum Dtype {
    F32,
    F16,
    BF16,
}

impl Dtype {
    pub fn element_size(self) -> usize {
        match self {
            Dtype::F32 => 4,
            Dtype::F16 | Dtype::BF16 => 2,
        }
    }
}
```

`element_size()`は1要素あたりのバイト数を返す。F32は4バイト、F16とBF16は2バイト。テンソルデータを読み込むときに、バイト列をこのサイズごとに区切って数値に変換する。バイト長がこのサイズの倍数になっているかのバリデーションにも使う。

safetensorsのフォーマット仕様ではF32/F16/BF16以外にもI64、U8、BOOLなど[22種類のdtype](https://github.com/huggingface/safetensors/blob/main/safetensors/src/tensor.rs)が定義されている。Pyxisではさしあたり必要なF32/F16/BF16だけをサポートして、それ以外のdtypeのテンソルはスキップするようにした。他のdtypeは必要になったら対応する。

---

## テンソルデータの読み込み

`tensor_f32()`メソッドがこのPRの本体。テンソル名を指定すると、そのテンソルのバイト列をdtypeに応じてf32に変換して返す。

```rust
pub fn tensor_f32(&self, name: &str) -> io::Result<Vec<f32>> {
    let info = self.tensors.get(name).ok_or_else(|| {
        io::Error::new(io::ErrorKind::NotFound, format!("tensor not found: {name}"))
    })?;

    let [start, end] = info.data_offsets;
    let start = self.data_offset + start;
    let end = self.data_offset + end;
    let bytes = &self.mmap[start..end];

    let values = match info.dtype {
        Dtype::F32 => bytes
            .chunks_exact(4)
            .map(|c| f32::from_le_bytes([c[0], c[1], c[2], c[3]]))
            .collect(),
        Dtype::F16 => bytes
            .chunks_exact(2)
            .map(|c| half::f16::from_le_bytes([c[0], c[1]]).to_f32())
            .collect(),
        Dtype::BF16 => bytes
            .chunks_exact(2)
            .map(|c| half::bf16::from_le_bytes([c[0], c[1]]).to_f32())
            .collect(),
    };

    Ok(values)
}
```

やっていることはシンプルで、ヘッダーに記録されている`data_offsets`にデータ領域の開始位置を足して、mmap上の該当バイト列をスライスする。あとはdtypeに応じてバイトの解釈を変えるだけ。

- **F32**: `chunks_exact(4)`で4バイトずつに切り、`f32::from_le_bytes`でリトルエンディアンのf32に変換する
- **F16**: `chunks_exact(2)`で2バイトずつに切り、`half::f16::from_le_bytes`でF16に変換してから`.to_f32()`でf32にする
- **BF16**: 同様に2バイトずつに切り、`half::bf16::from_le_bytes`でBF16に変換してから`.to_f32()`でf32にする

`chunks_exact`がバイト列を1要素あたりのバイト数（前述の`element_size()`、F32なら4、F16/BF16なら2）ごとに分割し、`from_le_bytes`がそのバイト列をリトルエンディアンの数値として解釈する、という二段階の処理。F16とBF16の変換には`half`クレートを使っている。

この時点ではすべてのテンソルをf32に変換して読み込んでいる。Rustの標準ライブラリにはBF16の型がなく、F16の型（`f16`）もnightly限定の実験的APIでstableでは使えない。`half`クレートがBF16/F16の型と算術演算を提供しているが、[演算の実装](https://github.com/starkat99/half-rs/blob/main/src/bfloat.rs)はf32に変換してから計算してBF16/F16に戻す、というもの。最初からf32で持てばRustのプリミティブ型の演算がそのまま使えて、変換の往復が不要になる。メモリ効率の面ではBF16のまま持って計算時にf32に変換する方が有利だけど、まずは動くものを作るのが優先なので、ここはシンプルな方を選んだ（後のPRでBF16のまま持つ最適化を入れている）。

---

これで、safetensorsファイルからテンソルの数値データをf32として取り出せるようになった。次回はこのデータを使って、トークンIDからベクトルを引くembeddingテーブルルックアップの実装について書く。
