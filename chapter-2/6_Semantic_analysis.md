
## 2.6 意味解析

構文解析が出力した構文木を読み込んで、名前を管理する処理を意味解析と呼びます。型検査も行うのですが、この章では型が int32_t 型のみなので、次章で説明します。
この章で作るコンパイラでは、
[int_calc_compiler/src/lib/tp_compiler/tp_semantic_analysis.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_semantic_analysis.c)
：tp_semantic_analysis 関数が該当します。名前の管理には、ハッシュ表を使います。

### 2.6.1 検出可能な名前に関するエラーの例

1. 変数の多重定義エラー

```
C:\>int_calc_compiler.exe -l source.txt
tp_semantic_analysis.c(278): ERROR: Duplicate DEFINED_REGISTER_OBJECT at register_defined_variable function.
tp_compiler.c(457): ERROR: Compile failed.

C:\>type source.txt:
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
int32_t value1 = value2 + 100;
```

2. 未定義変数の参照エラー

```
C:\>int_calc_compiler.exe -l source.txt
tp_make_wasm.c(1098): ERROR: use undefined symbol(value3).
tp_compiler.c(457): ERROR: Compile failed.

C:\>type source.txt:
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value1 = value3 + 100;
```

3. 未定義変数への代入エラー

```
C:\>int_calc_compiler.exe -l source.txt
tp_make_wasm.c(1098): ERROR: use undefined symbol(value3).
tp_compiler.c(457): ERROR: Compile failed.

C:\>type source.txt:
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value3 = value2 + 100;
```

### 2.6.2 初歩的な意味解析の方法

名前を管理するためのハッシュ表は、
[int_calc_compiler/src/lib/tp_compiler/tp_compiler.h](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_compiler.h)
の記号表(TP_SYMBOL_TABLE 構造体)の REGISTER_OBJECT_HASH member_object_hash; に格納され、中間コード生成(tp_make_wasm.c)で参照されます。

```
typedef enum register_object_type_ {
    NOTHING_REGISTER_OBJECT = 0,
    DEFINED_REGISTER_OBJECT,
    UNDEFINED_REGISTER_OBJECT,
}REGISTER_OBJECT_TYPE;

typedef struct register_object_ {
    REGISTER_OBJECT_TYPE member_register_object_type;
    uint32_t member_var_index;
}REGISTER_OBJECT;

typedef struct sama_hash_data_{
    REGISTER_OBJECT member_register_object;
    uint8_t* member_string;
}SAME_HASH_DATA;

typedef struct register_object_hash_element_{
    SAME_HASH_DATA member_sama_hash_data[UINT8_MAX + 1];
    struct register_object_hash_element_* member_next;
}REGISTER_OBJECT_HASH_ELEMENT;

typedef struct register_object_hash_{
    size_t member_mask;
    REGISTER_OBJECT_HASH_ELEMENT member_hash_table[UINT8_MAX + 1];
}REGISTER_OBJECT_HASH;
```

ハッシュ関数は、文字列の各バイトを排他的論理和(XOR)するだけの単純な関数です。改良の余地は、あるでしょう。ハッシュ値が衝突したら、ハッシュ表に、
同じハッシュ値を持つ名前を UINT8_MAX + 1 格納することができるようになっています。衝突した名前は配列 member_sama_hash_data を線形探索します。
ハッシュ値が衝突する名前の数が UINT8_MAX + 1 を超えたら、新たに配列を UINT8_MAX + 1 確保して、ポインタでつなぎます。構造体 REGISTER_OBJECT_HASH_ELEMENT が対応する内容です。

```
static uint8_t calc_hash(uint8_t* string)
{
    uint8_t hash = 0;

    while (*string){

        hash ^= *string;

        ++string;
    }

    return hash;
}
```

例として、以下のようなソースコードをコンパイルできることを前提としています。

```
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value1 = value2 + 100;
```

ハッシュ表の内容は、デバッグ用途のために int_calc_object_hash.log ファイルに出力できるようになっています。
以下は、上記の REGISTER_OBJECT_HASH 構造体をテキスト表現にしたものです。DEFINED_REGISTER_OBJECT は定義済みの変数であること、
member_var_index は次節の中間コード生成で利用する変数の番号です。変数の定義時に、記号表(TP_SYMBOL_TABLE 構造体)の uint32_t member_var_count; をインクリメントすることで
変数の番号が得られます。記号表の uint32_t member_var_count; は、中間コードで必要なローカル変数の数をカウントする用途で定義しています。

