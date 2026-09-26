# valk-csv

CSV for [Valk](https://valk-lang.dev): reading and writing RFC 4180, row by row from a file of any
size, and straight into and out of your own classes. Purely written in Valk, with no os-package
dependencies.

Requires Valk 0.7.0 or newer.

## Install

```
vman install github.com/ctxcode/valk-csv
```

## Example

```rust
use csv
use valk.fs

class Product {
    sku: String
    name: String
    price: f64
    stock: uint (0)
    note: ?String
}

let text = fs.read("products.csv") ! panic("Cannot read the file")

// Rows of fields
let rows = csv.parse(text) ! panic("%{E.message} at line %{E.line}")
println(rows[1][0])

// Or straight into a class, by the names in the header row
let products = csv.decode_to[Product](text) ! panic("%{E.message} at line %{E.line}")
println(products[0].price)

// And back out, with a header row
fs.write("products.csv", csv.encode_of(products)) ! panic("Cannot write the file")
```

## Big files

A `Reader` holds one chunk of the input and the row it is reading, so a file of any size fits.
It reads from a file, from text, or from any `io.Reader`, such as a socket or `io.stdin()`:

```rust
let reader = csv.Reader.open("orders.csv", csv.Options { header: true }) ! panic("%{E.message}")
while true {
    let row = reader.next_map() ! panic("%{E.message} at line %{E.line}")
    if !isset(row) : break
    println(row.get("email") !? "")
}
```

- `next()` returns the row as an `Array[String]`, `next_map()` as a map from column name to
  field, and `next_to[Product]()` as an object of your class. All three return `null` after the
  last row.
- With `header: true` the first row is the header: `reader.header()` returns it, and
  `reader.column("email")` gives the index of a column for `row[index]`.
- `reader.line` is the line the last row started on, for messages of your own.
- `csv.Reader.new(source, options, chunk_size)` reads any `io.Reader`, and
  `csv.Reader.from_string(text)` reads text without copying it.

`csv.parse_maps(text)` reads a whole text into maps by the header, the way `csv.parse` reads it
into rows.

## Classes

`decode_to[T]` and `next_to[T]` read each field of the class from the column with its name;
columns without a field are ignored. The text converts to the type of the field:

- `String`, every integer type and `f32`/`f64`. Numbers may have spaces around them.
- `bool` from `true`/`false`, `yes`/`no` or `1`/`0`, in any case.
- A nullable one of those: an empty field is `null`.
- A field with a default gets the default when its column is missing or the field is empty.
  Any other field must have a column, and a number or bool that does not read throws `type` with
  the line and column of the field.

`encode_of(items)` and `Writer.write_header_of[T]()` / `write_object(item)` write the fields in the
order they are declared, numbers in their shortest exact form, bools as `true`/`false` and `null`
as an empty field. Enums and nested classes are refused when the program is built.

## Writing

```rust
let file = fs.stream("out.csv", fs.OpenOptions { read: false, write: fs.WriteMode.truncate, create: true }) ! panic("open")
let writer = csv.Writer.new(file)
writer.write_row(.{ "name", "age" }) ! panic("%{E.message}")
writer.write_row(.{ "ada", 36 }) ! panic("%{E.message}")
writer.flush() ! panic("%{E.message}")
```

Rows collect in the writer and go out in large writes, so call `flush()` after the last one; a
`ByteBuffer` is written into directly. `csv.encode(rows)` returns the text at once.

A field is quoted only when it has to be: when it holds the delimiter, the quote, `\r` or `\n`,
or starts or ends with a space or a tab, which a reader that trims would lose. A quote inside a
quoted field is written twice. A row of one empty field is written as `""`, so it does not turn
into an empty line, and with a `comment` byte set a first field that starts with it is quoted.
Apart from the spaces at the ends of a field this is the rule of Python's `csv` module.

## Options

```rust
let options = csv.Options { delimiter: ';', header: true, trim: true }
```

| Option | Default | |
|---|---|---|
| `delimiter` | `,` | The byte between fields: `;`, `\t`, `\|`, ... |
| `quote` | `"` | The byte that quotes a field. |
| `header` | `false` | Reading: the first row is the header. |
| `trim` | `false` | Reading: removes spaces and tabs around unquoted fields, and allows them around quoted ones. |
| `skip_empty_lines` | `true` | Reading: when false, an empty line is a row of one empty field. |
| `comment` | `0` | Reading: skips lines that start with this byte, such as `#`. |
| `strict` | `false` | Reading: every row must have as many fields as the first, and quotes must follow RFC 4180. |
| `line_ending` | `"\r\n"` | Writing: the text after every row. |
| `quote_all` | `false` | Writing: quotes every field. |
| `bom` | `false` | Writing: starts with a UTF-8 byte order mark, which Excel needs. |

Options that cannot work, such as a delimiter that is also the quote, panic when the reader or
writer is made: that is a mistake in the program, not in the data.

## What it reads

Quoted fields with the delimiter, the quote written twice, and line breaks inside them; `\r\n`,
`\n` and a lone `\r` as line ends, mixed as they like; a last line with or without a line end; a
UTF-8 byte order mark at the start; empty fields and empty lines. A quoted field may span any
number of lines and chunk boundaries.

Outside strict mode it reads broken quoting the way Python's `csv` module does: a quote inside an
unquoted field is text, and text after a closing quote joins the field. A quoted field that is
never closed is always an error, where Python returns the rest of the file as its last field.

Not supported: escape characters other than the doubled quote, multi-byte delimiters, and
guessing the dialect of a file.

## Errors

`syntax` (quotes that break the rules), `fields` (a row of another length in strict mode),
`header` (a missing header or column), `type` (a field that does not convert) and `io` (the source
or destination failed), each with the `line` and `column` where it happened. Columns count
characters, not bytes.

```rust
csv.parse(text, csv.Options { strict: true }) ! {
    println(E.message + " at line " + E.line + ", column " + E.column)
    return
}
// This row has 2 fields, where the first row has 3 at line 4, column 1
```

## Development

`make test` runs the suite, `make example` runs the example, `make lint` checks the sources and
`make docs` regenerates the API documentation. `python3 tests/crosscheck.py` regenerates
`tests/python-samples.valk`, the samples read and written by Python's `csv` module.
