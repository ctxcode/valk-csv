
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when CSV cannot be read, or cannot be written to its destination.
+ error Error (syntax, fields, header, type, io) payload { message: String, line: uint (0), column: uint (0) }
```

### Error

Thrown when CSV cannot be read, or cannot be written to its destination.

- `syntax`: the quotes do not follow the rules. A quoted field that is never closed is
  always an error; in strict mode so is a quote inside an unquoted field, and text between a
  closing quote and the delimiter.
- `fields`: in strict mode, a row has another number of fields than the first row.
- `header`: a header is needed and there is none, a column is looked up by a name the header
  does not have, or a field of the class being read has no column.
- `type`: a field does not convert to the type of the class field it is read into.
- `io`: reading the source or writing the destination failed.

`line` and `column` say where, counting from one; the column counts characters, not bytes.
Both are 0 when the error has no place in the text, such as a failed write.

## Functions for 'main'

```js
// Reads CSV text whose first row is a header into objects of your own class or struct.
+ fn decode_to[T](text: String, options: ?Options (null)) Array[T] !Error
// Writes rows as CSV text.
+ fn encode(rows: Array[Array[String]], options: ?Options (null)) String
// Writes objects of a class or struct as CSV text: a header row with the names of the fields, then one row per object, which `decode_to` reads back.
+ fn encode_of(items: Array[$T], options: ?Options (null)) String
// Reads CSV text into rows of fields.
+ fn parse(text: String, options: ?Options (null)) Array[Array[String]] !Error
// Reads CSV text whose first row is a header into one map per row, from column name to field.
+ fn parse_maps(text: String, options: ?Options (null)) Array[Map[String]] !Error
```

### decode_to

Reads CSV text whose first row is a header into objects of your own class or struct.

Each field is read from the column with its name; columns without a field are ignored. The
text converts to the type of the field: numbers, `true`/`false` (also `yes`/`no` and `1`/`0`,
in any case) and strings, or a nullable one of those. An empty field gives `null` for a
nullable field and the default for a field that has one. A field that is neither nullable nor
has a default must have a column, and every number and bool must read.

```valk
class User {
    name: String
    age: uint
    email: ?String
}

let users = csv.decode_to[User](text) ! panic("%{E.message} at line %{E.line}")
```

### encode

Writes rows as CSV text.

```valk
let text = csv.encode(.{ .{ "name", "note" }, .{ "ada", "says \"hi\", twice" } })
// name,note\r\nada,"says ""hi"", twice"\r\n
```

### encode_of

Writes objects of a class or struct as CSV text: a header row with the names of the fields,
then one row per object, which `decode_to` reads back.

Numbers and bools are written as text, and a `null` as an empty field.

### parse

Reads CSV text into rows of fields.

Every row is returned, the first one too: the `header` option is for a `Reader`, while
`parse_maps` and `decode_to` always read a header.

```valk
let rows = csv.parse("name,age\nada,36\n") ! panic("%{E.message} at line %{E.line}")
println(rows[1][0]) // ada
```

### parse_maps

Reads CSV text whose first row is a header into one map per row, from column name to field.

A row shorter than the header leaves its missing columns out of its map.

```valk
let users = csv.parse_maps("name,age\nada,36\n") ! panic("%{E.message}")
println(users[0].get("age") !? "") // 36
```

## Classes for 'main'

```js
// How CSV is read and written: the dialect.
+ class Options {
    // Writing: starts the output with a UTF-8 byte order mark, which Excel needs to read the text as UTF-8.
    + bom: bool
    // Reading: skips lines that start with this byte, such as `#`. 0 turns comments off.
    + comment: u8
    // The byte between fields: `,` by default, `;` or `\t` for other dialects.
    + delimiter: u8
    // Reading: whether the first row is the header. A `Reader` then keeps it in `header` instead of returning it, and can hand out rows by column name.
    + header: bool
    // Writing: the text after every row. RFC 4180 asks for `\r\n`; `\n` is the other common choice.
    + line_ending: String
    // The byte that quotes a field. Inside a quoted field it is written twice.
    + quote: u8
    // Writing: quotes every field, not only the ones that need it.
    + quote_all: bool
    // Reading: skips lines that are empty. When false an empty line is a row with one empty field.
    + skip_empty_lines: bool
    // Reading: every row must have as many fields as the first row, and quotes must follow RFC 4180 exactly.
    + strict: bool
    // Reading: removes spaces and tabs around unquoted fields, and allows them around quoted fields.
    + trim: bool
}
```

### Options

How CSV is read and written: the dialect.

Every field has a default, so only what differs has to be named:

```valk
let options = csv.Options { delimiter: ';', header: true }
```

The same options can be given to a `Reader` and a `Writer`, so a file is written the way it
is read. Options that make no sense, such as a delimiter that is also the quote, panic when
the reader or writer is made.

#### bom

Writing: starts the output with a UTF-8 byte order mark, which Excel needs to read the
text as UTF-8.

#### comment

Reading: skips lines that start with this byte, such as `#`. 0 turns comments off.

#### delimiter

The byte between fields: `,` by default, `;` or `\t` for other dialects.

#### header

Reading: whether the first row is the header. A `Reader` then keeps it in `header` instead
of returning it, and can hand out rows by column name.

#### line_ending

Writing: the text after every row. RFC 4180 asks for `\r\n`; `\n` is the other common
choice.

#### quote

The byte that quotes a field. Inside a quoted field it is written twice.

#### quote_all

Writing: quotes every field, not only the ones that need it.

#### skip_empty_lines

Reading: skips lines that are empty. When false an empty line is a row with one empty
field.

#### strict

Reading: every row must have as many fields as the first row, and quotes must follow
RFC 4180 exactly.

