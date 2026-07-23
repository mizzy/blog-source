---
title: "Pyxisの開発: Transformerブロック"
date: 2026-07-23 20:55:03 +0900
---

[前回の記事](/blog/2026/07/22/1/)でFFN（SwiGLU）を実装した。これでTransformerを構成する主要な要素（RMSNorm、アテンション、FFN）がすべて揃ったので、今回はそれらを組み合わせて1つのTransformerブロックを組み立て、さらにブロックを積み重ねてTransformer全体を作る。対応するPRは[#21](https://github.com/mizzy/pyxis/pull/21)。

---

## Transformerブロックの構造

1つのTransformerブロックは、RMSNormとアテンションとFFNを組み合わせたもの。Qwen3を含むLLaMA以降のLLMでは、以下の構造になっている。

```
入力 x
  ↓
  RMSNorm (input_layernorm)
  ↓
  アテンション
  ↓
  + 残差接続 (入力 x を足す)
  ↓
  RMSNorm (post_attention_layernorm)
  ↓
  FFN
  ↓
  + 残差接続 (直前の値を足す)
  ↓
出力
```

Qwen3-1.7Bならこのブロックが28層積み重なっている。層の入出力はどちらも`hidden_dim = 2048`次元のベクトル列で、各層でその2048次元のベクトルが少しずつ変換されていく。

---

## 残差接続

このブロックで重要なのが残差接続（residual connection、skip connectionとも呼ばれる）。図中で「+ 残差接続」と書いた部分で、サブレイヤー（アテンションやFFN）の出力に、そのサブレイヤーの入力をそのまま足している。

```
出力 = 入力 + サブレイヤー(入力)
```

サブレイヤーが何を計算しても、その結果に元の入力を足すことで、入力の情報が層をまたいで直接下流に伝わるようにする、というのが残差接続の役割。

もともとは画像認識のResNet（[He et al., 2015](https://arxiv.org/abs/1512.03385)）で提案されたテクニックで、深いネットワーク（何十層、何百層と重ねる）を安定して学習できるようにするための工夫。深い層になるほど勾配が消失（値が小さくなりすぎて学習信号が伝わらなくなる）する問題があったが、残差接続で入力を直接下流に流すルートを作ることで、勾配も直接伝わるようになり、深いネットワークが学習可能になった。

Transformerは各ブロックにアテンションとFFNの2つのサブレイヤーがあり、それぞれに残差接続が付いている。Qwen3-1.7Bなら28ブロック × 2 = 56個の残差接続がある。

---

## Pre-Norm

もうひとつの重要な設計選択がPre-Norm。「RMSNormをサブレイヤーの前に適用する」という配置のこと。

先ほどの図を見ると、RMSNormがアテンションやFFNの**前**に来ている。これがPre-Norm。

対比としてPost-Normという配置もあって、こちらはサブレイヤーの**後**（残差接続の後）にNormを適用する。

```
Pre-Norm:  x + サブレイヤー(Norm(x))
Post-Norm: Norm(x + サブレイヤー(x))
```

オリジナルの"Attention Is All You Need"はPost-Normだったが、その後の研究で「Post-Normは学習が不安定になりやすく、深いモデルでは特にPre-Normの方が安定する」ことが分かってきて、現代のLLMはほぼPre-Normを採用している（[On Layer Normalization in the Transformer Architecture, Xiong et al., 2020](https://arxiv.org/abs/2002.04745)）。

Pre-Normでは、残差接続のルートに何も演算を挟まない（`x`がそのまま下流に流れる）ため、深く積み重ねても勾配が素直に伝わる、というのが安定性の理由。

---

## 実装

```rust
pub struct TransformerBlock {
    input_norm: RmsNorm,
    attention: Attention,
    post_attn_norm: RmsNorm,
    ffn: Ffn,
}
```

4つの構成要素をそのまま持つだけの構造体。

`forward()`の中身はこう。

```rust
pub fn forward(&self, x: &mut [f32], seq_len: usize) {
    let hidden_dim = x.len() / seq_len;

    // アテンションブロック
    let residual = x.to_vec();
    for position in x.chunks_exact_mut(hidden_dim) {
        self.input_norm.forward(position);
    }
    let attention_output = self.attention.forward(x, seq_len);
    for ((value, residual), attention) in x.iter_mut().zip(residual).zip(attention_output) {
        *value = residual + attention;
    }

    // FFNブロック
    let residual = x.to_vec();
    for position in x.chunks_exact_mut(hidden_dim) {
        self.post_attn_norm.forward(position);
    }
    for (position_idx, position) in x.chunks_exact_mut(hidden_dim).enumerate() {
        let ffn_output = self.ffn.forward(position);
        let start = position_idx * hidden_dim;
        for ((value, residual), ffn) in position
            .iter_mut()
            .zip(&residual[start..start + hidden_dim])
            .zip(ffn_output)
        {
            *value = residual + ffn;
        }
    }
}
```

前半がアテンションブロック、後半がFFNブロック。それぞれ:

1. 残差接続用に入力`x`のコピーを`residual`として取っておく
2. `x`にRMSNormを適用（各トークン位置ごとに独立に）
3. サブレイヤー（アテンションまたはFFN）を適用
4. サブレイヤーの出力に`residual`を足す

RMSNormとFFNは1つのトークン位置のベクトルに対して独立に適用できるので、`chunks_exact_mut(hidden_dim)`でトークンごとに区切って処理している。アテンションはトークン間の相互作用を扱うので、全トークン分の`x`と`seq_len`をまとめて渡す。

---

## Transformerの組み立て

複数のブロックを積み重ねて、最後にRMSNormをかけたものがTransformer全体。

```rust
pub struct Transformer {
    blocks: Vec<TransformerBlock>,
    final_norm: RmsNorm,
    hidden_dim: usize,
}

impl Transformer {
    pub fn forward(&self, x: &mut [f32], seq_len: usize) {
        for block in &self.blocks {
            block.forward(x, seq_len);
        }
        for position in x.chunks_exact_mut(self.hidden_dim) {
            self.final_norm.forward(position);
        }
    }
}
```

各ブロックを順に適用してから、最後にもう一度RMSNormをかける。この最後のRMSNormは`model.norm.weight`という名前でsafetensorsに格納されていて、Transformer全体の出力を正規化する役割。

Qwen3-1.7Bなら28個のブロックが積み重なっていて、embeddingで取り出したベクトル列がこの28層を順に流れて変換され、最後のRMSNormで整えられて出力される、という流れ。

---

これでTransformer本体ができた。次回はトークナイザーを組み込んで、テキストからトークンIDへの変換を実装する。
