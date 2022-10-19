---
title: example of symbolic execution using K
---

## Символьное выполнение

По своей сути символьное выполнение (абстрактная интерпретация) - это процесс
выполнения программы в котором часть состояния программы абстрагирована от
конкретных значений. То есть, например, некоторые переменные содержат не
конкретные значения, а некие абстрактные которые обозначаются символами.

Возьмём, например, следующий код:

    int a = 45;
    int b = 2;
    int c = a + b;

Тут всё понятно с процессом выполнения, все значения конкретны, в каждый момент
времени состояние программы конкретно и однозначно.

теперь посмотрим на следующий код:

    int a = 45;
    int b;
    scanf("%d", &b);
    int c = a + b;

Как мы можем выполнить данный код, не зная, какое значение может ввести
пользователь? Проверять все возможные значения переменной `b`?
2^64 ?

В таком случае и используется анализ через абстракцию. Мы просто считаем, что
в переменной `b` есть значение, но оно нам не известно. Примерно то же самое,
что в школьных задачах с неизвестными.

    int a = 45;      // a = 45
    int b;           // b = undefined
    scanf("%d", &b); // b = Value0
    int c = a + b;   // c = 45 + Value0

Теперь мы уже переходим к тому, что часть состояния программы состоих из
конкретных значений, а часть из абстрактных значений и выражений над этими
абстрактными значениями.


Далее будет краткий пример на `K` и `z3` как можно выполнять анализ программ
с помощью абстрактной интерпретации.

## Определение простого модельного языка


### Определение синтаксиса.

Синтаксис языка опишем в модуле `LANG-SYNTAX`.

Из базовых определений `K` возьмём определения синтаксиса идентификаторов,
булевских констант, целых чисел и списков.

```k
module LANG-SYNTAX

  import ID-SYNTAX
  import BOOL-SYNTAX
  import INT-SYNTAX
  import LIST
```

Язык у нас будет с динамической проверкой типов, строгой типизацией
(этим он будет немного похож на `Python`) и явном объявлении типов
переменных (в этом похож на `C`).

Ограничемся пока только целочисленным и булевским типами:

```k
  syntax Type ::= "int" | "bool"
```

Нормальные значения (ground values, то есть которые уже далее не редуцируются)
будут булевскими, целыми числами и символами вида `abstract(0), abstract(2), abstract(N)`:

```k
  syntax Value ::= Int | Bool | "abstract" "(" Int ")"
```

Объявим синтаксис выражений над нормальными значениями:

```k
  syntax InnerExpr ::= Value
                     | "!" InnerExpr
                     > InnerExpr "+" InnerExpr [left]
                     | InnerExpr "-" InnerExpr [left]
                     | InnerExpr "==" InnerExpr [nonassoc]
                     | InnerExpr "<" InnerExpr [nonassoc]
                     | InnerExpr "&&" InnerExpr [left]
                     | InnerExpr "||" InnerExpr [left]
                     | InnerExpr "->" InnerExpr [macro, right]
                     | "(" InnerExpr ")" [bracket]
                     | simplify(InnerExpr) [function, functional]
```

Выражения простые, все должны быть понятны, так как заимствованы почти все из
языка `C`.

Что касается атрибутов в квадратных скобках - это подсказки для k-framework для
генерации парсера (ассоциативность, учёт скобок, макро-подстановка).
Детальнее можно про это почитать в руководстве по k-framework.

Стоит пояснить импликацию `->` - это булевская функция в данном случае, а не
обращение к данным поля структуры по её указателю. Структур и пр. сложных типов
у нас пока в языке не будет.

И синтаксическая продукция `simplify`, которая объявлена функцией (function),
причём тотальной (functional). Сама функция служит для упрощения варажений
(вычисление подвыражений с конкретными константами и пр).
А по поводу `function` и `functional` можно почитать в документации k-framework.


Теперь нам нужны типизированные значения для нашего языка, которые будут
являться результатами вычислений в нашем языке.

Это будут просто пары `тип` (Type) и `выражение` (InnerExpr).

```k

  syntax Result ::= "(" Type "," InnerExpr ")"
```

Синтаксис переменных (это просто идентификаторы):
```k
  syntax Var ::= Id
```


