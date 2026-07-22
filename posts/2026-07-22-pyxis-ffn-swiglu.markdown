---
title: "Pyxisの開発: FFN (SwiGLU)"
date: 2026-07-22 21:46:47 +0900
---

[前回の記事](/blog/2026/07/21/1/)でアテンションを実装した。今回はトランスフォーマー層のもうひとつの構成要素、FFN（Feed-Forward Network）を実装した。対応するPRは[#20](https://github.com/mizzy/pyxis/pull/20)。

---

## FFNとは

トランスフォーマーの各層は、アテンションとFFNの2つのサブレイヤーからなる。アテンションが「トークン間で情報を混ぜる」役割だったのに対して、FFNは「各トークンのベクトル自体を非線形変換する」役割を担う。

計算の性質にも違いがある。アテンションは`QKᵀ`のようにトークン間の相互作用を扱うのでトークン列全体を見る必要がある。一方FFNは各トークンのベクトルを独立に処理する（トークンAのFFNの計算にトークンBは関与しない）。位置ごとに同じ計算をするだけなので、実装もシンプル。

FFN自体は"[Attention Is All You Need](https://arxiv.org/abs/1706.03762)"のオリジナルのトランスフォーマーから存在していて、そこでは2層の線形変換の間にReLUを挟むだけの構造だった。

```
FFN(x) = max(0, x × W1 + b1) × W2 + b2
```

やっていることは:

1. 入力`x`（`hidden_dim`次元）に重み行列`W1`を掛けてバイアス`b1`を足し、`hidden_dim`より大きい中間層の次元に変換する
2. その結果にReLU（`max(0, x)`）を適用する
3. さらに重み行列`W2`を掛けてバイアス`b2`を足し、元の`hidden_dim`次元に戻す

元論文では`hidden_dim = 512`、中間層の次元 = 2048で、中間層で4倍に次元を広げてから元に戻す構造。「中間で次元を広げてから戻す」というのが、表現力を確保するための工夫。

ReLU（Rectified Linear Unit）は`max(0, x)`という単純な活性化関数で、負の値は0にして、正の値はそのまま通す。

活性化関数というのは、線形変換（行列の掛け算）の出力に対してかける非線形な関数のこと。ニューラルネットワークでは、線形変換だけを重ねても表現力が上がらない（何層重ねても1つの線形変換と等価になってしまう）ので、間に非線形な関数を挟む必要がある。ReLUはその代表例で、実装が単純で計算も速いため広く使われてきた。

現代のLLMでは、この単純な構造は改良されている。Qwen3を含むLLaMA以降のLLMではSwiGLUという構造が使われている。

---

## SwiGLU

SwiGLUは2020年にNoam Shazeerが["GLU Variants Improve Transformer"](https://arxiv.org/abs/2002.05202)で提案した。GLU（Gated Linear Unit）というゲート機構と、Swish（SiLUとも呼ばれる）という活性化関数を組み合わせたもの。

計算はこうなる。

```
SwiGLU(x) = down_proj(SiLU(gate_proj(x)) * up_proj(x))
```

3つの重み行列（`gate_proj`、`up_proj`、`down_proj`）が使われる。

1. `x`（hidden_dim次元）に`gate_proj`と`up_proj`をそれぞれ掛けて、2つの中間ベクトル（intermediate_size次元）を作る
2. `gate_proj(x)`にSiLUを適用する
3. その結果と`up_proj(x)`を要素ごとに掛ける（ここがゲート機構）
4. 結果に`down_proj`を掛けて`hidden_dim`次元に戻す

Qwen3-1.7Bでは`hidden_dim = 2048`、`intermediate_size = 6144`。つまりFFNの中で一度2048次元から6144次元に「広げて」計算を行い、また2048次元に「戻す」という構造。中間で次元を広げることで表現力を高める、というのは元のFFNから引き継がれている考え方。

SiLUは`x * sigmoid(x)`という活性化関数で、値の範囲を非線形に潰す役割。ReLUと違って負の値も少し通す滑らかな関数で、[2017年にGoogleが提案した](https://arxiv.org/abs/1710.05941)。SiLUとSwishは同じ関数で、別々のグループがほぼ同時期に提案して名前が2つ付いた（後にSwishはSiLUに統一される流れになった）。

「なぜSwiGLUが良いのか」については、論文自体が「we offer no explanation as to why these architectures seem to work; we attribute their success, as all else, to divine benevolence.（なぜこれらのアーキテクチャが機能するのか説明できない。他のあらゆることと同じく、神の恩寵によるものとしておく）」と書いているくらいで、明確な理論的根拠はなく、経験的に性能が良いことが分かっている、という状態。

---

## 実装

```rust
pub struct Ffn {
    gate_proj: Vec<f32>,
    up_proj: Vec<f32>,
    down_proj: Vec<f32>,
    hidden_dim: usize,
    intermediate_size: usize,
}
```

3つの重み行列と、それぞれの次元を持つだけのシンプルな構造。Qwen3では`model.layers.0.mlp.gate_proj.weight`のような名前でsafetensorsに格納されている。

```rust
pub fn forward(&self, x: &[f32]) -> Vec<f32> {
    assert_eq!(x.len(), self.hidden_dim);

    let mut gate = matmul(x, &self.gate_proj, self.intermediate_size, self.hidden_dim);
    let up = matmul(x, &self.up_proj, self.intermediate_size, self.hidden_dim);

    for (gate_value, up_value) in gate.iter_mut().zip(up) {
        *gate_value = silu(*gate_value) * up_value;
    }

    matmul(
        &gate,
        &self.down_proj,
        self.hidden_dim,
        self.intermediate_size,
    )
}
```

計算式そのままで、`gate_proj`と`up_proj`で中間ベクトルを2本作り、片方にSiLUをかけてもう片方と要素ごとの積を取り、`down_proj`で元の次元に戻す。

SiLUの実装はこれ。

```rust
fn silu(x: f32) -> f32 {
    x / (1.0 + (-x).exp())
}
```

`x * sigmoid(x)`と等価だが、`sigmoid(x) = 1 / (1 + exp(-x))`を展開して`x / (1 + exp(-x))`と書いている。sigmoid関数を経由せずに一つの式で書けるので簡潔。

---

これでトランスフォーマー層を構成する主要な要素（RMSNorm、アテンション、FFN）がすべて揃った。次回はこれらを組み合わせて実際のトランスフォーマー層を組み立てる。