#### trim

Reading: removes spaces and tabs around unquoted fields, and allows them around quoted
fields.

```js
// Reads CSV one row at a time, from text in memory, a file or any `io.Reader`.
+ class Reader {
    // The line the last row that was read starts on, counting from one.
    ~ line: uint

    // Closes the file that `open` opened. Rows that were not read yet are not returned.
    + fn close() void
    // Returns the index of the column with this name in the header, for `row[index]`.
    + fn column(name: String) uint !Error
    // Creates a reader over text that is already in memory. The text is not copied.
    + static fn from_string(text: String, options: ?Options (null)) Reader
    // Returns the header row: empty without the `header` option, or when the input is empty.
    + fn header() Array[String] !Error
    // Creates a reader over any `io.Reader`, such as a `fs.FileStream` or a socket.
    + static fn new(source: Reader, options: ?Options (null), chunk_size: uint (65536)) Reader
    // Returns the next row, or `null` after the last one.
    + fn next() ?Array[String] !Error
    // Returns the next row as a map from column name to field, or `null` after the last one.
    + fn next_map() ?Map[String] !Error
    // Returns the next row as an object of your own class or struct, or `null` after the last one.
    + fn next_to[T]() ?T !Error
    // Opens a file and reads it `chunk_size` bytes at a time.
    + static fn open(path: String, options: ?Options (null), chunk_size: uint (65536)) Reader !Error
}
```

### Reader

Reads CSV one row at a time, from text in memory, a file or any `io.Reader`.

Only one chunk of the input and the row being read are held in memory, so a file of any size
can be read. A quoted field may run over chunk boundaries and over as many lines as it likes.

```valk
let reader = csv.Reader.open("users.csv", csv.Options { header: true }) ! panic("%{E.message}")
while true {
    let row = reader.next_map() ! panic("%{E.message} at line %{E.line}")
    if !isset(row) : break
    println(row.get("email") !? "")
}
```

#### line

The line the last row that was read starts on, counting from one.

#### close

Closes the file that `open` opened. Rows that were not read yet are not returned.

A reader over another source leaves closing it to its owner.

#### column

Returns the index of the column with this name in the header, for `row[index]`.

Throws `header` when there is no such column, or no header.

#### from_string

Creates a reader over text that is already in memory. The text is not copied.

#### header

Returns the header row: empty without the `header` option, or when the input is empty.

#### new

Creates a reader over any `io.Reader`, such as a `fs.FileStream` or a socket.

The source is read `chunk_size` bytes at a time, at least 4.

#### next

Returns the next row, or `null` after the last one.

Empty lines are skipped unless `skip_empty_lines` is off, and so are comment lines when a
`comment` byte is set. With the `header` option on, the first row is the header, which
`header` returns, and never comes out of here.

#### next_map

Returns the next row as a map from column name to field, or `null` after the last one.

Needs the `header` option. A row that is shorter than the header leaves its missing
columns out of the map, and fields past the end of the header are dropped; strict mode
turns both into an error.

#### next_to

Returns the next row as an object of your own class or struct, or `null` after the last
one.

Needs the `header` option: every field of `T` is read from the column with its name, the
way `decode_to` does it.

```valk
while true {
    let user = reader.next_to[User]() ! panic("%{E.message} at line %{E.line}")
    if !isset(user) : break
}
```

#### open

Opens a file and reads it `chunk_size` bytes at a time.

The file closes when it has been read to the end; `close` closes it sooner.

```js
// Writes CSV rows to any `io.Writer`: a `fs.FileStream`, a socket or a `ByteBuffer`.
+ class Writer {
    // Writes the rows that are waiting to the destination.
    + fn flush() void !Error
    // Creates a writer that writes to `out`.
    + static fn new(out: Writer, options: ?Options (null)) Writer
    // Writes the names of the fields of `T` as a row: the header for `write_object`.
    + fn write_header_of[T]() void !Error
    // Writes the fields of a class or struct as one row, in the order they are declared.
    + fn write_object(item: $T) void !Error
    // Writes one row.
    + fn write_row(fields: Array[String]) void !Error
    // Writes rows one after the other.
    + fn write_rows(rows: Array[Array[String]]) void !Error
}
```

### Writer

Writes CSV rows to any `io.Writer`: a `fs.FileStream`, a socket or a `ByteBuffer`.

Rows are collected and passed on in large writes, so call `flush` after the last one. A
`ByteBuffer` is written into directly and needs no flush.

A field is quoted only when it has to be: when it holds the delimiter, the quote, `\r` or
`\n`, or starts or ends with a space or a tab, which a reader that trims would lose. A quote
inside a quoted field is written twice. A row of one empty field is written as `""`, so it
does not become an empty line, and a row whose first field starts with the `comment` byte
has that field quoted.

```valk
let file = fs.stream("out.csv", fs.OpenOptions { read: false, write: fs.WriteMode.truncate, create: true }) ! panic("open")
let writer = csv.Writer.new(file)
writer.write_row(.{ "name", "age" }) ! panic("%{E.message}")
writer.write_row(.{ "ada", 36 }) ! panic("%{E.message}")
writer.flush() ! panic("%{E.message}")
```

#### flush

Writes the rows that are waiting to the destination.

#### new

Creates a writer that writes to `out`.

#### write_header_of

Writes the names of the fields of `T` as a row: the header for `write_object`.

#### write_object

Writes the fields of a class or struct as one row, in the order they are declared.

Numbers and bools are written as text, and a `null` as an empty field.

#### write_row

Writes one row.

Throws `io` when the destination fails, which may happen on a later row than the one that
did not arrive, because rows are collected before they are written.

#### write_rows

Writes rows one after the other.
