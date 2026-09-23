
# Documentation

Namespaces: [main](#main)

---

# main

## Errors for 'main'

```js
// Thrown when CSV cannot be read, or cannot be written to its destination.
+ error Error (syntax, fields, header, type, io) payload { message: String, line: uint (0), column: uint (0) }
```

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
