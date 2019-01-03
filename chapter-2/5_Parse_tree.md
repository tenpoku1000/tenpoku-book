
## 2.5 構文解析

字句解析が出力したトークン列を読み込んで、構文木を出力する処理を構文解析と呼びます。
本書の構文木は、後述の非終端記号や括弧が含まれるため、一般的に解析木や具象構文木と呼ばれているものです。
この章で作るコンパイラでは、
[int_calc_compiler/src/lib/tp_compiler/tp_make_parse_tree.c](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_make_parse_tree.c)
：tp_make_parse_tree 関数が該当します。

### 2.5.1 初歩的な構文解析の方法

例として、以下のようなソースコードをコンパイルできることを前提としています。

```
int32_t value1 = (1 + 2) * 3;
int32_t value2 = 2 + (3 * value1);
value1 = value2 + 100;
```

字句解析が出力したトークン列を、トークン列の末尾 TP_SYMBOL_NULL に到達するまで、以下のような文法に従って構文を解釈し、構文木を出力します。
文法の左辺は非終端記号と呼び、基本的に対応する C 言語の関数を作成します。文法の右辺は、C 言語の関数本体のコード内容を表したものと思ってください。
文法の右辺に出現する非終端記号は、対応する C 言語の関数呼び出しになります。文法の右辺に出現する非終端記号以外の要素は、終端記号と呼びます。
終端記号は、字句解析が出力したトークンです。文法の右辺の記載ルールは、正規表現的なものになっています。正規右辺文法と呼んでいる書籍も存在するようです。

```
Program -> Statement+
Statement -> Type? variable '=' Expression ';'
Expression -> Term (('+' | '-') Term)*
Term -> Factor (('*' | '/') Factor)*
Factor -> '(' Expression ')' | ('+' | '-')? (variable | constant)
Type -> int32_t
```

上記の文法の非終端記号 Expression は、Term -> Factor -> Expression と、自分自身を呼び出す実行経路を持ちます。いわゆる再帰呼び出しです。
本書では、このような再帰呼び出しで構文を解釈する、再帰降下構文解析を扱います。再帰降下構文解析では、左再帰の問題を解決する必要がありますが、次章で説明します。

