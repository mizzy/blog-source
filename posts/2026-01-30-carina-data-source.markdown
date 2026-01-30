---
title: Carinaにデータソース機能を追加
date: 2026-01-30 17:39:23 +0900
---

[Carina](https://github.com/mizzy/carina)に`read`キーワードを追加し、既存のインフラリソースを参照できるようにした。Terraformのdata sourceに相当する機能。

## 新しい構文

`read`キーワードを使って、既存のリソースを参照できる。

```crn
provider aws {
  region = aws.Region.ap_northeast_1
}

# 既存のVPCを読み取る（データソース）
let existing_vpc = read aws.vpc {
  name = "production-vpc"
}

# 既存のVPC内に新しいサブネットを作成
let app_subnet = aws.subnet {
  name              = "app-subnet"
  vpc_id            = existing_vpc.id
  cidr_block        = "10.0.100.0/24"
  availability_zone = aws.AvailabilityZone.ap_northeast_1a
}
```

`read`で定義したリソースは、ライフサイクルを管理しない。作成・更新・削除は行わず、現在の状態を読み取るだけ。

## プラン表示

`+`（作成）、`~`（更新）、`-`（削除）に加えて、`<=`（読み取り）が表示される。(Ghostty上だと違う文字になってるけど)

![プラン表示](/images/2026-01-30-carina-data-source-plan.png)

## 実装

内部的には、`Resource`構造体に`read_only`フィールドを追加した。

```rust
pub struct Resource {
    pub id: ResourceId,
    pub attributes: HashMap<String, Value>,
    pub read_only: bool,  // データソースの場合true
}
```

`Effect`列挙型の`Read`バリアントも変更し、リソース全体を保持するようにした。

```rust
pub enum Effect {
    Read { resource: Resource },  // 変更前: Read(ResourceId)
    Create(Resource),
    Update { id: ResourceId, from: State, to: Resource },
    Delete(ResourceId),
}
```

パーサーは、`read`キーワードに続くリソース定義を解析し、`read_only = true`のリソースを生成する。プラン生成時、`read_only`なリソースは通常のdiff処理をスキップし、常に`Effect::Read`を生成する。

## 関連リンク

- [Add read keyword for data sources by mizzy · Pull Request #35 · mizzy/carina](https://github.com/mizzy/carina/pull/35)
