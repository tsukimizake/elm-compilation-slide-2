## 前回の内容: https://github.com/tsukimizake/elm-compilation-time-slide/

### 前回を三行で

- 会社のelmプロジェクトのコンパイルが遅いのでelm-compilerのプロファイル取って調べた
- プロファイルの結果、 .elmiファイルの型情報が膨れているのが原因らしいとわかった
- 細かいことはよくわからんけどレコード(とtype alias)を剥いだら倍速になったよ！　やったね！

## 今回はなんで遅くなってたの？という話をもうちょっと詳しくやります

### elmiのフォーマットが悪いという話
型情報を何もかも展開している
- putのhaskellソース

### 特にまずいのはレコードというかエイリアス
何もかも展開している！！！
- putのhaskellソース

### 前回のコードで何が起きていたか

```
type alias PageModule urlParams model msg outerMsg =
    { init : Shared -> urlParams -> ( HasShared model, Cmd msg )
    , update : msg -> HasShared model -> ( HasShared model, Cmd msg )
    , subscriptions : HasShared model -> Sub msg
    , subscribeToSharedUpdate : HasShared model -> ( HasShared model, Cmd msg )
    , view : (msg -> outerMsg) -> HasShared model -> Browser.Styled.Document outerMsg
    }
    
type alias HasShared model =
    { model | shared : Shared }
```

Sharedレコードが一度使われるごとに約400kbの型定義がelmiファイルに吐かれる。msgやouterMsgはちゃんと見ていないけれどそこまでではないはず


### でかいレコードをOpaque Typeにすると良くなるという話
- discourse和訳してコピペ予定

たくさん使われていたデータ構造型をOpaque Typeにしてみたところさらに20%~30%ほど改善した
