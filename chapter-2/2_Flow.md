
## 2.2 全体の処理の流れ

### 2.2.1 データに着目したコンパイラの処理の流れ

以下の各データは、ソースコードを除き、
[int_calc_compiler/src/lib/tp_compiler/tp_compiler.h](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_compiler.h)
の記号表(TP_SYMBOL_TABLE 構造体)に格納されます。
トークン列の各要素には、ソースコード上の行と桁が記録されます。

1. ソースコード(UTF-8)
2. トークン列
3. 構文木
4. 中間言語(WebAssembly バイナリ表現)
5. x64 フラット・バイナリ・ファイル(flat binary file)

### 2.2.2 実際のコンパイラの処理の流れ

1. [int_calc_compiler/src/main.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/main.c)：main 関数 で、
コンパイラ本体(静的ライブラリ)の tp_compiler 関数を呼び出し

2. [int_calc_compiler/src/lib/tp_compiler/tp_compiler.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_compiler.c)：tp_compiler 関数
が compiler_main 関数を呼び出し

3. compiler_main 関数で、コンパイラの初期化処理 init_symbol_table 関数を呼び出し

### 2.2.2.1 ソースコードをコンパイルする通常の処理の流れ

1. 字句解析(トークン列を出力)：
[int_calc_compiler/src/lib/tp_compiler/tp_make_token.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_token.c)：tp_make_token 関数

2. 構文解析(再帰降下構文解析。構文木を出力)：
[int_calc_compiler/src/lib/tp_compiler/tp_make_parse_tree.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_parse_tree.c)
：tp_make_parse_tree 関数

3. 意味解析(未定義の変数の参照や、重複する変数の定義をチェックする)：
[int_calc_compiler/src/lib/tp_compiler/tp_semantic_analysis.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_semantic_analysis.c)
：tp_semantic_analysis 関数

4. 中間コード生成(WebAssembly バイナリ表現を出力)：
[int_calc_compiler/src/lib/tp_compiler/tp_make_wasm.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_wasm.c)
：tp_make_wasm 関数

5. コード生成(アセンブラを利用せず、機械語を直接出力)：
[int_calc_compiler/src/lib/tp_compiler/tp_make_x64_code.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_x64_code.c)
：tp_make_x64_code 関数

### 2.2.2.2 ソースコードをコンパイルせず、コンパイラに内蔵されたテスト用の WebAssembly 生成処理から処理を開始する場合

1. 中間コード生成：
int_calc_compiler/src/lib/tp_compiler/tp_make_wasm.c：tp_make_wasm 関数

2. コード生成：
int_calc_compiler/src/lib/tp_compiler/tp_make_x64_code.c：tp_make_x64_code 関数

### 2.2.3 その他のソースコード・ファイルについて

* WebAssembly で利用する可変長整数の LEB128 関連の関数定義：
[int_calc_compiler/src/lib/tp_compiler/tp_leb128.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_leb128.c)
* コード生成の実装詳細部(x64 機械語を生成する主な関数定義)：
[int_calc_compiler/src/lib/tp_compiler/tp_make_x64_code_body.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_x64_code_body.c)
* ファイルの読み書きに必要な関数定義：
[int_calc_compiler/src/lib/tp_compiler/tp_file.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_file.c)
* メッセージ出力やメモリ解放などの関数定義：
[int_calc_compiler/src/lib/tp_compiler/tp_utils.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_utils.c)

