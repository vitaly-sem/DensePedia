# F# на примерах — от новичка до уверенного использования

---

## Почему синтаксис F# такой компактный?

F# избавляется от синтаксического шума: нет фигурных скобок, точек с запятой, ключевого слова `return`. Блоки определяются **отступами** (как в Python). Типы выводятся автоматически. В результате код читается как псевдокод на доске.

---

## 1. Функции — основа всего

```fsharp
// Простая функция
let square x = x * x                    // int -> int
square 5                                // 25

// Типы выводятся, но можно указать явно
let add (x: int) (y: int) : int = x + y
add 3 4                                 // 7

// Функция как значение
let operation = add
operation 10 20                         // 30

// Лямбда
let double = fun x -> x * 2
let double' x = x * 2                   // То же самое, короче
```

---

## 2. Кортежи — лёгкая группировка

```fsharp
let person = ("Alice", 30, "London")    // string * int * string

// Декомпозиция
let name, age, city = person
printfn "%s is %d and lives in %s" name age city

// Кортежи в функциях
let distance (x1, y1) (x2, y2) =
    sqrt ((x2 - x1) ** 2.0 + (y2 - y1) ** 2.0)

distance (0.0, 0.0) (3.0, 4.0)         // 5.0

// Кортеж из трёх элементов — другой тип, чем из двух
let twoTuple = 1, 2                     // int * int
let threeTuple = 1, 2, 3               // int * int * int
```

---

## 3. Списки, массивы, последовательности

```fsharp
// Списки (неизменяемые, связные)
let numbers = [1; 2; 3; 4; 5]
let empty = []
let withHead = 0 :: numbers             // [0; 1; 2; 3; 4; 5]
let concatenated = numbers @ [6; 7]     // [1; 2; 3; 4; 5; 6; 7]

// Массивы (изменяемые, фиксированный размер)
let array = [| 1; 2; 3; 4; 5 |]
array[0] <- 10                          // Мутация!
// array: [| 10; 2; 3; 4; 5 |]

// Последовательности (ленивые)
let infiniteOnes = Seq.initInfinite (fun _ -> 1)
let firstTen = infiniteOnes |> Seq.take 10 |> Seq.toList
// [1; 1; 1; 1; 1; 1; 1; 1; 1; 1]

// Генераторы списков
let squares = [ for i in 1..5 -> i * i ]        // [1; 4; 9; 16; 25]
let combinations = [ for x in 1..3 do for y in 1..2 -> (x, y) ]
// [(1,1); (1,2); (2,1); (2,2); (3,1); (3,2)]
```

---

## 4. Pipeline — вместо вложенных вызовов

```fsharp
// Без pipeline: читаем изнутри наружу
let result = List.sum (List.filter (fun x -> x > 20) (List.map (fun x -> x * x) [1..10]))

// C pipeline: читаем слева направо — как думаем
let result =
    [1..10]
    |> List.map (fun x -> x * x)        // Квадраты: [1;4;9;16;25;36;49;64;81;100]
    |> List.filter (fun x -> x > 20)    // >20: [25;36;49;64;81;100]
    |> List.sum                         // Сумма: 355

// Pipeline с оператором композиции функций
let squareAll = List.map (fun x -> x * x)
let filterLarge = List.filter (fun x -> x > 20)
let processData = squareAll >> filterLarge >> List.sum

let result2 = processData [1..10]       // 355 — тот же результат
```

---

## 5. Pattern Matching — швейцарский нож

```fsharp
// По значению
let describe n =
    match n with
    | 0 -> "Zero"
    | 1 -> "One"
    | x when x < 0 -> "Negative"
    | x -> sprintf "Positive: %d" x

// По типу (Discriminated Union)
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of a: float * b: float * c: float

let area shape =
    match shape with
    | Circle r -> Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(a, b, c) ->
        let s = (a + b + c) / 2.0
        sqrt (s * (s - a) * (s - b) * (s - c))

area (Circle 5.0)                    // 78.54
area (Rectangle(3.0, 4.0))           // 12.0

// По спискам (рекурсивно)
let rec sumList list =
    match list with
    | [] -> 0
    | head :: tail -> head + sumList tail

sumList [1; 2; 3; 4; 5]             // 15

// Guard clauses
let classifyAge age =
    match age with
    | a when a < 0 -> "Not born"
    | a when a < 18 -> "Minor"
    | a when a < 65 -> "Adult"
    | _ -> "Senior"
```

