---
title: "zapabob/codexを読む"
free: true
---

# zapabob/codexを読む

この章では、公式openai/codexだけでなく、zapabob/codexを題材にします。

ただし、zapabob/codexを公式Codexの代替として扱うわけではありません。  
公式Codexを基準にしながら、フォークがどのように追従し、拡張を隔離し、検証可能な形で運用されるかを読むための実例として扱います。

## この章で見る観点

- upstream-first forkとは何か
- 公式との差分をどう読むか
- 拡張をどこに置くか
- Windows-first releaseをどう検証するか
- Git4DやDeepResearchをどう本編から切り分けるか

## 読む順番

最初から実装の細部へ入ると迷います。

まずは、次の順番で読みます。

1. README
2. release notes
3. repository layout
4. upstreamとの差分
5. build / test / release workflow
6. 独自拡張の境界

## この章の完了条件

読者が次のことを説明できる状態を目指します。

- 公式とフォークの関係
- 追従すべき部分と拡張してよい部分
- 壊れにくいfork運用の考え方
- release evidenceを残す意味
