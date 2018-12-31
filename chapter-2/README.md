
## 第2章：32 ビット整数式の x64 JIT コンパイラを試作する

次章で C コンパイラを開発するための事前準備作業として、変数を含む整数式の計算ができる初歩的なコンパイラを C 言語で開発します。
開発済みのリポジトリは、以下で参照できます。

tenpoku1000/int_calc_compiler: WebAssembly を中間言語に採用した、32 ビット整数式の x64 JIT コンパイラ  
https://github.com/tenpoku1000/int_calc_compiler

以下のようなソースコード(1 つのファイルを入力でき、1 つだけのスコープを持ちます)をコンパイルできることを想定します。

```
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value1 = value2 + 100;
```

外部のアセンブラは利用せず、リンカの開発も省略するため、メモリ上に機械語を直接生成して JIT 実行します。

この章は、以下の各節で構成されています。[2.11 参考文献・資料](11_Bibliography.md)にある参考書や WebAssembly の仕様書、
インテルの公式マニュアルなどを参照しながら読むようにすると、理解が進むと思います。

* [2.1 コンパイラを自作する理由](1_Reason.md)
* [2.2 全体の処理の流れ](2_Flow.md)
* [2.3 コーディングの方針](3_Policy.md)
* [2.4 字句解析](4_Token.md)
* [2.5 構文解析](5_Parse_tree.md)
* [2.6 意味解析](6_Semantic_analysis.md)
* [2.7 中間コード(WebAssembly)生成](7_Wasm.md)
* [2.8 x64 コード生成](8_x64_code.md)
* [2.9 デバッグとテストコード](9_Debug_test.md)
* [2.10 改善を検討すべき点](10_Consideration.md)
* [2.11 参考文献・資料](11_Bibliography.md)