---

## 6. Option — вместо null

```fsharp
// Option — либо Some value, либо None
let tryDivide numerator denominator =
    if denominator = 0.0 then None
    else Some (numerator / denominator)

// Использование через pattern matching
match tryDivide 10.0 2.0 with
| Some result -> printfn "Result: %f" result
| None -> printfn "Cannot divide by zero"

// Через Option.map (не трогаем None)
let result = tryDivide 10.0 0.0 |> Option.map (fun x -> x * 2.0)
// None — map не применяется к None

// Option.defaultValue — значение по умолчанию
let safe = tryDivide 10.0 0.0 |> Option.defaultValue 0.0
// 0.0

// Option.bind — цепочка операций, которые могут вернуть None
type Db = { GetUser: int -> User option; GetOrders: User -> Order list option }

let getUserOrders (db: Db) userId =
    db.GetUser userId
    |> Option.bind db.GetOrders
    |> Option.defaultValue []
```

---

## 7. Result — типизированные ошибки

```fsharp
type ValidationError =
    | Empty of fieldName: string
    | TooShort of fieldName: string * minLength: int
    | InvalidFormat of fieldName: string * expected: string

let validateName (name: string) =
    if String.IsNullOrWhiteSpace name then Error (Empty "Name")
    elif name.Length < 2 then Error (TooShort ("Name", 2))
    else Ok name

let validateAge (age: int) =
    if age < 0 then Error (InvalidFormat ("Age", "non-negative"))
    elif age > 150 then Error (InvalidFormat ("Age", "<= 150"))
    else Ok age

// Композиция валидаций через Result
type Person = { Name: string; Age: int }

let createPerson name age =
    match validateName name, validateAge age with
    | Ok validName, Ok validAge -> Ok { Name = validName; Age = validAge }
    | Error e, _ | _, Error e -> Error e

createPerson "Al" 30       // Error (TooShort ("Name", 2))
createPerson "Alice" 30    // Ok { Name = "Alice"; Age = 30 }
```

---

## 8. Коллекции — модули List, Array, Seq

```fsharp
let nums = [1..10]

// List — строгие, неизменяемые
nums |> List.filter (fun n -> n % 2 = 0)          // [2;4;6;8;10]
nums |> List.map (fun n -> n * n)                  // [1;4;9;16;25;36;49;64;81;100]
nums |> List.fold (+) 0                            // 55 (сумма)
nums |> List.reduce max                            // 10
nums |> List.partition (fun n -> n % 2 = 0)        // ([2;4;6;8;10], [1;3;5;7;9])
nums |> List.chunkBySize 3                         // [[1;2;3];[4;5;6];[7;8;9];[10]]
nums |> List.pairwise                              // [(1,2);(2,3);(3,4);...]
nums |> List.windowed 3                            // [[1;2;3];[2;3;4];...]

// Array — изменяемые, для производительности
let arr = [|1..10|]
arr |> Array.map (fun n -> n * 2)                  // [|2;4;6;...;20|]
Array.sortInPlace arr

// Seq — ленивые, для больших/бесконечных данных
Seq.initInfinite (fun i -> i * i)
|> Seq.take 5
|> Seq.toList                                       // [0;1;4;9;16]
```

---

## 9. Рекурсия — вместо циклов

```fsharp
// Обычная рекурсия
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

factorial 5  // 120

// Tail recursion — не растёт стек
let factorialTR n =
    let rec loop acc n =
        if n <= 1 then acc
        else loop (acc * n) (n - 1)
    loop 1 n

factorialTR 5  // 120 (без роста стека)

// Рекурсия по списку
let rec map f list =
    match list with
    | [] -> []
    | head :: tail -> f head :: map f tail

map (fun x -> x * 2) [1;2;3;4;5]  // [2;4;6;8;10]

// Взаимная рекурсия
let rec isEven n =
    if n = 0 then true else isOdd (n - 1)
and isOdd n =
    if n = 0 then false else isEven (n - 1)
```

