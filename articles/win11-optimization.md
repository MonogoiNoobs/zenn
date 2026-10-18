---
title: "真・Windows 11高速化レジストリ設定"
emoji: "⛰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows","レジストリ"]
published: true
---

（註・ここで言う「Windows 11」はAMD64版を想定）

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Segment Heap`に`Enabled`という名前の`DWORD`値を作成し、値を`3`に設定。

## 解説

システム全体のヒープメモリ割り当て戦略を、新型の「セグメント・ヒープ」に強制する（0ビット目でセグメント・ヒープ有効化、1ビット目で追加の最適化設定を有効化。両方有効化すると`0b11`、すなわち`3`）。
互換性のため、既定では全体で有効化されていない（新しく出たため考慮不要なArm64版を除く）。

マニフェストでビルド時に個別で有効化されている場合もあり（[例・Edge](https://forest.watch.impress.co.jp/docs/news/1259773.html)）、この場合は設定前から既に使用している。

## 出典

https://news.ycombinator.com/item?id=47858177