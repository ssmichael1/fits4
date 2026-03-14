# fits4

Pure Rust implementation of the [FITS](https://fits.gsfc.nasa.gov/fits_standard.html) (Flexible Image Transport System) v4.0 standard for reading and writing astronomical data files.

Zero external dependencies for core functionality.

## Features

- **All standard HDU types** — Primary, IMAGE, ASCII TABLE, BINTABLE
- **All BITPIX types** — 8, 16, 32, 64, -32, -64
- **Variable-length arrays** — P and Q heap descriptors in binary tables
- **BSCALE/BZERO** — raw and scaled access with unsigned integer convention
- **CHECKSUM/DATASUM** — ones-complement integrity verification
- **`image` crate integration** — optional, behind `image` feature flag

## Usage

Add to your `Cargo.toml`:

```toml
[dependencies]
fits4 = "0.1"
```

### Reading a FITS file

```rust
use fits4::{FitsFile, HduData, PixelData};

let fits = FitsFile::from_file("image.fits")?;

let primary = fits.primary();
println!("BITPIX = {}", primary.header.get_int("BITPIX").unwrap());

if let HduData::Image(img) = &primary.data {
    println!("{}x{}", img.width().unwrap(), img.height().unwrap());
}

for hdu in fits.extensions() {
    match &hdu.data {
        HduData::Image(img) => println!("IMAGE: {:?}", img.axes),
        HduData::BinTable(t) => println!("BINTABLE: {} rows, {} cols", t.nrows, t.columns.len()),
        HduData::AsciiTable(t) => println!("TABLE: {} rows", t.nrows),
        HduData::Empty => {}
    }
}
# Ok::<(), fits4::Error>(())
```

### Writing a FITS file

```rust
use fits4::{FitsFile, Hdu, ImageData, PixelData, HeaderValue};

let pixels: Vec<i16> = (0..10000).map(|i| (i % 1000) as i16).collect();
let img = ImageData::new(vec![100, 100], PixelData::I16(pixels));

let mut fits = FitsFile::with_primary_image(img);
fits.primary_mut().header.set("OBJECT", HeaderValue::String("M31".into()), None);

fits.to_file("output.fits")?;
# Ok::<(), fits4::Error>(())
```

### Building a binary table

```rust
use fits4::{FitsFile, Hdu, BinTableBuilder, BinColumnType};

let table = BinTableBuilder::new()
    .add_column("RA", BinColumnType::D64(1))
    .add_column("DEC", BinColumnType::D64(1))
    .add_column("MAG", BinColumnType::E32(1))
    .push_row(|row| {
        row.write_f64(180.0);
        row.write_f64(45.0);
        row.write_f32(12.5);
    })
    .push_row(|row| {
        row.write_f64(90.0);
        row.write_f64(-30.0);
        row.write_f32(8.2);
    })
    .build();

let mut fits = FitsFile::with_empty_primary();
fits.push_extension(Hdu::bintable_extension(table));
# let _ = fits.to_bytes();
```

### Checksums

```rust
use fits4::{FitsFile, ImageData, PixelData};

let img = ImageData::new(vec![4], PixelData::U8(vec![1, 2, 3, 4]));
let fits = FitsFile::with_primary_image(img);

// Write with CHECKSUM/DATASUM keywords
let bytes = fits.to_bytes_with_checksum()?;

// Verify on read
let fits2 = FitsFile::from_bytes(&bytes)?;
fits2.primary().verify_datasum()?;
# Ok::<(), fits4::Error>(())
```

## Not supported

- **Tile-compressed images** (`ZIMAGE`, Rice/GZIP/HCOMPRESS) — use [`fpack`/`funpack`](https://heasarc.gsfc.nasa.gov/fitsio/fpack/) to decompress externally
- **Random groups** (deprecated in FITS v4.0)

## Feature flags

| Flag | Description |
|------|-------------|
| `image` | Enables conversion between `ImageData` and the [`image`](https://crates.io/crates/image) crate's `DynamicImage` |

## License

[MIT](LICENSE)
