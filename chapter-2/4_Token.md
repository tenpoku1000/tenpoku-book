
## 2.4 字句解析

ソースコードを読み込んで、後述のトークン列を出力する処理を字句解析と呼びます。
この章で作るコンパイラでは、
[int_calc_compiler/src/lib/tp_compiler/tp_make_token.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_token.c)
：tp_make_token 関数が該当します。

### 2.4.1 トークン列の出力

例として、以下のようなソースコードをコンパイルできることを前提としています。

```
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value1 = value2 + 100;
```

ソースコード中の文字列を、以下のようなルールに従って、トークンという単位に切り出します。ルールの左辺は正規表現です。
正規表現を実装すると大がかりになってしまうため、手書きのコードで、ソースコードを読み込んだバッファの先頭から 1 バイトずつ文字の種類を判別することで
文字列をトークン化する処理を実装しています。空白・改行・ASCII コードの制御文字は読み捨てます。

```
'+' = TP_SYMBOL_PLUS
'-' = TP_SYMBOL_MINUS
'*' = TP_SYMBOL_MUL
'/' = TP_SYMBOL_DIV
'(' = TP_SYMBOL_LEFT_PAREN
')' = TP_SYMBOL_RIGHT_PAREN
'=' = TP_SYMBOL_EQUAL
';' = TP_SYMBOL_SEMICOLON
[0-9]+ = TP_SYMBOL_CONST_VALUE(数値)
[^0-9+-*/()=;][^+-*/()=;]* = TP_SYMBOL_ID(識別子)
```

上記のソースコードの 1 行目を字句解析すると、以下のようにトークン化されます。これはイメージで、実際には後述の TP_TOKEN 構造体の配列で表現されます。

```
int32_t
value1
=
(
1
+
2
)
*
3
;
```

トークン列は、
[int_calc_compiler/src/lib/tp_compiler/tp_compiler.h](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_compiler.h)
の記号表(TP_SYMBOL_TABLE 構造体)の TP_TOKEN\* member_tp_token; に格納されます。
トークン列の各要素には、ソースコード上の行と桁が記録されます。数値や識別子などの文字列が格納される member_string メンバ変数は固定長の配列とし、
文字列の長さに制限を設けています。ソースコードを読み終わった時に、トークン列の末尾は TP_SYMBOL_NULL で終端されるようになっています。
トークンは、構文解析(tp_make_parse_tree.c)と、意味解析(tp_semantic_analysis.c)で参照されます。トークンを表す TP_TOKEN 構造体は、以下の通りです。

```
typedef enum TP_SYMBOL_
{
    TP_SYMBOL_NULL,
    TP_SYMBOL_ID,
    TP_SYMBOL_CONST_VALUE,
    TP_SYMBOL_PLUS,
    TP_SYMBOL_MINUS,
    TP_SYMBOL_MUL,
    TP_SYMBOL_DIV,
    TP_SYMBOL_LEFT_PAREN,
    TP_SYMBOL_RIGHT_PAREN,
    TP_SYMBOL_EQUAL,
    TP_SYMBOL_SEMICOLON
}TP_SYMBOL;

typedef enum TP_SYMBOL_TYPE_
{
    TP_SYMBOL_UNSPECIFIED_TYPE,
    TP_SYMBOL_ID_INT32,
    TP_SYMBOL_TYPE_INT32,
    TP_SYMBOL_CONST_VALUE_INT32
}TP_SYMBOL_TYPE;

#define TP_MAX_ID_BYTES 63
#define TP_ID_SIZE (TP_MAX_ID_BYTES + 1)

typedef struct tp_token_{
    TP_SYMBOL member_symbol;
    TP_SYMBOL_TYPE member_symbol_type;
    rsize_t member_line;
    rsize_t member_column;
    uint8_t member_string[TP_ID_SIZE];
    int32_t member_i32_value;
}TP_TOKEN;
```

