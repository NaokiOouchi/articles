---
title: "OSSから学ぶGo Generics"
emoji: "🏃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","generics","Tech"]
published: false
---

Generics使うの難しいから事例から学ぼう運動

# チュートリアル
https://go.dev/doc/tutorial/generics

# そもそもGenericsを使わないとどんなコードになるのか
すべて interface{} で受け取るため、呼び出し側で毎回型アサーションが必要
ランタイムエラーが起こりやすい（ミスったら panic）

```go:main.go
package main

import (
	"fmt"
)

func FilterInterface(arr []interface{}, predicate func(interface{}) bool) []interface{} {
	var result []interface{}
	for _, v := range arr {
		if predicate(v) {
			result = append(result, v)
		}
	}
	return result
}

func main() {
	nums := []interface{}{1, 2, 3, 4, 5}
	filtered := FilterInterface(nums, func(x interface{}) bool {
		n := x.(int) // 型アサーション
		return n%2 == 0
	})

	// filteredは[]interface{}型なので型変換必須
	for _, v := range filtered {
		n := v.(int)
		fmt.Println(n)
	}
}
```

Map相当のものを独自実装しようとした時もお願い型アサーションに頼ってランタイムエラーになりがち

```go:main.go
package main

import (
	"fmt"
)

func MapInterface(arr []interface{}, transform func(interface{}) interface{}) []interface{} {
	var result []interface{}
	for _, v := range arr {
		result = append(result, transform(v))
	}
	return result
}

func main() {
	nums := []interface{}{1, 2, 3}
	mapped := MapInterface(nums, func(x interface{}) interface{} {
		n := x.(int)
		return fmt.Sprintf("Val:%d", n)
	})

	// mappedは[]interface{}、呼び出し側でstringに変換必要
	str := mapped[0].(int)
	fmt.Println(str) // panic: interface conversion: interface {} is string, not int

	str1 := mapped[0].(string)
	fmt.Println(str1) // Val:1
}
```

これだと毎回transform内外で型アサーションが必要になるし出力は[]interface{}になるし扱いにくい

# 💡Genericsを使ってるOSSを見てみよう
## samber/lo Map
https://github.com/samber/lo/blob/master/slice.go#L26-L34
### T と R は何者？
1. Map[T any, R any]
`T any` と `R any` が型パラメータとして宣言されている
TとかRは自由命名だけれど宣言自体はそういう文法

1. collection []T
入力は `[]T` というスライス
T は「この関数を呼び出したときに決まる型」
例えば `[]int` や `[]string` など、呼び出し時に決定される

1. iteratee func(item T, index int) R
各要素 `item (型 T)` と `index` を受け取り、型 `R` の戻り値を生成する関数
つまり呼び出す側が好きな型 `R` を指定できる

1. 戻り値 []R
`iteratee` で変換された要素群を `[]R` で返す
つまり呼び出し側のコードは「結果が何のスライスか」を明確に把握できる
コンパイラが「`T` -> `R` に変換する iteratee は正しいか」をチェックしてくれるっぽい

### 使ってみる
```go:main.go
package main

import (
	"fmt"
	"github.com/samber/lo"
)

func main() {
	nums := []int{1, 2, 3}
	strs := lo.Map(nums, func(n int, i int) string {
		return fmt.Sprintf("Val:%d", n)
	}) // strsは[]string型として認識される！
	fmt.Println(strs) // [Val:1 Val:2 Val:3]
}
```

1. 第1引数は `nums`（型は `[]int`）
   `collection[]T`に照らし合わせて、Tがintだと決定づけられる

2. 第2引数は`func(n int, i int) string`
    シグネチャが`iteratee func(item T, index int) R`であるため、`T`は第1引数で決定された`int`、`R`は`string`と決定される
3. 戻り値は`[]string`
    `R` が `string` と決まったので、Map の戻り値は `[]string` になる
    変換後のスライス `strs` は型が明確に `[]string` であるとコンパイラに認識され先述のような型アサーションが不要になる

## samber/lo Filter もついでに見てみる
https://github.com/samber/lo/blob/master/slice.go#L12-L22

入力スライス `[]T` をそのまま受け取り条件に合致した要素だけを抽出して、再び `[]T` で返すという構造
第2引数 `predicate func(item T, index int) bool` は、要素とインデックスを受け取り、真偽値を返す関数

### 使ってみる
```go:main.go
package main
import (
    "fmt"
    "github.com/samber/lo"
)
func main() {
    nums := []int{1, 2, 3, 4, 5}
    evens := lo.Filter(nums, func(n int, i int) bool {
        return n%2 == 0
        fmt.Println(evens) // [2 4]
    })
}
```
1. 第1引数は `nums`（型は `[]int`）
    `collection[]T`に照らし合わせて、Tがintだと決定づけられる
1. 第2引数は`func(n int, i int) bool`
    シグネチャが`predicate func(item T, index int) bool`であるため、`T`は第1引数で決定された`int`
1. 戻り値は`[]int`

## Errorハンドリング周り
https://github.com/samber/lo/blob/master/errors.go#L32-L67

これは使ってみた方がわかりやすい

### 使ってみる
```go:main.go
package main

import (
	"fmt"
	"github.com/samber/lo"
	"strconv"
)

func parseNumber(s string) (int, error) {
	return strconv.Atoi(s)
}

func main() {
	// 例1: (T, error)の組み合わせに適用
	num := lo.Must(parseNumber("42")) 
	fmt.Println(num) // 42
	
	// 例2: boolチェックに使う
	isValid := func(name string) (string, bool) {
		return name, name != ""
	}
	name := lo.Must(isValid("Alice")) 
	fmt.Println(name) // "Alice"

	// 例3: bool判定失敗 -> panic
	// lo.Must(isValid(""))  // panic: not ok
}
```

1. `(T, error)`を渡す例
    `parseNumber("42")` が `(int, error)` を返す
    `err` が `nil` かどうかチェックし、`nil` でなければpanic。nilであれば `val（int）`を返す
2. `(T, bool)`を渡す例
    `isValid("Alice")` が `(string, bool)` を返す
    後ろの返り値が `true` ならOK、`false` ならpanic

プロジェクト的にpanicが困るなら参考にしてエラーハンドリングUtilを作ると良さそう

# Genericsは便利
型アサーションとかその他もろもろ恩恵を受けられる
プロジェクト内で `interface{}` を使った汎用関数があるならGenerics化を試してみよう
Generics以外にも、OSSをリファレンスしてベストプラクティスを学んでみるのも一興