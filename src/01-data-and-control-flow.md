# ГЛАВА 1. ДАННЫЕ И УПРАВЛЕНИЕ ПОТОКОМ

В прологе мы настроили окружение и запустили Hello World. Теперь пора написать что-то полезное.

Начнем с простой задачи: генератор паролей. По пути познакомимся с типами `Option` и `Result` — двумя столпами обработки ошибок в Rust. Затем изучим управление потоком: условия, циклы, сопоставление с образцом. И превратим простой генератор в интерактивное приложение.

---

## Первый проект: генератор паролей

Создаем проект:

```bash
cargo new passgen
cd passgen
```

### Зависимости

Для генерации случайных чисел используем крейт `rand`. Добавим зависимость в `Cargo.toml`:

```toml
[dependencies]
rand = "0.9.2"
```

Или через команду:

```bash
cargo add rand
```

### Код

`src/main.rs`:

```rust
use rand::Rng;

fn generate_password(length: usize) -> String {
    const CHARSET: &[u8] = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*";

    let mut rng = rand::rng();

    (0..length)
        .map(|_| {
            let idx = rng.random_range(0..CHARSET.len());
            CHARSET[idx] as char
        })
        .collect()
}

fn main() {
    let password = generate_password(16);
    println!("Generated password: {}", password);
}
```

Запуск:

```bash
cargo run
```

Вывод:

```
Generated password: n*EbG1KXg2CAiIBQ
```

### Разбор кода

**`use rand::Rng;`**
Импорт трейта `Rng` из крейта `rand`. Трейт — это набор методов, которые тип может реализовать. `Rng` предоставляет методы для генерации случайных чисел. Подробнее о трейтах — в Главе 3.


**`const CHARSET: &[u8]`**
Константа — срез байтов, содержащий допустимые символы пароля.
`b"..."` — байтовый строковый литерал, где каждый элемент имеет тип `u8`.


**`let mut rng = rand::rng();`**
Создание генератора случайных чисел.
`mut` необходим, так как генератор хранит внутреннее состояние и изменяется при каждом вызове. Это важный момент: даже "чтение" случайного числа — это изменение состояния генератора.


**`(0..length)`**
Полуоткрытый диапазон от `0` до `length` (верхняя граница не включается). Используется для задания количества генерируемых символов.


**`.map(|_| { ... })`**
Применение замыкания к каждому элементу диапазона.
`|_|` — замыкание с игнорируемым аргументом, так как значение индекса диапазона не используется. Подробнее о замыканиях и итераторах — в Главе 5.


**`rng.random_range(0..CHARSET.len())`**
Генерация равномерно распределенного случайного индекса в пределах длины массива `CHARSET`.


**`CHARSET[idx] as char`**
Получение байта по индексу и приведение его к `char`.
Корректно в данном случае, так как `CHARSET` содержит только ASCII-символы.


**`.collect()`**
Сбор последовательности `char` в коллекцию.
Тип `String` выводится компилятором из возвращаемого типа функции.

---

## Option и Result

Вышеописанная программа работает, но что если пользователь передаст длину 0? Или запросит чтение несуществующего файла? В большинстве языков такие ситуации обрабатываются через исключения или возврат null. Rust идет другим путем (алгебраические типы).

### Option — что-то или ничего

В Rust переменная не может быть «просто пустой». Чтобы выразить отсутствие значения, используется обертка `Option<T>`:

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 {
        Some(String::from("Lenie"))
    } else {
        None
    }
}

fn main() {
    let user = find_user(1);
    
    match user {
        Some(name) => println!("Найден: {}", name),
        None => println!("Не найден"),
    }
}
```

`Option<T>` имеет два варианта:
- `Some(value)` — значение присутствует
- `None` — значение отсутствует

Компилятор **заставляет** обработать оба случая. Забыть проверить на null не получится.

### Result — успех или провал

В отличие от `Option` (значение есть или нет), `Result<T, E>` говорит нам, **почему** операция не удалось:

```rust
use std::fs;

fn read_config() -> Result<String, std::io::Error> {
    fs::read_to_string("config.txt")
}

