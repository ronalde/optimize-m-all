# optimize-m-all

`optimize-m-all` is a bash script to perform lossless optimization and
optionally (re-)compression of image files.

The script is designed to be used in conjunction with GNU find and GNU
parallel to do (massive) batch jobs.


## File formats and requirements

`optimize-m-all` handles the following image formats with the tools
listed:

- `png`:  `optipng`  (http://optipng.sourceforge.net/)
- `gif`:  `gifsicle` (https://www.lcdf.org/gifsicle/)
- `jpeg`: `jpegtrans` (https://github.com/mozilla/mozjpeg):

NOTE:
  with the `-r QVAL` or `--recompress QVAL` argument, `identify` (from
  imagemagick) is used on the jpeg inputfile to check its compression
  ratio (`'Q'`uality level);
- when that exceeds QVAL `convert` from imagemagick is used to perform
  lossy re-compression (and optimization),
- otherwise `jpegtrans` is used losslessly.


## Exit/Return values

The script reports the effectiveness of the optimization on screen and
with the following return codes:

|   | value |   | description                                               |   |
|   | ===== |   | ========================================================= |   |
|   | 0     |   | original < optimized                                      |   |
|   | 1     |   | original = optimized                                      |   |
|   | 2     |   | original > optimized                                      |   |
|   | 3     |   | original file does not exist                              |   |
|   | 4     |   | optimized file does not exist, ie optimization failed     |   |
|   | 5     |   | other error / bug                                         |   |


## Outputs

|   | descriptor |   | description                                             |   |
|   | ========== |   |                                                         |   |
|   | stdout     | 1 | screen optimized output                                 |   |
|   | stderr     | 2 | errors reported by optimize function (ie effectiveness) |   |

For analyzing and/or logging purposes the `-c|--csvfile PATH` argument
may be used to save raw csv formatted record to a file specified:

```bash
optimize-m-all -i /in/path -o /out/path -c ~/stats.csv
```

## Usage

The script can be used from bash and even sourced, but is designed to
be used together with gnu parallel:

```bash
find /tmp/test -type f -print0 | \
    parallel -0 do_compress -i {} -o /path/out{}
```

A sample bash loop:

```
inpath="/tmp/test"
outpath="/path/out"
for infile in "${inpath}"/**/*.{jpg,png}; do
	outfile="${outpath}/${infile//${inpath}}"
	do_compress -i "${infile}" -o "${outfile}" 
done
```

Run `do_compress --help` or show usage information, or use `-p` to see
examples for using the script with GNU parallel.


## Default settings for the optimization tools
 
### jpegtran arguments

1. `-optimize`: do lossless optimization using Trellis quantization maximizing quality/filesize ratio.
2. `-progressive`: encode progressive with `jpegrescan` optimization
   (TODO: maybe depend on image size; ie <10KB do regular baseline
   encoding?)
3. `-copy "all"`: determine which metadata blocks to keep; any of (in
    order of increasing potential file size reduction and/or potential
    quality loss due to stripping color space info): all, comments,
    none

### convert (imagemagick) arguments
1. `-strip`: see remarks on `jpegtrans` `copy` argument above
2. `-define "jpeg:dct-method=float"`: seems like a good requantisation filter
3. `-interlace "Plane"`: progressive encoding

### optipng arguments

1. `-preserve`: preserve file attributes
2. `-clobber`: overwrite the existing output and backup files.

### gifsicle arguments

1. `--optimize=3`: optimize level 3 combines 1) stores only the
   changed portion of each image, 2) use transparency to shrink the
   file further, and 3) try several optimization methods (usually
   slower, sometimes better results).

## Example usage with GNU find and GNU parallel

### To optimize all images in the directory ``/tmp/test/in`
and save the resulting images to the directory `/tmp/test/out` one
might use:

```bash
 sourcedir=\"/tmp/test/in\"
 targetdir=\"/tmp/test/out\"
 find "${sourcedir}" -type f -print0 | parallel -0 ${appname} -i {} -o ${targetdir}{}
 ```

Result:
* `/tmp/test/in/some/sub/dir/image001.jpg => /tmp/test/out/some/sub/dir/image001.jpg`

### Example: flatten output directory structure

Naming of the target file name and path can be easily manipulated
using parallels `'{}'` switches.

For example, flatten the directory structure of the target directory:
```bash
find "${sourcedir}" -type f -print0 | parallel -0 ${appname} -i {} -o ${targetdir}{/}
```
Result:
* `/tmp/test/in/some/sub/dir/image001.jpg => /tmp/test/out/image001.jpg`

### Example: add an extra suffix to the output files

```
find "${sourcedir}" -type f -print0 | parallel -0 ${appname} -i {} -o ${targetdir}{/.}.optimized
```

Result:
* `/tmp/test/in/some/sub/dir/image001.jpg => /tmp/test/out/image001.jpg.optimized`

## Analysis of optimization using R

```R
## define header
colNames <- c("file_original","file_type","file_optimized","bytes_original","bytes_optimized","bytes_reduction","percentage_reduction")
m <- read.csv(file="stats.csv", sep=",", quote="\"", dec=".", header=FALSE, col.names=colNames)
summary(m)

```
