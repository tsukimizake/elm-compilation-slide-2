## 前回の内容: https://github.com/tsukimizake/elm-compilation-time-slide/

### 前回を三行で

- 会社のelmプロジェクトのコンパイルが遅いのでelm-compilerのプロファイル取って調べた
- プロファイルの結果、 .elmiファイルの型情報が膨れているのが原因らしいとわかった
- 細かいことはよくわからんけどレコードを剥いだら倍速になったよ！　やったね！

## 今回はなんで遅くなってたの？という話をもうちょっと詳しくやります

### elmiのフォーマットが悪いという話
型情報を何もかも展開している

例えばこのコードのvarの型
```
type alias Three a = {p: a, q: a, r: a}

var: Three Int
```

コンパイラ内部ではこう
```
  (TAlias 
    (Canonical {_package = Name {_author = author, _project = project}, _module = Main})
    Three 
    [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
    (Filled (TRecord (fromList [(p,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(q,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(r,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int []))]) Nothing))
  ) 
```

シンタックスを目に優しくするとだいたいこう
```
TAlias (author.project.Main.Three)
  [(a, elm.core.Basics.Int[])]
  (Record 
   [(p, elm.core.Basics.Int[])
   ,(q, elm.core.Basics.Int[])
   ,(r, elm.core.Basics.Int[])
   ])
```

Intでなく巨大なレコードが渡ると？
```
TAlias (author.project.Main.Three)
  [(a, {VERY_BIG_RECORD})]
  (Record 
   [(p, {VERY_BIG_RECORD})
   ,(q, {VERY_BIG_RECORD})
   ,(r, {VERY_BIG_RECORD})
   ])
```

elmのコード上では `Three Record` と書いているだけで内部ではRecordが4回展開される


### elmコード上でできる対処
#### 無意味なtype aliasを噛ませない
実験だるいにゃ！

#### 大きなレコードをOpaque Typeにする
実験だるいにゃ！！
(たくさん使われていたデータ構造型をOpaque Typeにしてみたところさらに20%~30%ほど改善した)

### 前回のコードで何が起きていたか　再現実験あまりにもめんどいのでたぶん消す

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


