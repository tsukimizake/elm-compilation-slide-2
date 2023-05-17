## 前回の内容: https://github.com/tsukimizake/elm-compilation-time-slide/

### 前回を三行で

- 会社のelmプロジェクトのコンパイルが遅いのでelm-compilerのプロファイル取って調べた
- プロファイルの結果、 .elmiファイルの型情報のパース時間がほとんどを占めているとわかった
- 細かいことはよくわからんけどレコードを剥いだら倍速になったよ！　やったね！

## 今回は「なんで遅くなってたの？」という話をもうちょっと詳しくやります

### elmiのフォーマットが悪い
型情報を何もかも展開している

例えばこのコードのvarの型を考える。
```elm
type alias Three a = {p: a, q: a, r: a}

var: Three Int
```

コンパイラ内部ではこう。elmiでのバイナリエンコードも多少圧縮してあるが意味としては同じ
```hs
  (TAlias 
    (Canonical {_package = Name {_author = author, _project = project}, _module = Main})
    Three 
    [(a,TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])]
    (Filled (TRecord (fromList [(p,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(q,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int [])),(r,FieldType 0 (TType (Canonical {_package = Name {_author = elm, _project = core}, _module = Basics}) Int []))]) Nothing))
  ) 
```

シンタックスを目に優しくするとだいたいこう。エイリアス名、エイリアスの型引数、展開後の型が全部ベタ書きされている
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
module Lib exposing(Hoge)

type alias Hoge = Huga Int
type alias Huga a = a
```

```elm
module Main exposing (..)
import Lib exposing(..)

hoge : Hoge
hoge = 1
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

## 劇的ビフォーアフター

SPAの各ページで共有されるSharedデータ型がelmi上で500kbになるところまで太っていてこれが主な原因であるということまで突き止めたため、これをOpaque Type化した。

### ビフォー
```
Success! Compiled 515 modules.

    AdminApp ───> /tmp/adminapp.js

  51,393,437,512 bytes allocated in the heap
  10,860,466,032 bytes copied during GC
   4,068,292,856 bytes maximum residency (8 sample(s))
      11,589,384 bytes maximum slop
            3879 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0        17 colls,     0 par   28.301s  33.395s     1.9644s    8.0567s
  Gen  1         8 colls,     0 par   17.776s  21.548s     2.6935s    10.8063s

  TASKS: 60 (1 bound, 59 peak workers (59 total), using -N16)

  SPARKS: 0(0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.012s  (  0.027s elapsed)
  MUT     time   52.212s  (  9.845s elapsed)
  GC      time   46.077s  ( 54.943s elapsed)
  EXIT    time    0.001s  (  0.007s elapsed)
  Total   time   98.302s  ( 64.821s elapsed)

  Alloc rate    984,315,939 bytes per MUT second

  Productivity  53.1% of total user, 15.2% of total elapsed

```
### アフター

```
Success! Compiled 519 modules.

    AdminApp ───> /tmp/adminapp.js

   8,834,810,432 bytes allocated in the heap
      30,342,376 bytes copied during GC
     149,273,496 bytes maximum residency (3 sample(s))
       1,741,928 bytes maximum slop
             142 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0         3 colls,     0 par    0.741s   0.793s     0.2645s    0.3971s
  Gen  1         3 colls,     0 par    0.315s   0.389s     0.1297s    0.3328s

  TASKS: 60 (1 bound, 59 peak workers (59 total), using -N16)

  SPARKS: 0(0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.012s  (  0.028s elapsed)
  MUT     time    8.826s  (  3.310s elapsed)
  GC      time    1.056s  (  1.183s elapsed)
  EXIT    time    0.001s  (  0.009s elapsed)
  Total   time    9.895s  (  4.529s elapsed)

  Alloc rate    1,000,987,575 bytes per MUT second

  Productivity  89.2% of total user, 73.1% of total elapsed
```
## 一行でまとめ

ごついレコードをOpaque Typeにするとコンパイル時間が改善します

## おまけ
https://github.com/tsukimizake/elm-compiler 

elm-compilerを小改造してelmiを人間が読める形でdumpする機能をつけたもの

`cabal v2-exec elm -- --dump-elmi  ~/elm-test-proj/elm-stuff/0.19.1/Main.elmi` のように使用するとモジュールの型情報を吐く



