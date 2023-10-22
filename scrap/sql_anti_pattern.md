## 2章　ナイーブツリー
### アンチパターン
組織図やニュースサイトのコメント欄などのある程度・制限のない深さがある階層構造を表現するために隣接リストを使用すること

### デメリット
- 深い構造を取得するために多重の再帰呼び出しが必要
  - 再帰をサポートする関数があるが全てのベンダーで実装されているわけではない
- データのメンテナンスコストが高い　

### アンチパターンを使用しても良い場合
- 構造が深くない・深さが決まっている
- ノードの直近の要素の取得が主なタスクである場合

など、要件によってはベストな選択肢となりうる