コンパイル対象のソースコードの 1 行目を、もう一度見てみましょう。

```
int32_t value1 = (1 + 2) * 3;
```

このソースコードを字句解析して得られたトークン列は、デバッグ用途のために int_calc_token.log ファイルに出力できるようになっています。
以下は、上記の TP_TOKEN 構造体をテキスト表現にしたものです。1 つのトークンが { } で囲まれていて、その中に TP_TOKEN 構造体の内容が
入っていることがお分かりいただけるでしょうか。

```
   {
    TP_SYMBOL_ID
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(0)
     member_string(int32_t)
     member_i32_value(0)
   }

   {
    TP_SYMBOL_ID
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(8)
     member_string(value1)
     member_i32_value(0)
   }

   {
    TP_SYMBOL_EQUAL
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(15)
     member_string()
     member_i32_value(0)
   }

   {
    TP_SYMBOL_LEFT_PAREN
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(17)
     member_string()
     member_i32_value(0)
   }

   {
    TP_SYMBOL_CONST_VALUE
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(18)
     member_string(1)
     member_i32_value(0)
   }

   {
    TP_SYMBOL_PLUS
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(20)
     member_string()
     member_i32_value(0)
   }

   {
    TP_SYMBOL_CONST_VALUE
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(22)
     member_string(2)
     member_i32_value(0)
   }

   {
    TP_SYMBOL_RIGHT_PAREN
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(23)
     member_string()
     member_i32_value(0)
   }

   {
    TP_SYMBOL_MUL
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(25)
     member_string()
     member_i32_value(0)
   }

   {
    TP_SYMBOL_CONST_VALUE
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(27)
     member_string(3)
     member_i32_value(0)
   }

   {
    TP_SYMBOL_SEMICOLON
    TP_SYMBOL_UNSPECIFIED_TYPE
     member_line(0)
     member_column(28)
     member_string()
     member_i32_value(0)
   }
```

### 2.4.2 ソースコードの読み込み時の処理

自動生成された巨大なソースコードを受け付けないようにする目的で、TP_MAX_FILE_BYTES を超えるサイズのファイルはコンパイル・エラーにしています。

```
#define TP_MAX_LINE_BYTES 4095
#define TP_BUFFER_SIZE (TP_MAX_LINE_BYTES + 1)
#define TP_MAX_FILE_BYTES (TP_MAX_LINE_BYTES * 4096)
```

改行は LF に統一します。改行が CR の場合に LF に置き換え、CR/LF の場合は CR を空白に置き換えます。

ファイルやソケットなどの信頼できない入力は、バッファの途中に NUL 文字が混入していることがあるため、予期せぬ不具合の原因になります。
バイナリモードでファイルを開き、固定長 TP_MAX_LINE_BYTES のバッファに fread() でファイルを読み込むことで、バッファの途中に NUL 文字が混入していても、
NUL 文字を空白に置き換えることが可能になります。

入力するソースコードが UTF-8 であるか、バリデーション(妥当性確認)します。先頭に BOM(Byte Order Mark: 0xEF, 0xBB, 0xBF) が含まれていたら、空白で置き換えます。

1. ASCII コードの範囲内だったら、1 バイトのコード
2. 第 1 バイトが、0xC0 ～ 0xDF の範囲内だったら、2 バイトのコード
3. 第 1 バイトが、0xE0 ～ 0xEF の範囲内だったら、3 バイトのコード
4. 第 1 バイトが、0xF0 ～ 0xF7 の範囲内だったら、4 バイトのコード
5. 第 2 バイト以降が、0x80 ～ 0xBF の範囲外だったら、不正な UTF-8
6. 第 1 バイトが、0xC0・0xE0・0xF0 いずれかの場合に、第 2 バイト以降 & 0x3F の結果が、最終バイト以外が全部 0x00 だったら、冗長な ASCII コードで、不正な UTF-8