---

## 10. Async — асинхронность без async/await

```fsharp
open System.Net.Http

let httpClient = new HttpClient()

// Асинхронная функция
let downloadPageAsync (url: string) = async {
    let! response = httpClient.GetStringAsync(url) |> Async.AwaitTask
    return response.Length
}

// Параллельное выполнение
let urls = [
    "https://fsharp.org"
    "https://dotnet.microsoft.com"
    "https://github.com"
]

let downloadAllAsync = async {
    let! results =
        urls
        |> List.map downloadPageAsync
        |> Async.Parallel         // Запускает ВСЕ параллельно!
    return results
}

let lengths = downloadAllAsync |> Async.RunSynchronously
// [||] — массив длин страниц, загруженных параллельно
```

---

## 11. Скриптинг (.fsx) — C# так не умеет

```fsharp
// script.fsx — выполняется без компиляции проекта
#r "nuget: Newtonsoft.Json, 13.0.3"

open System.IO
open Newtonsoft.Json

type Weather = JsonConvert.DeserializeObject<{| Temp: float; Humidity: int |}>

let analyzeWeather (jsonPath: string) =
    let json = File.ReadAllText jsonPath
    let data = JsonConvert.DeserializeObject<Weather>(json)
    printfn "Temperature: %.1f°C, Humidity: %d%%" data.Temp data.Humidity

analyzeWeather "weather.json"
```

Запуск: `dotnet fsi script.fsx` — ни проекта, ни .csproj, ни компиляции. Идеально для автоматизации, миграций, прототипов.

---

## 12. Объектно-ориентированный код в F# (да, можно!)

```fsharp
// Интерфейсы
type ILogger =
    abstract Log: string -> unit

// Класс
type ConsoleLogger(prefix: string) =
    interface ILogger with
        member _.Log msg = printfn "[%s] %s" prefix msg

// Использование
let logger = ConsoleLogger("INFO")
(logger :> ILogger).Log("Application started")

// object expression — реализация интерфейса «на лету»
let testLogger = { new ILogger with member _.Log msg = () }
```

---

## Карманный справочник синтаксиса

| Конструкция | Синтаксис | Пример |
|---|---|---|
| Функция | `let f x = ...` | `let add x y = x + y` |
| Лямбда | `fun x -> ...` | `fun x -> x * 2` |
| Кортеж | `(a, b, c)` | `(1, "Alice", true)` |
| Список | `[a; b; c]` | `[1; 2; 3]` |
| Массив | `[\| a; b; c \|]` | `[\|1; 2; 3\|]` |
| Discriminated Union | `type T = A \| B of int` | `type Shape = Circle of float \| Rect of float*float` |
| Record | `type R = { A: int; B: string }` | `type Person = { Name: string; Age: int }` |
| Pattern Match | `match x with \| P -> E` | `match opt with Some v -> v \| None -> 0` |
| Pipeline | `x \|> f` | `data \|> filter \|> map \|> sum` |
| Option | `Some v` / `None` | `Some 42` / `None` |
| Result | `Ok v` / `Error e` | `Ok "valid"` / `Error "invalid"` |
| Async | `async { let! x = f() ... }` | `async { let! data = fetchAsync() ... }` |
| Скрипт | `.fsx` файл | `dotnet fsi script.fsx` |

---

## Чек-лист: что попробовать в первую очередь

- [ ] Функции и pipeline `|>` — перепишите 3-5 операций цепочкой
- [ ] Discriminated Union — смоделируйте домен из 5+ вариантов
- [ ] Pattern Matching — exhaustive matching по DU (проверьте, что компилятор находит пропуски)
- [ ] Option — замените null на `Option.map`/`Option.bind`
- [ ] Result — цепочка операций с типизированными ошибками
- [ ] List/Array/Seq модули — `filter`, `map`, `fold`, `reduce`
- [ ] Рекурсия — факториал, обход дерева
- [ ] Async — `Async.Parallel` для параллельных запросов
- [ ] `.fsx` скрипт — автоматизируйте одну рутинную задачу
- [ ] F# Interactive — `dotnet fsi` для быстрой проверки идей