```
   {
    member_hash_table[89].member_sama_hash_data[0]
    DEFINED_REGISTER_OBJECT
    member_var_index(1)
    member_string(value2)
   }

   {
    member_hash_table[90].member_sama_hash_data[0]
    DEFINED_REGISTER_OBJECT
    member_var_index(0)
    member_string(value1)
   }
```

以下の文法で変数を参照している文法を抜き出してみます。

```
Program -> Statement+
Statement -> Type? variable '=' Expression ';'
Expression -> Term (('+' | '-') Term)*
Term -> Factor (('*' | '/') Factor)*
Factor -> '(' Expression ')' | ('+' | '-')? (variable | constant)
Type -> int32_t
```

以下の文法が該当します。

```
Statement -> variable '=' Expression ';'
Factor -> ('+' | '-')? (variable | constant)
```

これらの文法では、ハッシュ表を検索して名前が見つかったら何もせず、名前が見つからなかったら、UNDEFINED_REGISTER_OBJECT としてハッシュ表に登録します。

変数を定義している文法を抜き出してみます。以下の文法が該当します。この文法の場合に、
ハッシュ表を検索して UNDEFINED_REGISTER_OBJECT として登録されている名前が見つかったら未定義変数の代入エラーに、
DEFINED_REGISTER_OBJECT として登録されている名前が見つかったら変数の多重定義エラーになります。名前が見つからなかったら、DEFINED_REGISTER_OBJECT としてハッシュ表に登録します。

```
Statement -> Type variable '=' Expression ';'
```

意味解析では、構文解析が出力した構文木を読み込んで、名前をハッシュ表に登録したり検索したりすることになりますが、深さ優先探索で構文木をたどっていくことで実現します。
具体的には、構文木の要素に部分木が存在する場合に(つまり子供の木がある場合に)、以下の関数のように自分自身を呼び出す、再帰呼び出しで実装します。

```
static bool search_parse_tree(TP_SYMBOL_TABLE* symbol_table, TP_PARSE_TREE* parse_tree)
{
    bool is_semantic_analysis_success = true;

    TP_PARSE_TREE_ELEMENT* element = parse_tree->member_element;

    size_t element_num = parse_tree->member_element_num;

    for (size_t i = 0; element_num > i; ++i){

        if (TP_PARSE_TREE_TYPE_NULL == element[i].member_type){

            break;
        }

        if (TP_PARSE_TREE_TYPE_NODE == element[i].member_type){

            if ( ! search_parse_tree(symbol_table, (TP_PARSE_TREE*)(element[i].member_body.member_child))){

                is_semantic_analysis_success = false;
            }
        }
    }

    if ( ! variable_reference_check(symbol_table, parse_tree)){

        is_semantic_analysis_success = false;
    }

    return is_semantic_analysis_success;
}
```

### 2.6.3 その他の事項

前項で、構文木をたどっていますが、正しい文法のデータであるか検査するようになっています。文法の要素数の過不足がないか、特定の位置のトークンが不正な種別でないか、などを
確認してエラーの場合にコンパイルを停止します。この場合、構文解析と意味解析で処理内容に相違があるということですから、
内部コンパイラ・エラー(ICE: Internal Compiler Error)となります。

中間コード生成で、最後の文であること示す情報が必要なため、記号表(TP_SYMBOL_TABLE 構造体)の TP_PARSE_TREE\* member_last_statement; に、以下の文法の場合に部分木を
代入するようにしています。複数の文があるソースコードをコンパイルすると何度も代入されますが、最後に代入された部分木が、最後の文として中間コード生成で参照されます。

```
Statement -> Type? variable '=' Expression ';'
```

ハッシュ表に登録する名前は、トークンの文字列をポインタで参照していますので、SAME_HASH_DATA 構造体の uint8_t\* member_string; は free しないように注意する必要があります。
トークン列のメモリ解放の際に、ハッシュ表に登録した名前の実体が解放されます。

