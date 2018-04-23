# optimize-m-all

`optimize-m-all` is a bash script to perform lossless optimization and
optionally (re-)compression of image files.

The script is designed to be used in conjunction with [GNU
find](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
and [GNU parallel](https://www.gnu.org/software/parallel/) to do
(massive) batch jobs.


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

Run `do_compress --help` or show usage information.

Also see:
* [More usage examples with GNU find and GNU parallel](#more-usage-examples-with-gnu-find-and-gnu-parallel)
* [Example: analysis of optimization results in generated csv file using R](#more-usage-examples-with-gnu-find-and-gnu-parallel)

## File formats and requirements

`optimize-m-all` handles the following image formats with the tools
listed:

- `png`:  `optipng`  (http://optipng.sourceforge.net/)
- `gif`:  `gifsicle` (https://www.lcdf.org/gifsicle/)
- `jpeg`: `jpegtrans` (https://github.com/mozilla/mozjpeg):

  NOTE: with the `-r QVAL` or `--recompress QVAL` argument, `identify`
  (from imagemagick) is used on the jpeg inputfile to check its
  compression ratio (`'Q'`uality level);
  - when that exceeds QVAL `convert` from imagemagick is used to
    perform lossy re-compression (and optimization),
  - otherwise `jpegtrans` is used losslessly.


## Exit/Return values

The script reports the effectiveness of the optimization on screen and
with the following return codes:

| value | variable               | description                                                                   |
| ----: | ---------------------: | :---------------------------------------------------------------------------- |
| 0     | `err_effect_positive`  | original > optimized (positive effect)                                        |
| 1     | `err_effect_zero`      | original = optimized (no effect)                                              |
| 2     | `err_effect_negative`  | original < optimized (negative effect)                                        |
| 3     | `err_input_file`       | original (input) file does not exist (anymore)                                |
| 4     | `err_input_type`       | original (input) file is unhandled file type                                  |
| 5     | `err_output_file`      | optimized (output) file does not exist (anymore), and/or optimization failed  |
| 6     | `err_output_overwrite` | optimized (output) file did exit before optimization, and `arg_force` not set |
| 7     | `err_output_mkdir`     | could not create target directory for optimized (output) file                 |
| 8     | `err_usage`            | invalid commandline argument or argument value (or `--help` argument)         |
| 9     | `err_bug`              | other error ie program bug                                                    |
| >=10  | `err_cmd_optimize`     | external optimization tool failed with exit value (`x-10`)                    |



## Outputs

| descriptor | # | description                                             |
| ---------: | - | ------------------------------------------------------  |
| stdout     | 1 | screen optimized output                                 |
| stderr     | 2 | errors reported by optimize function (ie effectiveness) |

For analyzing and/or logging purposes the `-c|--csvfile PATH` argument
may be used to save raw csv formatted record to a file specified:

```bash
optimize-m-all -i /in/path -o /out/path -c ~/stats.csv
```

## Default settings for the optimization tools

### gifsicle arguments

1. `--optimize=3`: optimize level 3 combines 1) stores only the
   changed portion of each image, 2) use transparency to shrink the
   file further, and 3) try several optimization methods (usually
   slower, sometimes better results).

### optipng arguments

1. `-preserve`: preserve file attributes
2. `-clobber`: overwrite the existing output and backup files.


### jpegtran arguments

1. `-optimize`: do lossless optimization using 
   [Trellis quantization](https://en.wikipedia.org/wiki/Trellis_quantization)
   maximizing quality/filesize ratio.
2. `-progressive`: encode progressive with `jpegrescan` optimization
   (TODO: maybe depend on image size; ie <10KB do regular baseline
   encoding?)
3. `-copy "all"`: determine which metadata blocks to keep; any of (in
    order of increasing potential file size reduction and/or potential
    quality loss due to stripping color space info): all, comments,
    none

### convert (imagemagick) arguments
1. `-quality` set dynamically when `-r/--recompression` is set:
   Imagemagick convert defaults to the estimated quality ofthe input
   image or uses `92` when the estimation fails. When the quality is
   greater than 90, then the chroma channels are not downsampled, see
   notes on the `-sampling-factor` argument below. The script does the
   same before handing it over to `convert`; it uses `identify` to
   read the source jpeg `Q` value and translates that to
   `max_jpeg_quality`; if `source Q > max_jpeg_quality` it adds the
   convert argument:

```bash
convert_args+=(-quality "${max_jpeg_quality}")
```

2. `-strip`: see remarks on `jpegtrans` `copy` argument above
3. `-interlace "plane"`: use "baseline progressive" JPEG interlaced
   encoding (instead of "baseline sequential"), use either `line` or
   `plane`:
	* `line = scanline: RRR...GGG...BBB...RRR...GGG...BBB...`
	* `plane:           RRRRRR...GGGGGG...BBBBBB...`
4. `-sampling-factor 4:2:2`: [Chroma subsampling](https://en.wikipedia.org/wiki/Chroma_subsampling);

| subsampling name | downsampling of luminance: Y | Downsampling of resolution of chroma: Cb(lue) Cr(ed) | net effect on file size | block splitting MCU size |
| ---------------: | ---------------------------: | ---------------------------------------------------: | ----------------------: | -----------------------: |
| 4:2:0            | none                         | halved vertical and horizontal                       | 1/2                     | 16x16                    |
| 4:2:2            | none                         | halved horizontal                                    | 1/3                     | 16x8                     |
| 4:4:4            | none                         | none                                                 | 1                       | 8x8                      |

5. `-define "jpeg:dct-method=float"`; After applying sub sampling and
    block splitting in Minimum Coded Units (MCU), 
	[DCT quantization](https://en.wikipedia.org/wiki/Quantization_(image_processing)#Quantization_matrices)
	is applied. First DCT coefficients are calculated, than those
	values are divided with a standardized quantisation matrix, and
	than rounded to integers. The rounding is lossy by introducing
	rounding errors applied to high frequency brightness variation
	(luminance components). The higher the precision used to round the
	brightness values, the less the probability of visual degradation
	becomes. Tradeoff for higher precision: encoding time and cost.

   The last step in the JPEG encoding is 
   [entropy encoding](https://en.wikipedia.org/wiki/Entropy_encoding) which is
   lossless: a run-length encoding (RLE) algorithm is applied on each
   block in effect grouping similar frequencies together and inserts
   length coding zeros, and then uses Huffman coding to compress what
   is left.


## More usage examples with GNU find and GNU parallel

### Example: use bash variables for specifying input and output directories

To optimize all images in the directory `/tmp/test/in` and save the
resulting images to the directory `/tmp/test/out` one might use:

```bash
 sourcedir=\"/tmp/test/in\"
 targetdir=\"/tmp/test/out\"
 find "${sourcedir}" -type f -regextype posix-extended -regex '.*\.(jp[e]*[g]*|jfi[f]*|jif|png|gif)' -print0 | parallel -0 ./optimize-m-all -i {} -o ${targetdir}{}
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

### Example: strip path and add an extra suffix to the output files

```
find "${sourcedir}" -type f -print0 | parallel -0 ${appname} -i {} -o ${targetdir}{/.}.optimized
```

Result:
* `/tmp/test/in/some/sub/dir/image001.jpg => /tmp/test/out/image001.jpg.optimized`

## Example: analysis of optimization results in generated csv file using R


1. perform optimization and store results in a csv file:
```bash
find /var/www/wp-uploads -type f -print0 | parallel -0 ${appname} -i {} -o /tmp/{} -c stats.csv
```
2. Use R to import hte csv, add some columns, sumarrize the results and plot the effectiveness per file type:

```R
## define header
colNames <- c("file_original","file_type","file_optimized","bytes_original","bytes_optimized","bytes_reduction","percentage_reduction")
m <- read.csv(file="stats.csv", sep=",", quote="\"", dec=".", header=FALSE, col.names=colNames)
## display summary
summary(m)

## add human readable sizes (KiB)
m$hbytes_original <- m$bytes_original / 1024
m$hbytes_optimized <- m$bytes_optimized / 1024
m$hbytes_reduction <- m$bytes_reduction / 1024

## plot per file type
library("ggplot2")
perc_plot <- ggplot(m, aes(x=hbytes_original, y=percentage_reduction, size = hbytes_optimized, colour=file_type)) + labs(title="Optimization effectiveness by file size and type", x = "Original file size (KiB)", y = "Percentage reduced (%)", colour="File type", size="Size of reduction (KiB)")  + geom_point()  + facet_wrap(~file_type)
## display
perc_plot
```