構文木は、
[int_calc_compiler/src/lib/tp_compiler/tp_compiler.h](https://github.com/tenpoku1000/int_calc_compiler/blob/master/src/lib/tp_compiler/tp_compiler.h)
の記号表(TP_SYMBOL_TABLE 構造体)の TP_PARSE_TREE\* member_tp_parse_tree; に格納され、
意味解析(tp_semantic_analysis.c)と、中間コード生成(tp_make_wasm.c)で参照されます。構文木を表す TP_PARSE_TREE 構造体は、以下の通りです。
TP_PARSE_TREE_GRAMMER member_grammer; に文法の種類、size_t member_element_num; に文法に含まれる要素数、TP_PARSE_TREE_ELEMENT\* member_element; に文法の要素が格納されます。
文法の要素は、トークン・構文木の他の部分(部分木と呼びます)・文法の要素の末尾 TP_PARSE_TREE_TYPE_NULL が含まれます。文法の種類は、部分木の種類であるとも言えます。

```
typedef enum TP_PARSE_TREE_TYPE_
{
    TP_PARSE_TREE_TYPE_NULL = 0,
    TP_PARSE_TREE_TYPE_TOKEN,
    TP_PARSE_TREE_TYPE_NODE
}TP_PARSE_TREE_TYPE;

typedef union tp_parse_tree_element_union_{
    TP_TOKEN* member_tp_token;
    struct tp_parse_tree_* member_child;
}TP_PARSE_TREE_ELEMENT_UNION;

typedef struct tp_parse_tree_element_{
    TP_PARSE_TREE_TYPE member_type;
    TP_PARSE_TREE_ELEMENT_UNION member_body;
}TP_PARSE_TREE_ELEMENT;

typedef enum TP_PARSE_TREE_GRAMMER_
{
    TP_PARSE_TREE_GRAMMER_PROGRAM,
    TP_PARSE_TREE_GRAMMER_STATEMENT_1,
    TP_PARSE_TREE_GRAMMER_STATEMENT_2,
    TP_PARSE_TREE_GRAMMER_EXPRESSION_1,
    TP_PARSE_TREE_GRAMMER_EXPRESSION_2,
    TP_PARSE_TREE_GRAMMER_TERM_1,
    TP_PARSE_TREE_GRAMMER_TERM_2,
    TP_PARSE_TREE_GRAMMER_FACTOR_1,
    TP_PARSE_TREE_GRAMMER_FACTOR_2,
    TP_PARSE_TREE_GRAMMER_FACTOR_3
}TP_PARSE_TREE_GRAMMER;

typedef struct tp_parse_tree_{
    TP_PARSE_TREE_GRAMMER member_grammer;
    size_t member_element_num;
    TP_PARSE_TREE_ELEMENT* member_element;
}TP_PARSE_TREE;
```

前節で既出ですが、参照しやすいように、トークンを表す TP_TOKEN 構造体を再掲します。

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

この代入文の右辺は、以下の文法の 2 行目から Expression が対応することが分かります。

```
Program -> Statement+
Statement -> Type? variable '=' Expression ';'
Expression -> Term (('+' | '-') Term)*
Term -> Factor (('*' | '/') Factor)*
Factor -> '(' Expression ')' | ('+' | '-')? (variable | constant)
Type -> int32_t
```

右辺の最初にコンパイルされる部分は、終端記号 '(' と ')' が出現することから、文法： Factor -> '(' Expression ')' が対応しますので、以下の式であることが分かります。

```
(1 + 2)
```

上記の式に関係のない部分を除外して簡略化した文法は、以下のようになります。

```
Expression -> Term ('+' Term)*
Term -> Factor
Factor -> '(' Expression ')' | constant
```

ソースコードを構文解析して得られた構文木は、デバッグ用途のために int_calc_parse_tree.log ファイルに出力できるようになっています。
以下は、上記の TP_PARSE_TREE 構造体をテキスト表現にしたものです。これは 1 + 2 の構文解析結果です。まず、数値 1 が見つかり、次に数値 2 が見つかり、
最後に 1 + 2 が組み立てられました。このように木構造が成長していくことで、最終的に構文木全体を完成させます。構文木はトークンを含むため、
TP_TOKEN 構造体も出力されています。

```
=== Dump parse subtree. ===

   {
    TP_PARSE_TREE_GRAMMER_FACTOR_3
    TP_PARSE_TREE_TYPE_TOKEN
       {
        TP_SYMBOL_CONST_VALUE
        TP_SYMBOL_CONST_VALUE_INT32
         member_line(0)
         member_column(18)
         member_string(1)
         member_i32_value(1)
       }

   }

---------------------------

   {
    TP_PARSE_TREE_GRAMMER_FACTOR_3
    TP_PARSE_TREE_TYPE_TOKEN
       {
        TP_SYMBOL_CONST_VALUE
        TP_SYMBOL_CONST_VALUE_INT32
         member_line(0)
         member_column(22)
         member_string(2)
         member_i32_value(2)
       }

   }

---------------------------

   {
    TP_PARSE_TREE_GRAMMER_EXPRESSION_1
    TP_PARSE_TREE_TYPE_NODE
       {
        TP_PARSE_TREE_GRAMMER_FACTOR_3
        TP_PARSE_TREE_TYPE_TOKEN
           {
            TP_SYMBOL_CONST_VALUE
            TP_SYMBOL_CONST_VALUE_INT32
             member_line(0)
             member_column(18)
             member_string(1)
             member_i32_value(1)
           }

       }

    TP_PARSE_TREE_TYPE_TOKEN
       {
        TP_SYMBOL_PLUS
        TP_SYMBOL_UNSPECIFIED_TYPE
         member_line(0)
         member_column(20)
         member_string()
         member_i32_value(0)
       }

    TP_PARSE_TREE_TYPE_NODE
       {
        TP_PARSE_TREE_GRAMMER_FACTOR_3
        TP_PARSE_TREE_TYPE_TOKEN
           {
            TP_SYMBOL_CONST_VALUE
            TP_SYMBOL_CONST_VALUE_INT32
             member_line(0)
             member_column(22)
             member_string(2)
             member_i32_value(2)
           }

       }

   }

---------------------------
```

### 2.5.2 具体的な構文解析の処理の例(簡略化したソースコードによる説明)

前項で検討した以下の式について、

```
(1 + 2)
```

上記の式に関係のない部分を除外して簡略化した文法は、以下のようになると書きました。

```
Expression -> Term ('+' Term)*
Term -> Factor
Factor -> '(' Expression ')' | constant
```

以下は、文法： Expression -> Term ('+' Term)\* に対応した parse_expression 関数の例です。非終端記号 Term に対応した parse_term 関数を呼んでいるのが分かると思います。
冒頭の TP_TOKEN\* backup_token_position = TP_POS(symbol_table); で、読んでいるトークンの現在位置を保存しておき、構文エラーだったら保存しておいた
トークンの位置まで戻る処理の TP_POS(symbol_table) = backup_token_position; まで goto し、構文解析失敗(文法にマッチしなかった)を意味する NULL ポインタを返しています。
これは、部分木を呼び出し元の関数に返さないことを意味しています。構文解析に成功すると(文法にマッチすると)、MAKE_PARSE_SUBTREE マクロで構文木の部分木を出力します。
これは、次項で説明します。なお、式が入れ子になる段数を無制限にしてしまうと、スタックを確保できる上限を超えてコンパイラが異常終了するため、
NESTING_LEVEL_OF_EXPRESSION_MAXIMUM 以上の入れ子になった場合は、NULL ポインタを返してコンパイル・エラーになるようにしています。

```
#define TP_POS(symbol_table) ((symbol_table)->member_tp_token_position)

#define IS_TOKEN_PLUS(token) ((token) && (TP_SYMBOL_PLUS == (token)->member_symbol))

static const uint8_t NESTING_LEVEL_OF_EXPRESSION_MAXIMUM = 63;

static TP_PARSE_TREE* parse_expression(TP_SYMBOL_TABLE* symbol_table)
{
    if (NESTING_LEVEL_OF_EXPRESSION_MAXIMUM <=
        symbol_table->member_nesting_level_of_expression){

        return NULL;
    }

    ++(symbol_table->member_nesting_level_of_expression);

    TP_TOKEN* backup_token_position = TP_POS(symbol_table);

    // Grammer: Expression -> Term ('+' Term)*
    // Example: (1 + 2)
    {
        TP_PARSE_TREE* tmp_expression_1 = NULL;

        TP_PARSE_TREE* tmp_term_1 = NULL;
        TP_TOKEN* tmp_plus = NULL;
        TP_PARSE_TREE* tmp_term_2 = NULL;

        if (tmp_term_1 = parse_term(symbol_table)){

            while (IS_TOKEN_PLUS(TP_POS(symbol_table))){

                tmp_plus = TP_POS(symbol_table)++;

                if ( ! (tmp_term_2 = parse_term(symbol_table))){

                    goto skip;
                }

                if (NULL == tmp_expression_1){

                    tmp_expression_1 = MAKE_PARSE_SUBTREE(
                        symbol_table,
                        TP_PARSE_TREE_GRAMMER_EXPRESSION_1,
                        TP_TREE_NODE(tmp_term_1),
                        TP_TREE_TOKEN(tmp_plus),
                        TP_TREE_NODE(tmp_term_2)
                    );

                    if (NULL == tmp_expression_1){

                        goto skip;
                    }
                }
            }

            return tmp_expression_1;
        }
skip:
        TP_POS(symbol_table) = backup_token_position;
    }

    return NULL;
}
```

以下は、文法： Term -> Factor に対応した parse_term 関数の例です。非終端記号 Factor に対応した parse_factor 関数を呼んでいるのが分かると思います。

```
static TP_PARSE_TREE* parse_term(TP_SYMBOL_TABLE* symbol_table)
{
    return parse_factor(symbol_table);
}
```

以下は、文法： Factor -> '(' Expression ')' | constant に対応した parse_factor 関数の例です。非終端記号 Expression に対応した parse_expression 関数を呼んでいるのが
分かると思います。読み込んだトークン列が、文法： Factor -> '(' Expression ')' にマッチしなかった場合、トークンの位置を後戻り(バックトラック)して、
文法： Factor -> constant にマッチするか、試みるようになっています。文法： Factor -> constant にマッチした場合、calc_const_value 関数を呼び出し、
トークンの文字列を数値に変換しています。この関数の内容は省略します。式 (1 + 2) は、文法： Factor -> '(' Expression ')' にマッチし、parse_expression 関数が呼ばれ、
parse_expression 関数にて、それ以上は文法： Factor -> '(' Expression ')' にはマッチしないため、その結果として、文法： Expression -> constant '+' constant のように解釈されます。
前項で示した、構文木をテキスト表現にした log ファイルの例のようになります。

```
#define TP_POS(symbol_table) ((symbol_table)->member_tp_token_position)

#define IS_TOKEN_LEFT_PAREN(token) ((token) && (TP_SYMBOL_LEFT_PAREN == (token)->member_symbol))
#define IS_TOKEN_RIGHT_PAREN(token) ((token) && (TP_SYMBOL_RIGHT_PAREN == (token)->member_symbol))
#define IS_TOKEN_CONST_VALUE(token) ((token) && (TP_SYMBOL_CONST_VALUE == (token)->member_symbol))

static TP_PARSE_TREE* parse_factor(TP_SYMBOL_TABLE* symbol_table)
{
    // Grammer: Factor -> '(' Expression ')' | constant
    TP_TOKEN* backup_token_position = TP_POS(symbol_table);

    // Factor -> '(' Expression ')'
    {
        TP_TOKEN* tmp_left_paren = TP_POS(symbol_table);

        if (IS_TOKEN_LEFT_PAREN(tmp_left_paren)){

            ++TP_POS(symbol_table);

            TP_PARSE_TREE* tmp_expression = NULL;

            if (tmp_expression = parse_expression(symbol_table)){

                TP_TOKEN* tmp_right_paren = TP_POS(symbol_table);

                if ( ! IS_TOKEN_RIGHT_PAREN(tmp_right_paren)){

                    goto skip_1;
                }

                ++TP_POS(symbol_table);

                return MAKE_PARSE_SUBTREE(
                    symbol_table,
                    TP_PARSE_TREE_GRAMMER_FACTOR_1,
                    TP_TREE_TOKEN(tmp_left_paren),
                    TP_TREE_NODE(tmp_expression),
                    TP_TREE_TOKEN(tmp_right_paren)
                );
            }
        }
skip_1:
        TP_POS(symbol_table) = backup_token_position;
    }

    // Factor -> constant
    {
        if (IS_TOKEN_CONST_VALUE(TP_POS(symbol_table))){

            TP_TOKEN* tmp_constant = TP_POS(symbol_table)++;

            if ( ! calc_const_value(symbol_table, tmp_constant)){

                return NULL;
            }

            return MAKE_PARSE_SUBTREE(
                symbol_table,
                TP_PARSE_TREE_GRAMMER_FACTOR_3,
                TP_TREE_TOKEN(tmp_constant)
            );
        }
skip_2:
        TP_POS(symbol_table) = backup_token_position;
    }

    return NULL;
}
```

### 2.5.3 構文木の部分木を出力する MAKE_PARSE_SUBTREE マクロの仕組み

前項までに検討した以下の式について、

```
(1 + 2)
```

上記の式に関係のない部分を除外して簡略化した文法は、以下のようになると書きました。

```
Expression -> Term ('+' Term)*
Term -> Factor
Factor -> '(' Expression ')' | constant
```

以下は、文法： Expression -> Term ('+' Term)* に対応した parse_expression 関数で、構文解析に成功(文法にマッチ)した際に、
MAKE_PARSE_SUBTREE マクロで構文木の部分木を出力する部分を抜き出したものです。1 + 2 の各要素は揃っていて、部分木に組み立てる場面です。

```
    // Grammer: Expression -> Term ('+' Term)*
    // Example: (1 + 2)
    {
        TP_PARSE_TREE* tmp_expression_1 = NULL;

        TP_PARSE_TREE* tmp_term_1 = NULL;
        TP_TOKEN* tmp_plus = NULL;
        TP_PARSE_TREE* tmp_term_2 = NULL;

        if (tmp_term_1 = parse_term(symbol_table)){

            while (IS_TOKEN_PLUS(TP_POS(symbol_table))){

                tmp_plus = TP_POS(symbol_table)++;

                if ( ! (tmp_term_2 = parse_term(symbol_table))){

                    goto skip;
                }

                if (NULL == tmp_expression_1){

                    tmp_expression_1 = MAKE_PARSE_SUBTREE(
                        symbol_table,
                        TP_PARSE_TREE_GRAMMER_EXPRESSION_1,
                        TP_TREE_NODE(tmp_term_1),
                        TP_TREE_TOKEN(tmp_plus),
                        TP_TREE_NODE(tmp_term_2)
                    );
```

以下の、MAKE_PARSE_SUBTREE マクロで構文木の部分木を出力する部分は、

```
TP_PARSE_TREE* tmp_expression_1 = MAKE_PARSE_SUBTREE(
    symbol_table,
    TP_PARSE_TREE_GRAMMER_EXPRESSION_1,
    TP_TREE_NODE(tmp_term_1),
    TP_TREE_TOKEN(tmp_plus),
    TP_TREE_NODE(tmp_term_2)
);
```

MAKE_PARSE_SUBTREE マクロの可変長引数が以下のように展開され、TP_PARSE_TREE_ELEMENT 構造体が複数並んだ状態になり、MAKE_PARSE_SUBTREE マクロが
展開された先の make_parse_subtree 関数の第 3 引数に、TP_PARSE_TREE_ELEMENT 構造体の配列として、第 4 引数に TP_PARSE_TREE_ELEMENT 構造体の配列の要素数が渡ります。
MAKE_PARSE_SUBTREE マクロが可変長引数を受け付けるようになっているため、文法の右辺の要素数を固定化する必要がなくなり、MAKE_PARSE_SUBTREE マクロの呼び出しで、
多様な種類の部分木を構築することが可能になっています。

```
#define MAKE_PARSE_SUBTREE(symbol_table, grammer, ...) \
  make_parse_subtree( \
    (symbol_table), (grammer), \
    (TP_PARSE_TREE_ELEMENT[]){ __VA_ARGS__ }, \
    sizeof((TP_PARSE_TREE_ELEMENT[]){ __VA_ARGS__ }) / sizeof(TP_PARSE_TREE_ELEMENT) \
  )
#define TP_TREE_NODE(child) (TP_PARSE_TREE_ELEMENT){ \
    .member_type = TP_PARSE_TREE_TYPE_NODE, \
    .member_body.member_child = (child) \
}
#define TP_TREE_TOKEN(token) (TP_PARSE_TREE_ELEMENT){ \
    .member_type = TP_PARSE_TREE_TYPE_TOKEN, \
    .member_body.member_tp_token = (token) \
}

TP_PARSE_TREE* tmp_expression_1 = MAKE_PARSE_SUBTREE(
    symbol_table,
    TP_PARSE_TREE_GRAMMER_EXPRESSION_1,
    (TP_PARSE_TREE_ELEMENT){
        .member_type = TP_PARSE_TREE_TYPE_NODE,
        .member_body.member_child = (tmp_term_1)
    },
    (TP_PARSE_TREE_ELEMENT){
        .member_type = TP_PARSE_TREE_TYPE_TOKEN,
        .member_body.member_tp_token = (tmp_plus)
    },
    (TP_PARSE_TREE_ELEMENT){
        .member_type = TP_PARSE_TREE_TYPE_NODE,
        .member_body.member_child = (tmp_term_2)
    }
);
```

make_parse_subtree 関数では、構文木の部分木のメモリ領域を確保し、引数で渡される TP_PARSE_TREE_ELEMENT 構造体の配列などを、確保したメモリ領域にコピーしています。
また、文法の要素の最後に、文法の要素の末尾を示す TP_PARSE_TREE_TYPE_NULL を追加します。部分木だけでなく、トークンについてもポインタがコピーされるだけですから、
トークンの実体は字句解析で確保したメモリ領域であることに注意する必要があります。トークンのメモリ領域を解放する場合は、
記号表(TP_SYMBOL_TABLE 構造体)の TP_TOKEN\* member_tp_token; のメモリ領域を解放する必要があります。その他の領域のポインタを free してしまうと多重解放になり、
不具合が生じる原因になります。

```
#define TP_TREE_TOKEN_NULL (TP_PARSE_TREE_ELEMENT){ \
    .member_type = TP_PARSE_TREE_TYPE_NULL, \
    .member_body.member_tp_token = NULL \
}

static TP_PARSE_TREE* make_parse_subtree(
    TP_SYMBOL_TABLE* symbol_table, TP_PARSE_TREE_GRAMMER grammer,
    TP_PARSE_TREE_ELEMENT* parse_tree_element, size_t parse_tree_element_num)
{
    TP_PARSE_TREE* parse_subtree = (TP_PARSE_TREE*)calloc(1, sizeof(TP_PARSE_TREE));

    if (NULL == parse_subtree){

        TP_PRINT_CRT_ERROR(symbol_table);

        return NULL;
    }

    parse_subtree->member_grammer = grammer;
    parse_subtree->member_element_num = parse_tree_element_num;
    parse_subtree->member_element = (TP_PARSE_TREE_ELEMENT*)calloc(
        parse_tree_element_num + 1, sizeof(TP_PARSE_TREE)
    );

    if (NULL == parse_subtree->member_element){

        TP_PRINT_CRT_ERROR(symbol_table);

        TP_FREE(symbol_table, &parse_subtree, sizeof(TP_PARSE_TREE));

        return NULL;
    }

    memcpy(
        parse_subtree->member_element, parse_tree_element,
        sizeof(TP_PARSE_TREE_ELEMENT) * parse_tree_element_num
    );
    TP_PARSE_TREE_ELEMENT null = TP_TREE_TOKEN_NULL;
    memcpy(
        parse_subtree->member_element + parse_tree_element_num, &null,
        sizeof(TP_PARSE_TREE_ELEMENT)
    );

    return parse_subtree;
}
```