Теперь нужно сказать компилятору формальной опсемантики k-framework, какие
сорты (нетерминалы) он должен считать финальными результатами редукции:

```k
  syntax KResult ::= Result
```

Это делается через субсортинг наших сортов в базовый сорт `KResult`.
Подробнее про это можно прочитать в документации k-framework.


Теперь зададим синтаксис выражений нашего языка:

```k
  syntax Expr ::= Result
                | Int
                | Bool
                | Var
                | "!" Expr       [strict]
                > Expr "+" Expr  [seqstrict, left]
                | Expr "-" Expr  [seqstrict, left]
                | Expr "==" Expr [seqstrict, nonassoc]
                | Expr "<" Expr  [seqstrict, nonassoc]
                | Expr "&&" Expr [seqstrict, left]
                | Expr "||" Expr [seqstrict, left]
                | Expr "->" Expr [macro, right]
                | "(" Expr ")"   [bracket]
```

Пояснение относительно `Int` и `Bool`. У нас в языке валидны только значения
сорта `Result`, но пользователю неудобно вводить `int a = (int, 43);`, поэтому
мы даём возмоность пользователю писать `int a = 42;`, но сами тут же это `42`
конвертируем в `(int, 42)`. Это будет видно в дальнейшем, когда будем
определять правила редукции.

Помимо выражений, которые возвращают результат, у нас в языке будут
утверждения (statement), которые результата не возвращают:

```k
  syntax Stmt ::=
```

- объявление переменной:

```k
                  Type Var "=" Expr ";" [strict(3)]
```
- присваивание:

```k
                | Var "=" Expr ";" [strict(2)]
```

- условное ветвление:

```k
                | "if" Expr Stmt "else" Stmt [strict(1)]
                | "if" Expr Stmt [macro]
```

- блоки утверждений:
```k
                | "{" "}"
                | "{" Stmt "}" [bracket]
```

- ассерты:
```k
                | "assert" Expr ";" [strict]
```

Список утверждений:
```k
  syntax Stmts ::= List{Stmt, ""}
```

Правила раскрытия макросов:

