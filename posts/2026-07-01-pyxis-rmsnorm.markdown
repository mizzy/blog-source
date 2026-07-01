---
title: "Pyxisの開発: RMSNorm"
date: 2026-07-01 21:10:07 +0900
---

[前回の記事](/blog/2026/07/01/1/)でembeddingテーブルルックアップを実装して、トークンIDからベクトルを取り出せるようになった。今回はそのベクトルに対して最初に適用される正規化処理、RMSNormを実装した。対応するPRは[#18](https://github.com/mizzy/pyxis/pull/18)。

---

## RMSNormとは

RMSNormは2019年にBiao ZhangとRico Sennrichが[提案](https://arxiv.org/abs/1910.07467)した正規化手法。LLaMA以降のLLM（Mistral、Gemma、Qwen、DeepSeekなど）で広く採用されている。

正規化とは、ベクトルの各要素の値の大きさを一定の基準に揃える処理。学習時に値が大きくなりすぎたり小さくなりすぎたりするのを防ぐために導入された手法で、モデルは正規化が入った状態で訓練されている。推論時にも同じ正規化を適用しないと正しい結果が出ないので、Pyxisでも実装する必要がある。

RMSNormは2つのステップからなる。まず各要素をRMS（二乗平均平方根）で割る。RMSは各要素を二乗して平均を取り、その平方根を取った値。たとえば`[1.0, 3.0, -2.0, 4.0]`のRMSは、二乗して`[1.0, 9.0, 4.0, 16.0]`、平均が`(1.0 + 9.0 + 4.0 + 16.0) / 4 = 7.5`、平方根を取って`sqrt(7.5) ≈ 2.74`。各要素をこのRMSで割ると`[0.37, 1.10, -0.73, 1.46]`になる。要素間の比率は保ったまま、正規化後のベクトルのRMSが1.0になる。

次に、学習済みのweightを要素ごとに掛ける。このweightはsafetensorsファイルにテンソルとして保存されているパラメータで、各要素の重要度に応じた調整を行う。

もともとよく使われていたのはLayerNormで、計算式はこうなる。

```
y[i] = (x[i] - mean(x)) / sqrt(var(x) + ε) * weight[i] + bias[i]
```

まず平均を引いて（re-centering）、標準偏差で割って（re-scaling）正規化し、そこに学習済みのweightとbiasを適用する。biasはweightと同様の学習済みパラメータで、足し算でオフセットを加える。RMSNormはLayerNormを簡略化したもので、平均を引く処理とbiasを省いて、二乗平均平方根（Root Mean Square）で割るだけにしている。Zhang & Sennrichの論文では、平均を引く処理はなくても性能に影響しないことが実験で示されていて、計算量を7〜64%削減できるとしている。

---

## 計算

RMSNormの計算式はこうなる。

```
y[i] = x[i] / sqrt(mean(x²) + ε) * weight[i]
```

1. 入力ベクトル`x`の全要素を二乗して平均を取る（= 二乗平均）
2. それにε（小さい定数、ゼロ除算防止）を足してから平方根を取る（= RMS）
3. 各要素をRMSで割る
4. 学習済みの重み`weight`を要素ごとに掛ける

εはゼロ除算を防ぐための定数で、Qwen3では1e-6が使われている。前のセクションの具体例ではεを省略したが、実際の計算ではRMSに加算される。

---

## 実装

```rust
pub struct RmsNorm {
    weight: Vec<f32>,
    eps: f32,
}
```

`weight`が学習済みパラメータで、`eps`がε。

```rust
pub fn forward(&self, x: &mut [f32]) {
    assert_eq!(x.len(), self.weight.len());

    let mean_square = x.iter().map(|value| value * value).sum::<f32>() / x.len() as f32;
    let factor = 1.0 / (mean_square + self.eps).sqrt();

    for (value, weight) in x.iter_mut().zip(&self.weight) {
        *value *= factor * weight;
    }
}
```

---

Qwen3では、トランスフォーマーの各層の中でRMSNormが2回適用される。embeddingで取り出したベクトルが0番目の層に入ると、まずRMSNormで正規化され、アテンション（トークン間の関連度を計算する処理、次回の記事で詳しく書く）に渡される。アテンションの出力は再びRMSNormで正規化され、FFN（フィードフォワードネットワーク、ベクトルを非線形変換する処理）に渡される。それぞれのRMSNormは別の学習済みweightを持っていて、Qwen3ではsafetensorsファイルに以下の名前で格納されている。

- アテンションの前: `model.layers.0.input_layernorm.weight`
- FFNの前: `model.layers.0.post_attention_layernorm.weight`

名前に「layernorm」とあるが、実際の処理はRMSNorm。これはLLaMAのアーキテクチャに由来する命名で、Qwen3もこれを踏襲している。名前はモデルの構造をそのまま反映していて、`model.layers.0`が0番目のトランスフォーマー層、`input_layernorm`/`post_attention_layernorm`がその層の中の各RMSNorm、`weight`がそのパラメータ、を意味する。トランスフォーマー層は複数あり（Qwen3-1.7Bでは28層）、各層がそれぞれ別のRMSNorm weightを持っている。

次回はアテンションの実装について書く。
