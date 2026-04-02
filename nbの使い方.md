# nbの使い方

  nb list                 # 一覧
  notes                   # 一覧の短縮
  nb add                  # 新規作成
  note "タイトル"         # タイトル付きで新規作成
  nb edit 1               # 1番を編集
  noted 1                 # 1番を編集の短縮
  nb show 1               # 中身を表示
  nb peek 1               # ターミナルで軽く確認
  nb open 1               # エディタ/関連付けで開く
  nb search foo           # 検索
  nb delete 1             # 削除
  nb todos                # todo 一覧
  nb todo add "やること"  # todo 追加

  使い分けはこんな感じです。

  - 書き始める: nb add or note "タイトル"
  - 一覧から探す: nb list
  - 番号で開く: nb edit 3
  - 文言で探す: nb search keyword
  - さっと読む: nb show 3

  ノートブック切替もたまに使います。

  nb notebooks            # notebook 一覧
  nb notebooks current    # 今どこを見てるか
  nb use knowledge-notes  # notes 用に戻す
  nb use knowledge        # knowledge 全体を見る

  要するに日常運用はほぼこれです。

  note "タイトル"
  notes
  noted 1
  nb search foo


