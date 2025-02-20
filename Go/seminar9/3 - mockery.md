- [mockery](#mockery)
  - [Простейшие моки](#простейшие-моки)
  - [codegen](#codegen)

# mockery

## Простейшие моки

Моки имитируют для тестов работу реальной зависимости (заданной как интерфейс). 

Простой пример использования мока -- уже рассмотренный [пример](./examples/part2/myjson). В этом примере, параметр
функции `ParseAndSortSlice` определяется как интерфейс `io.Reader`:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

Скорее всего, в реальном коде в качестве источника данных (`input`) будет использоваться файл или сетевое соединение.

В тестах в качестве этого параметра использовались `strings.NewReader("...")` и `iotest.ErrReader(someErr)`. 

```go
type Reader struct {
    s        string
    i        int64 // current reading index
    prevRune int   // index of previous rune; or < 0
}

...

// Read implements the [io.Reader] interface.
func (r *Reader) Read(b []byte) (n int, err error) {
    if r.i >= int64(len(r.s)) {
        return 0, io.EOF
    }
    r.prevRune = -1
    n = copy(b, r.s[r.i:])
    r.i += int64(n)
    return
}

...
```

Т.е., в данном случае, тип `*strings.Reader` удерживает в себе переданную строку и копирует её байты в байтовый буфер, 
переданный в функцию-ресивер `Read`. В тесте это использовалось, как способ указать, что из источника данных
будут **успешно** прочитаны указанные байты.

Вместо `strings.NewReader(...)` в качестве `io.Reader` можно использовать `bytes.NewBuffer(...)`, который принимает
в качестве входного параметра слайс байт, а не строку (в остальном принцип работы тот же).

```go
// ErrReader returns an [io.Reader] that returns 0, err from all Read calls.
func ErrReader(err error) io.Reader {
    return &errReader{err: err}
}

type errReader struct {
    err error
}

func (r *errReader) Read(p []byte) (int, error) {
    return 0, r.err
}
```

Тип `*iotest.errReader` работает по-другому: он удерживает в себе переданную ошибку, и использует её в функции-ресивере `Read`.
В тесте это использовалось, как способ указать, что из источника данных **не получилось** прочитать байты (произошла ошибка).

`*strings.Reader` (или `*bytes.Buffer`) и `*iotest.errReader` для этих тестов являются простейшими моками -- они имитируют
нужным образом как успешную, так и неуспешную работу зависимости (`input io.Reader`).

## codegen

Рассмотренные на данный момент простейшие моки обладают недостатком: нужен отдельный код, который будет имитировать 
поведение зависимости нужным образом. Для простой зависимости типа `io.Reader` с единственной функцией-ресивером
`Read(p []byte) (n int, err error)` потребовалось 2 разных мока, чтобы имитировать успешное поведение и ошибку.

Зависимости могут иметь другой вид, и кода, который имитирует нужное поведение, может просто не найтись ни в стандартной
библиотеке, ни в сторонних пакетах. 

Утилита `mockery` позволяет генерировать вспомогательные структуры для тестов, которые позволяют очень гибко 
имитировать любое поведение зависимости. Кроме этого, можно выполнять дополнительные проверки, например, 
что функция-ресивер зависимости будет вызвана с конкретными параметрами нужное количество раз.

[Пример использования mockery](./examples/part3/hostrole)

Генерация мок-структуры для зависимости `PatroniClient` осуществляется вызовом
`mockery --name PatroniClient --structname MockPatroniClient --filename mock_patroni_client_test.go --outpkg hostRole_test --output .`.

Параметры подробнее:

- `--name PatroniClient` -- имя интерфейса, для которого нужно сгенерировать мок-структуру
- `--structname MockPatroniClient` -- имя сгенерированной мок-структуры
- `--filename mock_patroni_client_test.go` -- имя файла, в котором будет сгенерированная мок-структура
- `--outpkg hostRole_test` -- имя пакета
- `--output .` -- директория, в которой должен быть создан файл

Мок-структуры можно создавать в отдельном пакете (`mocks`) или в общем пакете с тестами. Рекомендуется
придерживаться следующего формата для имени файлов: `mock_..._test.go`:

- суффикс `_test` требуется, т.к. код моков относится к коду тестов, а не к основному коду приложения
- префикс `mock_` -- способ отличать сгенерированный код от кода тестов
