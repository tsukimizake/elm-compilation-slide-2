## 前回の内容: https://github.com/tsukimizake/elm-compilation-time-slide/

### 前回を三行で

- 会社のelmプロジェクトのコンパイルが遅いのでelm-compilerのプロファイル取って調べた
- プロファイルの結果、 .elmiファイルの型情報のパース時間がほとんどを占めているとわかった
- 細かいことはよくわからんけどレコードを剥いだら倍速になったよ！　やったね！

## 今回はなんで遅くなってたの？という話をもうちょっと詳しくやります

### elmiのフォーマットが悪いという話
型情報を何もかも展開している

例えばこのコードのvarの型
```elm
type alias Three a = {p: a, q: a, r: a}

var: Three Int
```

コンパイラ内部ではこう、elmiでのバイナリエンコードも多少圧縮してあるが意味としては同じ
```hs
  (TAlias 
    (Canonical {_package = Name {_author = author, _project = project}, _module = Main})
    Three 
    [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
    (Filled (TRecord (fromList [(p,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(q,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(r,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int []))]) Nothing))
  ) 
```

シンタックスを目に優しくするとだいたいこう
エイリアス名、エイリアスの型引数、展開後の型が全部ベタ書きされている
```hs
TAlias (author.project.Main.Three)
  [(a, elm.core.Basics.Int[])]
  (Record 
   [(p, elm.core.Basics.Int[])
   ,(q, elm.core.Basics.Int[])
   ,(r, elm.core.Basics.Int[])
   ])
```

Intでなく巨大なレコードが渡ると、elmのコード上では `Three Record` と書いているだけで内部では型が4回展開される。
これが何重にも重なると大変よろしくないことに
```hs
TAlias (author.project.Main.Three)
  [(a, {VERY_BIG_RECORD})]
  (Record 
   [(p, {VERY_BIG_RECORD})
   ,(q, {VERY_BIG_RECORD})
   ,(r, {VERY_BIG_RECORD})
   ])
```

### elmコード上でできる対処
#### 無意味なtype aliasを噛ませない

HogeとHugaの2つのtype aliasを噛ませた場合
```elm
type alias Hoge a = a
type alias Huga a = Hoge a

hoge : Huga Int
hoge = 1
```

```hs
(hoge,
  Forall (fromList []) 
    (TAlias 
      (Canonical {_package = Name {_author = author, _project = project}, _module = Main}) 
      Huga 
        [(a, TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
        (Filled 
          (TAlias 
            (Canonical {_package = Name {_author = author, _project = project}, _module = Main}) 
            Hoge [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
            (Filled (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int []))
            ))))
```

Hugaだけにした場合
```elm
type alias Huga a = a

hoge : Huga Int
hoge = 1
```

上と比べるとFilled以下がシンプルになっている。単純な型引数を一度使うtype aliasを噛ませるだけでその中身が倍に増える
```hs
(hoge,
  Forall (fromList []) 
    (TAlias 
      (Canonical {_package = Name {_author = author, _project = project}, _module = Main})
      Huga 
      [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])] 
      (Filled (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int []))
      ))
```

#### 大きなレコードをOpaque Typeにする

https://discourse.elm-lang.org/t/some-advice-on-elm-compile-performance/604 より

上の例と同様のHoge,Hugaで二重のtype aliasをLibモジュールで定義した場合
```elm
module Main exposing (..)
import Lib exposing(..)

hoge : Hoge
hoge = 1
```

```elm
module Lib exposing(Hoge)

type alias Hoge = Huga Int
type alias Huga a = a
```

```hs
(hoge,
    Forall (fromList []) (TAlias 
      (Canonical {_package = Name {_author = author, _project = project}, _module = Lib})
      Hoge [] (Filled 
        (TAlias (Canonical {_package = Name {_author = author, _project = project}, _module = Lib})
        Huga [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
        (Filled 
          (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])
          )))))
```

Lib.HogeをOpaque typeにした場合

```elm
module Lib exposing(Hoge, getVal, makeHoge)

getVal : Hoge -> Int
getVal (Hoge huga) = huga

makeHoge : Int -> Hoge
makeHoge = Hoge

type Hoge = Hoge (Huga Int)
type alias Huga a = a
```

```elm
module Main exposing (..)
import Lib exposing(..)

hoge : Hoge
hoge = makeHoge 1
```

Lib.Hogeという名前の型だという情報しか展開されなくなる！(型引数がある場合は一度だけ展開される)
```hs
(hoge,
  Forall (fromList []) 
  (TType (Canonical {_package = Name {_author = author, _project = project}, _module = Lib}) Hoge []))
```

## おまけ
https://github.com/tsukimizake/elm-compiler 
elm-compilerを小改造してelmiを人間が読める形でdumpする機能をつけたもの
`cabal v2-exec elm -- --dump-elmi  ~/elm-test-proj/elm-stuff/0.19.1/Main.elmi` のように使用するとモジュールの型情報を吐く