```k
  rule if B S => if B S else { }
  rule E1:Expr -> E2 => (! E1) || E2
  rule E1:InnerExpr -> E2 => simplify((! (simplify(E1))) || simplify(E2))
endmodule

module LANG-CONFIG

  import LANG-SYNTAX
  import MAP

  configuration
  <branches>
    <branch multiplicity="*" type="Set">
      <id> 0 </id>
      <k> $PGM:Stmts </k>
      <vars> .Map </vars>
      <preds> .List </preds>
    </branch>
  </branches>
  <nextId> 1 </nextId>

endmodule

module LANG-ERROR
  import LANG-SYNTAX
  syntax Error ::= AssignmentTypingError(KItem, Type, Type)
                 | AlreadyDeclared(KItem, Var)
                 | ExprBinTypingError(KItem, Type, Type, Type)
                 | ExprEqTypingError(KItem, Type, Type)
                 | ExprUnTypingError(KItem, Type, Type)
                 | IfTypingError(KItem, Type, Type)
                 | AssertIsAlwaysFalse(KItem)
endmodule

module LANG-SIMPLIFY
  import LANG-SYNTAX
  import LANG-ERROR
  import INT
  import BOOL

  rule simplify(A:Int + B:Int) => A +Int B
  rule simplify(A:Int - B:Int) => A -Int B
  rule simplify(A:Int == B:Int) => A ==Int B
  rule simplify(A:Bool == B:Bool) => A ==Bool B
  rule simplify(A:Int < B:Int) => A <Int B
  rule simplify(A:Bool && B:Bool) => A andBool B
  rule simplify(false && B) => false requires notBool(isBool(B))
  rule simplify(A && false) => false requires notBool(isBool(A))
  rule simplify(true && A) => A requires notBool(isBool(A))
  rule simplify(A && true) => A requires notBool(isBool(A))
  rule simplify(A:Bool || B:Bool) => A orBool B
  rule simplify(true || A) => true requires notBool(isBool(A))
  rule simplify(A || true) => true requires notBool(isBool(A))
  rule simplify(false || A) => A requires notBool(isBool(A))
  rule simplify( A || false) => A requires notBool(isBool(A))
  rule simplify(! A:Bool) => notBool(A)
  rule simplify(! ! A) => A

  rule simplify(A) => A
  [owise]

endmodule

module LANG
  import LANG-SYNTAX
  import LANG-CONFIG
  import LANG-SIMPLIFY
  import LANG-ERROR
  import ID
  import BOOL
  import INT
  import K-EQUAL

  rule S:Stmt Ss:Stmts => S ~> Ss
  rule .Stmts => .

  rule <k> T:Type V:Var = (T, _) #as Val ; => . ...</k>
       <vars> Vs => Vs [V <- Val] </vars>
       requires notBool(V in_keys(Vs))

  rule <k> V:Var = (T, Val) ; => . ...</k>
       <vars>... V |-> (T, _ => Val) </vars>

  rule <k> _T:Type V:Var = _ ; #as S => AlreadyDeclared(S, V)  ...</k>
       <vars> ... V |-> _ ... </vars>
       [owise]

  rule <k> V:Var = (T1, _) ; #as S => AssignmentTypingError(S, T1, T2) ...</k>
       <vars>... V |-> (T2, _) </vars>
       [owise]

  rule <k> V:Var => Val ... </k>
       <vars>... V |-> Val ... </vars>

  rule V:Int => (int, V)
  rule V:Bool => (bool, V)

  // inner
  rule (int, V1) + (int, V2) => (int, simplify(V1 + V2))
  rule (int, V1) - (int, V2) => (int, simplify(V1 - V2))
  rule (T:Type, V1) == (T, V2) => (bool, simplify(V1 == V2))
  rule (int, V1) < (int, V2) => (bool, simplify(V1 < V2))
  rule (bool, V1) && (bool, V2) => (bool, simplify(V1 && V2))
  rule (bool, V1) || (bool, V2) => (bool, simplify(V1 || V2))
  rule (bool, V1) -> (bool, V2) => (bool, simplify(V1 -> V2))
  rule ! (bool, V1) => (bool, simplify(! V1))

  // handling typing errors
  rule (T1, _) + (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) - (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) == (T2, _) #as E => ExprEqTypingError(E, T1, T2)        [owise]
  rule (T1, _) < (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) && (T2, _) #as E => ExprBinTypingError(E, bool, T1, T2) [owise]
  rule (T1, _) || (T2, _) #as E => ExprBinTypingError(E, bool, T1, T2) [owise]
  rule (T1, _) -> (T2, _) #as E => ExprBinTypingError(E, bool, T1, T2) [owise]
  rule ! (T, _)           #as E => ExprUnTypingError(E, bool, T)       [owise]

  // if
  rule if (bool, true) S else _  => S
  rule if (bool, false) _ else S => S

  // split if stmt
  rule
    <branch>
    ...
    <k> if (bool, Pred) S1 else S2 ~> Rest => S1 ~> Rest </k>
    <vars> Vars </vars>
    <preds> Ps => Ps ListItem(Pred) </preds>
    </branch>
    (.Bag =>
    <branch>
      <id> N </id>
      <k> S2 ~> Rest </k>
      <vars> Vars </vars>
      <preds> Ps ListItem(simplify(! Pred)) </preds>
    </branch>)
    <nextId> N => N +Int 1 </nextId>
    requires notBool(isBool(Pred))

  rule if (T, _) _ else _ #as S => IfTypingError(S, bool, T) [owise]

  // assert processing
  syntax InnerExpr ::= collectPreds(InnerExpr, List) [function, functional]

  rule collectPreds(IExpr, .List) => IExpr
  rule collectPreds(IExpr => IExpr && P, ListItem(P) Rest => Rest)
  rule collectPreds(_, _) => false [owise]
  rule
    <k> assert (bool, Pred => simplify(collectPreds(true, Preds) -> Pred)) ; ... </k>
    <preds> Preds => .List </preds>
    requires size(Preds) =/=Int 0

  rule assert (T, _); #as S => ExprUnTypingError(S, bool, T)
       requires T =/=K bool

  // assert is proven to be true, so just continue
  rule assert (bool, true) ; => .

  // assert is proven to be alwasy false, stop reduction
  rule assert (bool, false) ; #as S => AssertIsAlwaysFalse(S)

  // remove branches w/o interesting points
  rule (<branch> ... <k> .K </k> ... </branch>) => .Bag

endmodule
```