fn main() {
    match read_config() {
        Ok(content) => println!("Конфигурация: {}", content),
        Err(e) => println!("Ошибка: {}", e),
    }
}
```

`Result<T, E>`:
- `Ok(value)` — операция успешна
- `Err(error)` — произошла ошибка

### Оператор ?

Синтаксический сахар для пробрасывания ошибок:

```rust
use std::fs;
use std::io;

fn read_and_process() -> Result<(), io::Error> {
    let content = fs::read_to_string("data.txt")?;
    println!("Прочитано {} байт", content.len());
    Ok(())
}
```

`?` работает так:
- Если `Ok(value)` — извлекается `value`
- Если `Err(e)` — немедленно возвращается `Err(e)` из функции

Работает только в функциях, возвращающих `Result` или `Option`.

Сравни:

```rust
// Без оператора ?
fn read_file() -> Result<String, io::Error> {
    let content = match fs::read_to_string("file.txt") {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    Ok(content)
}

// С оператором ?
fn read_file() -> Result<String, io::Error> {
    let content = fs::read_to_string("file.txt")?;
    Ok(content)
}
```

### unwrap() и expect() — извлечение с паникой

Иногда нужно извлечь значение, зная, что ошибка невозможна (или ее наличие означает критический баг):

```rust
let config = std::fs::read_to_string("config.txt").unwrap();
```

**`unwrap()`** работает так:
- Если `Ok(value)` → возвращается `value`
- Если `Err(e)` → программа **паникует** (аварийно завершается)

**`expect(message)`** — то же самое, но с пояснением:

```rust
let config = std::fs::read_to_string("config.txt")
    .expect("Файл config.txt обязан существовать");
```

При панике выведет:
```
thread 'main' panicked at 'Файл config.txt обязан существовать: ...'
```

**Когда использовать:**

✓ **Прототипы и примеры** — когда обработка ошибок не важна:
```rust
let number = "42".parse::<i32>().unwrap();
```

✓ **Истинные инварианты** — когда ошибка означает критический баг в логике:
```rust
let mutex = Arc::new(Mutex::new(data));
let guard = mutex.lock().unwrap();  // Если fail — deadlock, это критический баг
```

✗ **Продакшен с внешними данными** — файлы, сеть, пользовательский ввод:
```rust
// ПЛОХО: файл может не существовать
let config = std::fs::read_to_string("user_data.json").unwrap();

// ХОРОШО: обработка ошибки
let config = std::fs::read_to_string("user_data.json")?;
```

---

> **История из жизни: unwrap(), который положил интернет**
>
> 18 ноября 2025 года Cloudflare пережила крупнейший сбой с 2019 года. X, ChatGPT, Canva и тысячи других сервисов стали недоступны для 2.4 миллиарда пользователей. Причина: один `unwrap()` в Rust-коде прокси-сервера FL2.
>
> Система Bot Management использовала конфигурационный файл с лимитом в 200 фич (обычно их было около 60). После изменения прав в базе данных файл внезапно вырос — и превысил лимит. Код проверил размер, получил ошибку... и вызвал `unwrap()`:
>
> ```
> thread fl2_worker_thread panicked: called Result::unwrap() on an Err value
> ```
>
> Паника распространилась на 330+ дата-центров по всему миру. Шесть часов даунтайма. Сотни миллионов долларов убытков.
>
> Мораль: `unwrap()` — это не "я уверен, что ошибки не будет". Это "если ошибка случится, пусть горит весь мир". В прототипах — допустимо. В продакшене — только для истинных инвариантов: ситуаций, когда ошибка означает баг в логике программы, а не проблему с внешними данными.
>
> (Cloudflare, к их чести, опубликовала полный постмортем с исходным кодом. Прозрачность, достойная уважения.)

---

### Улучшение генератора: параметризация

Добавим возможность указать длину пароля через аргумент командной строки:

```rust
use rand::Rng;
use std::env;

fn generate_password(length: usize) -> String {
    const CHARSET: &[u8] = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*";

    let mut rng = rand::rng();

    (0..length)
        .map(|_| {
            let idx = rng.random_range(0..CHARSET.len());
            CHARSET[idx] as char
        })
        .collect()
}

fn parse_length() -> Result<usize, String> {
    let args: Vec<String> = env::args().collect();

    if args.len() < 2 {
        return Ok(16);
    }

    args[1]
        .parse::<usize>()
        .map_err(|_| format!("Некорректная длина: {}", args[1]))
}

fn main() {
    match parse_length() {
        Ok(length) => {
            let password = generate_password(length);
            println!("Generated password: {}", password);
        }
        Err(e) => {
            eprintln!("Ошибка: {}", e);
            std::process::exit(1);
        }
    }
}
```

Запуск:

```bash
cargo run          # длина 16 по умолчанию
cargo run -- 24    # длина 24
```

**Что нового:**

**`env::args()`**
Возвращает итератор аргументов командной строки.

**`.collect()`**
Собирает итератор в `Vec<String>`.

**`.parse::<usize>()`**
Парсинг строки в число. Возвращает `Result<usize, ParseIntError>`.

**`.map_err(|_| ...)`**
Преобразование типа ошибки. `ParseIntError` → `String`.

**`eprintln!`**
Печать в stderr (стандартный поток ошибок).

**`std::process::exit(1)`**
Завершение программы с кодом возврата 1 (ошибка).

---

## Syntax Capsule

Генератор паролей работает, но он линейный: запустил → выполнил → завершил. Реальные программы принимают решения, повторяют действия, хранят коллекции данных. Научим нашу программу думать.

### Условные выражения: if/else

Базовая форма:

```rust
fn main() {
    let number = 7;
    
    if number < 5 {
        println!("Число меньше 5");
    } else if number < 10 {
        println!("Число меньше 10");
    } else {
        println!("Число больше или равно 10");
    }
}
```

**Ключевые особенности:**

1. Условие **не требует скобок**
2. Условие должно быть типа `bool` (не `0`, и не `1`)
3. `if` — это выражение, может возвращать значение:

```rust
let number = 5;
let result = if number > 0 {
    "положительное"
} else {
    "неположительное"
};
println!("{}", result);
```

Обе ветки должны возвращать один и тот же тип:

```rust
// Ошибка компиляции:
let x = if true { 5 } else { "hello" };

// Правильно:
let x = if true { 5 } else { 0 };
```

### Сопоставление с образцом: match

`match` — мощный инструмент для работы с перечислениями и паттернами:

```rust
fn describe_number(n: i32) {
    match n {
        0 => println!("ноль"),
        1 | 2 => println!("один или два"),
        3..=5 => println!("от трех до пяти"),
        _ => println!("что-то другое"),
    }
}
```

**Ключевые моменты:**

1. **Exhaustive matching** — все варианты должны быть покрыты
2. `_` — универсальный паттерн (catch-all)
3. `|` — альтернатива (или)
4. `..=` — инклюзивный диапазон

`match` тоже выражение:

```rust
let result = match value {
    0 => "ноль",
    1 => "один",
    _ => "много",
};
```

Деструктуризация кортежей:

```rust
let point = (3, 5);

match point {
    (0, 0) => println!("Начало координат"),
    (x, 0) => println!("На оси X: {}", x),
    (0, y) => println!("На оси Y: {}", y),
    (x, y) => println!("Точка ({}, {})", x, y),
}
```

Работа с `Option` и `Result`:

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

fn main() {
    let result = divide(10.0, 2.0);
    
    match result {
        Some(value) => println!("Результат: {}", value),
        None => println!("Деление на ноль"),
    }
}
```

Guard-выражения:

```rust
let number = Some(4);

match number {
    Some(x) if x < 5 => println!("Меньше пяти: {}", x),
    Some(x) => println!("Больше или равно пяти: {}", x),
    None => println!("Ничего"),
}
```

### Циклы: loop, while, for

#### loop — бесконечный цикл

```rust
let mut count = 0;

loop {
    count += 1;
    println!("{}", count);
    
    if count == 5 {
        break;
    }
}
```

`loop` может возвращать значение через `break`:

```rust
let mut counter = 0;

let result = loop {
    counter += 1;
    
    if counter == 10 {
        break counter * 2;
    }
};

println!("Результат: {}", result);  // 20
```

Метки для вложенных циклов:

```rust
'outer: loop {
    println!("Внешний цикл");
    
    loop {
        println!("Внутренний цикл");
        break 'outer;  // выход из внешнего цикла
    }
}
```

#### while — цикл с условием

```rust
let mut number = 3;

while number != 0 {
    println!("{}!", number);
    number -= 1;
}

println!("Пуск!");
```

#### for — итерация по коллекции

```rust
let a = [10, 20, 30, 40, 50];

for element in a {
    println!("значение: {}", element);
}
```

Диапазоны:

```rust
// 1..5 -> 1, 2, 3, 4 (без 5)
for n in 1..5 {
    println!("{}", n);
}

// 1..=5 -> 1, 2, 3, 4, 5 (включая 5)
for n in 1..=5 {
    println!("{}", n);
}
```

С индексами:

```rust
let names = ["Alice", "Bob", "Carol"];

for (index, name) in names.iter().enumerate() {
    println!("{}: {}", index, name);
}
```

Обратный порядок:

```rust
for n in (1..5).rev() {
    println!("{}", n);
}
```

### Массивы — фиксированный размер

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
let first = a[0];
let second = a[1];

// Все элементы одинаковые:
let zeros = [0; 100];  // [0, 0, 0, ..., 0]
```

Массивы:
- Фиксированный размер (часть типа)
- Данные на стеке
- Быстрый доступ по индексу

### Vec — динамический массив

```rust
let mut v: Vec<i32> = Vec::new();

v.push(1);
v.push(2);
v.push(3);

println!("{:?}", v);  // [1, 2, 3]
```

Макрос `vec!`:

```rust
let v = vec![1, 2, 3, 4, 5];
```

Доступ к элементам:

```rust
let v = vec![1, 2, 3, 4, 5];

// Метод 1: паника при выходе за границы
let third = v[2];

// Метод 2: безопасный доступ
match v.get(2) {
    Some(value) => println!("Третий элемент: {}", value),
    None => println!("Нет третьего элемента"),
}
```

Итерация:

```rust
let v = vec![10, 20, 30];

for value in &v {
    println!("{}", value);
}

// Изменение элементов:
let mut v = vec![10, 20, 30];

for value in &mut v {
    *value += 50;
}

println!("{:?}", v);  // [60, 70, 80]
```

Полезные методы:

```rust
let mut v = vec![1, 2, 3];

v.push(4);              // добавить в конец
let last = v.pop();     // извлечь из конца (Option<T>)
let len = v.len();      // длина
let is_empty = v.is_empty();  // пустой ли

v.clear();              // очистить
```

---

## Проект: Интерактивный генератор паролей

Превратим генератор в интерактивное приложение с простым меню.

**Требования:**
- Меню: Генерировать / Изменить длину / Выход
- Цикл работы до явного выхода

```bash
cargo new passgen_interactive
cd passgen_interactive
cargo add rand
```

### Реализация

`src/main.rs`:

```rust
use rand::Rng;
use std::io::{self, Write};

fn generate_password(
    length: usize,
    use_upper: bool,
    use_lower: bool,
    use_digits: bool,
    use_special: bool,
) -> Option<String> {
    let mut charset = Vec::new();

    if use_upper {
        charset.extend_from_slice(b"ABCDEFGHIJKLMNOPQRSTUVWXYZ");
    }
    if use_lower {
        charset.extend_from_slice(b"abcdefghijklmnopqrstuvwxyz");
    }
    if use_digits {
        charset.extend_from_slice(b"0123456789");
    }
    if use_special {
        charset.extend_from_slice(b"!@#$%^&*");
    }

    if charset.is_empty() {
        return None;
    }

    let mut rng = rand::rng();

    let password: String = (0..length)
        .map(|_| {
            let idx = rng.random_range(0..charset.len());
            charset[idx] as char
        })
        .collect();

    Some(password)
}

fn read_line() -> String {
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("Ошибка чтения ввода");
    input.trim().to_string()
}

fn print_menu() {
    println!("\n=== ГЕНЕРАТОР ПАРОЛЕЙ ===");
    println!("1. Сгенерировать пароль");
    println!("2. Изменить длину");
    println!("3. Выход");
    print!("Выбор: ");
    io::stdout().flush().unwrap();
}

fn main() {
    // Настройки как отдельные переменные
    let mut length = 16;
    let use_upper = true;
    let use_lower = true;
    let use_digits = true;
    let use_special = true;

    loop {
        print_menu();
        let choice = read_line();

        match choice.as_str() {
            "1" => {
                match generate_password(length, use_upper, use_lower, use_digits, use_special) {
                    Some(password) => {
                        println!("\nПароль: {}", password);
                        println!("Длина: {} символов", password.len());
                    }
                    None => {
                        println!("Ошибка: нет доступных символов");
                    }
                }
            }
            "2" => {
                print!("Новая длина (4-128): ");
                io::stdout().flush().unwrap();
                let input = read_line();

                match input.parse::<usize>() {
                    Ok(new_len) if new_len >= 4 && new_len <= 128 => {
                        length = new_len;
                        println!("Длина изменена на {}", length);
                    }
                    Ok(_) => {
                        println!("Длина должна быть от 4 до 128");
                    }
                    Err(_) => {
                        println!("Некорректное число");
                    }
                }
            }
            "3" => {
                println!("¡Adiós!");
                break;
            }
            _ => {
                println!("Некорректный выбор");
            }
        }
    }
}
```

### Разбор кода

#### Передача параметров

```rust
fn generate_password(
    length: usize,
    use_upper: bool,
    use_lower: bool,
    use_digits: bool,
    use_special: bool,
) -> Option<String> {
```

Пять параметров — это неудобно. Каждый раз при вызове нужно передавать все:

```rust
generate_password(length, use_upper, use_lower, use_digits, use_special)
```

Выглядит громоздко? Так и есть. В Главе 3 мы научимся упаковывать связанные данные в структуры.

#### Построение charset:

```rust
let mut charset = Vec::new();

if use_upper {
    charset.extend_from_slice(b"ABCDEFGHIJKLMNOPQRSTUVWXYZ");
}
```

Создаем пустой вектор и добавляем символы в зависимости от флагов. `extend_from_slice` добавляет все элементы среза в конец вектора.

#### Чтение ввода:

```rust
fn read_line() -> String {
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("Ошибка чтения ввода");
    input.trim().to_string()
}
```

`read_line` добавляет символ новой строки — `trim()` убирает его.

#### Главный цикл:

```rust
loop {
    print_menu();
    let choice = read_line();

    match choice.as_str() {
        "1" => { /* генерация */ }
        "2" => { /* изменение длины */ }
        "3" => break,
        _ => println!("Некорректный выбор"),
    }
}
```

Бесконечный цикл, прерываемый при выборе "3". `match` обрабатывает все варианты ввода. Паттерн `_` ловит все, что не подошло под предыдущие варианты.

#### Guard в match:

```rust
match input.parse::<usize>() {
    Ok(new_len) if new_len >= 4 && new_len <= 128 => {
        length = new_len;
    }
    Ok(_) => {
        println!("Длина должна быть от 4 до 128");
    }
    Err(_) => {
        println!("Некорректное число");
    }
}
```

`if new_len >= 4 && new_len <= 128` — это guard, дополнительное условие для паттерна. Позволяет разделить `Ok` на два случая: валидное и невалидное значение.

---

## Проект: Утилита wordcount

Напишем полноценную CLI-утилиту, аналог Unix-команды `wc` (word count).

**Функционал:**
- Подсчет строк, слов, символов, байтов
- Обработка нескольких файлов
- Флаги для выбора типа подсчета
- Итоговая строка при обработке нескольких файлов

```bash
cargo new wordcount
cd wordcount
```

### Реализация

`src/main.rs`:

```rust
use std::env;
use std::fs;

fn count_stats(content: &str) -> (usize, usize, usize, usize) {
    let lines = content.lines().count();
    let words = content.split_whitespace().count();
    let chars = content.chars().count();
    let bytes = content.len();

    (lines, words, chars, bytes)
}

fn print_stats(
    lines: usize,
    words: usize,
    chars: usize,
    bytes: usize,
    show_lines: bool,
    show_words: bool,
    show_chars: bool,
    show_bytes: bool,
    filename: &str,
) {
    if show_lines {
        print!("{:>8}", lines);
    }
    if show_words {
        print!("{:>8}", words);
    }
    if show_chars {
        print!("{:>8}", chars);
    }
    if show_bytes {
        print!("{:>8}", bytes);
    }
    if !filename.is_empty() {
        print!(" {}", filename);
    }
    println!();
}

fn main() {
    let args: Vec<String> = env::args().skip(1).collect();

    // Флаги как отдельные переменные
    let mut show_lines = false;
    let mut show_words = false;
    let mut show_chars = false;
    let mut show_bytes = false;
    let mut files: Vec<String> = Vec::new();

    // Парсинг аргументов
    for arg in &args {
        if arg.starts_with('-') && arg.len() > 1 {
            for c in arg[1..].chars() {
                if c == 'l' {
                    show_lines = true;
                } else if c == 'w' {
                    show_words = true;
                } else if c == 'c' {
                    show_bytes = true;
                } else if c == 'm' {
                    show_chars = true;
                } else {
                    eprintln!("wordcount: неизвестный флаг: -{}", c);
                }
            }
        } else {
            files.push(arg.clone());
        }
    }

    // Если флаги не указаны — включаем по умолчанию
    if !show_lines && !show_words && !show_chars && !show_bytes {
        show_lines = true;
        show_words = true;
        show_bytes = true;
    }

    // Обработка файлов
    let mut total_lines = 0;
    let mut total_words = 0;
    let mut total_chars = 0;
    let mut total_bytes = 0;
    let mut file_count = 0;

    for path in &files {
        match fs::read_to_string(path) {
            Ok(content) => {
                let (lines, words, chars, bytes) = count_stats(&content);

                print_stats(
                    lines, words, chars, bytes,
                    show_lines, show_words, show_chars, show_bytes,
                    path,
                );

                total_lines += lines;
                total_words += words;
                total_chars += chars;
                total_bytes += bytes;
                file_count += 1;
            }
            Err(e) => {
                eprintln!("wordcount: {}: {}", path, e);
            }
        }
    }

    // Итоговая строка
    if file_count > 1 {
        print_stats(
            total_lines, total_words, total_chars, total_bytes,
            show_lines, show_words, show_chars, show_bytes,
            "total",
        );
    }

    // Если файлы не указаны
    if files.is_empty() {
        eprintln!("Использование: wordcount [-lwcm] <файлы...>");
        eprintln!("Флаги: -l (строки), -w (слова), -c (байты), -m (символы)");
    }
}
```

### Разбор кода

#### Подсчет статистики

```rust
fn count_stats(content: &str) -> (usize, usize, usize, usize) {
    let lines = content.lines().count();
    let words = content.split_whitespace().count();
    let chars = content.chars().count();
    let bytes = content.len();

    (lines, words, chars, bytes)
}
```

Функция возвращает кортеж из четырех значений. Это временное решение — в Главе 3 мы научимся создавать структуры для таких случаев.


`.lines()` — итератор по строкам.

`.split_whitespace()` — итератор по словам (разделитель — любой пробельный символ).

`.chars()` — итератор по Unicode-символам.

`.len()` — количество байт (не символов!).

#### Парсинг аргументов:

```rust
for arg in &args {
    if arg.starts_with('-') && arg.len() > 1 {
        for c in arg[1..].chars() {
            if c == 'l' {
                show_lines = true;
            } else if c == 'w' {
                show_words = true;
            }
            // ...
        }
    } else {
        files.push(arg.clone());
    }
}
```

`arg[1..]` — срез строки, все символы кроме первого (убираем дефис).
`.chars()` — итератор по символам, позволяет обрабатывать `-lwc` как три отдельных флага.


#### Обработка ошибок:

```rust
match fs::read_to_string(path) {
    Ok(content) => {
        // обработка
    }
    Err(e) => {
        eprintln!("wordcount: {}: {}", path, e);
    }
}
```

При ошибке чтения файла выводим сообщение в stderr, но продолжаем обработку остальных файлов. Это стандартное поведение Unix-утилит.

### Тестирование

Создадим тестовые файлы:

```bash
echo "Hello world" > test1.txt
echo -e "Line one\nLine two\nLine three" > test2.txt
```

Запуск:

```bash
cargo run -- test1.txt
cargo run -- -l test1.txt              # только строки
cargo run -- -wl test1.txt             # слова и строки
cargo run -- test1.txt test2.txt       # несколько файлов
```

### Unit-тесты

Добавим модуль тестов в конец файла:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_empty_string() {
        let (lines, words, chars, bytes) = count_stats("");
        assert_eq!(lines, 0);
        assert_eq!(words, 0);
        assert_eq!(bytes, 0);
    }

    #[test]
    fn test_single_line() {
        let (lines, words, _, bytes) = count_stats("hello world");
        assert_eq!(lines, 1);
        assert_eq!(words, 2);
        assert_eq!(bytes, 11);
    }

    #[test]
    fn test_multiple_lines() {
        let (lines, words, _, _) = count_stats("line one\nline two\nline three");
        assert_eq!(lines, 3);
        assert_eq!(words, 6);
    }

    #[test]
    fn test_unicode() {
        let (_, words, chars, bytes) = count_stats("Привет мир");
        assert_eq!(words, 2);
        assert_eq!(chars, 10);
        assert_eq!(bytes, 19);  // кириллица: 2 байта на символ
    }

    #[test]
    fn test_whitespace_handling() {
        let (_, words, _, _) = count_stats("  много    пробелов   ");
        assert_eq!(words, 2);
    }
}
```

Запуск тестов:

```bash
cargo test
```
А теперь разберем магические заклинания, которые мы только что написали:

1.  **`#[cfg(test)]`**
    Это атрибут условной компиляции. Он говорит Rust: "Весь модуль `tests` нужно компилировать и включать в бинарник **только** когда я запускаю `cargo test`". В обычном билде (`cargo build`) этот код просто исчезнет и не будет занимать место.

2.  **`mod tests { ... }`**
    В Rust тесты принято писать в том же файле, что и код, но изолировать их в отдельный модуль. Это позволяет тестировать приватные функции, не загрязняя основное пространство имен.

3.  **`use super::*;`**
    Поскольку тесты живут во вложенном модуле (`tests`), они не видят функции из родительского модуля (`main`). Эта строка импортирует всё (`*`) из родительского модуля (`super`) в область видимости тестов.

4.  **`#[test]`**
    Помечает функцию как тест. `cargo test` ищет именно эти пометки.

5.  **`assert_eq!(left, right)`**
    Макрос утверждения. Проверяет, что левый аргумент равен правому. Если нет — паникует и выводит оба значения, что и считается провалом теста.

---

## Задания

Мы написали прототипы двух программ (MVP). Твоя задача — превратить их в полноценные инструменты, которыми удобно пользоваться. Выбери уровень сложности.

### Уровень 1: Стажер

**Задача 1.1: Справка для Wordcount**

Добавь обработку флага `-h` или `--help`. Если пользователь запускает `wordcount --help`, программа должна вывести инструкцию по использованию и завершиться.

**Задача 1.2: Безопасный ввод в PassGen**

В интерактивном генераторе мы используем `.expect()` при чтении ввода. Что будет, если stdin закроется? Замени `expect` на обработку `Result` — при ошибке выведи сообщение и заверши программу корректно.

### Уровень 2: Разработчик

**Задача 2.1: Оценка силы пароля**

Добавь в `PassGen` функцию, которая оценивает сгенерированный пароль:
- < 8 символов → "Слабый"
- ≥ 8 символов, только буквы → "Средний"  
- ≥ 8 символов, буквы + цифры → "Хороший"
- ≥ 12 символов, буквы + цифры + спецсимволы → "Сильный"

Выводи оценку после генерации.

**Задача 2.2: Поиск самого длинного слова**

Добавь в `wordcount` флаг `-M` (max). При его наличии программа должна не считать статистику, а найти и вывести самое длинное слово в файле.

*Подсказка:* используй переменную `let mut longest = "";` и цикл по `content.split_whitespace()`.

### Уровень 3: Инженер

**Задача 3.1: Буфер обмена**

Сделай так, чтобы `PassGen` предлагал скопировать пароль в буфер обмена.

*Челлендж:* Rust из коробки не работает с буфером обмена. Зайди на [crates.io](https://crates.io), найди подходящий крейт (например, `arboard` или `cli-clipboard`), добавь через `cargo add` и разберись с документацией.

**Задача 3.2: Чтение из stdin**

Настоящая утилита `wc` читает из stdin, если файлы не указаны:

```bash
echo "Hello Rust" | wc
```

Реализуй такое же поведение. Если список файлов пуст, читай данные из `std::io::stdin()`.

*Подсказка:* `io::stdin().read_to_string(&mut buffer)` работает аналогично чтению файла.

---

## Бонус: Weasel Program

В 1986 году Ричард Докинз опубликовал книгу «Слепой часовщик» — ответ на аргумент о невероятности случайного возникновения сложных структур. Креационисты утверждали: вероятность случайного появления функционального белка или осмысленного текста настолько мала, что требует разумного дизайнера.

Докинз предложил элегантную демонстрацию. Возьмем фразу из Шекспира:

```
METHINKS IT IS LIKE A WEASEL
```

Вероятность набрать эту строку случайным образом — около 1 к 10^40. Даже если каждый атом во Вселенной генерировал бы по миллиону строк в секунду с момента Большого взрыва, мы бы не успели.

Но эволюция работает иначе. Она не бросает кости заново каждый раз — она **накапливает** удачные изменения.

Алгоритм Докинза:
1. Начать со случайной строки той же длины
2. Создать несколько копий с небольшими мутациями
3. Выбрать копию, наиболее похожую на цель
4. Повторять, пока строка не совпадет с целью

Результат: вместо 10^40 попыток достаточно нескольких сотен поколений. Кумулятивный отбор превращает астрономически невероятное в неизбежное.

Weasel Program — идеальная площадка для практики с конструкциями этой главы: `loop` для поколений, `Vec` для популяции, `if` для подсчета совпадений, циклы для мутаций и отбора.

`→ Приложение F.1: Weasel Program — полная реализация с разбором`

---

## ✓ Фильтр качества

Ты освоил эту главу, если можешь:

- [ ] Объяснить разницу между `Option` и `Result`
- [ ] Использовать оператор `?` для пробрасывания ошибок
- [ ] Объяснить, когда уместен `unwrap()`, а когда нет
- [ ] Понимать, почему генератор случайных чисел требует `mut`
- [ ] Написать `if`/`else` выражение, возвращающее значение
- [ ] Написать `match` для `Option` без подглядывания
- [ ] Использовать guard-выражения в `match`
- [ ] Объяснить разницу между `loop`, `while` и `for`
- [ ] Написать цикл `for` с индексами (`enumerate`)
- [ ] Использовать `break` с меткой для выхода из вложенного цикла
- [ ] Создать `Vec`, добавить элементы, проитерировать
- [ ] Обработать аргументы командной строки через `env::args()`
- [ ] Прочитать файл с обработкой ошибки
- [ ] Написать тест с `#[test]` и `assert_eq!`

**Если хотя бы одна галочка не стоит — вернись к соответствующему разделу.**

---

## Что дальше?

Ты написал приложение с текстовым меню и классическую системную утилиту. Научился обрабатывать ошибки, управлять потоком выполнения, работать с коллекциями.

Но код получился громоздким. Функция `generate_password` принимает пять параметров. `print_stats` — девять. Это неудобно и ненадежно: легко перепутать порядок аргументов, сложно добавлять новые.

**Глава 2** покажет, как Rust управляет памятью без сборщика мусора. Почему одни данные копируются, а другие перемещаются? Что такое владение и заимствование?

**Глава 3** даст инструмент для борьбы с хаосом параметров: структуры, перечисления и трейты. Мы вернемся к этим проектам и сделаем код чистым.

---
