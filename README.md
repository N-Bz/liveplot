# What is liveplot
`liveplot` is a plotter (using matplotlib) reading data from stdin, and suitable for live-plotting of high frequency data.

It was designed as a way to directly plot data from a live text log without requiring a text processing intermediate program (as `awk` or `sed`), and to allow multiple plots to be drawn at the same time.

By default, the Y axis is autoscaled based on the data found in the plots, and retains the min/max values even when no longer visible.

# Requirements
`liveplot` requires python3 (tested on 3.8), with the matplotlib module installed (available from PyPI)

# Simple usages

## Space-separated values (SSV)
`liveplot` can plot space-separated values without any argument.

The first line will be used to determine the number of column of the SSV document, and each column will take the name of the first non-numeric/non-empty value it encounters (and will otherwise be called `columnN`).
If a line contains only non-numeric/empty values, it will not generate any data point for the plotters, this allows full-title lines to appear multiple times, inlined with actual data. Otherwise, any invalid data (non-numeric value, or absent column) will generate a repeat of the latest valid data for this column (or a 0.0 if the column had no previous valid data). Extra columns are ignored.

This mode can also be used for single value lines.

## Comma-separated values (CSV)
`liveplot` accepts a `--csv [SEPARATOR]` flag, setting it into a csv mode, where values are expected to be separated by the SEPARATOR character (defaulting to `,`). This mode works otherwise like the default SSV mode.

# Regex usage
If the input is more complex (i.e. when parsing a log which was not designed to be liveplotted), `liveplot` accepts python regexes as its arguments. Each given regex will create a plot on the graph, fed only by data matching the given expression.

As such, if the value generator dumps line with the form `value = 1.234` calling liveplot as `<value_generator> | liveplot 'value = ([.0-9]+)'` would plot the value, ignoring any output line not matching the regex.

# Other arguments
`liveplot` accepts the following arguments:

 - `-t`/`--title`: The title of the graph window
 - `-w`/`--window`: The number of values in the graph (i.e. `liveplot` will only display the latest N values read). Defaults to 500, but will shrink if stdin is closed before this amount of data is read.
 - `-r`/`--refresh-time`: The minimum time between redraws of the graph (in seconds). When stdin is closed, a final draw will be called. Defaults to 0.1
 - `-m`/`--min`: The initial minimum value of the Y axis
 - `-M`/`--max`: The initial maximum value of the Y axis
 - `--no-auto`: Disable the autoscale feature. When given, the `-m`/`--min` and `-M`/`--max` must also be present
 - `--no-auto-memory`: When set, this flag disables the memory part of the autoscaler, meaning that the min/max values of the graph will only be taken from the displayed data.
 - `--csv [separator]`: When set, the input is expected to be in CSV format. If the separator is given, it will be used instead of the normal `,`.
 - `--columns [col1 [col2 [...]]]` : All data are plotted by default. This argument restricts the plotted columns to the selected one (index starting at 1). It can be given multiple times to create multiple figures.
 - `-f`/`--filter`: Filter all inputs based on given regex. All lines not matching the filter regex will be discarded, other will have only the first capturing group passed to the plotters.

 # Examples

Plot a line (y=x), 500 last points of 1000 given:
`seq 1000 | liveplot`

Plot multiple values from space-separated data:
```
echo "1 2 3
4 5 6
7 8 9" | liveplot
```

Plot multiple values from a CSV document
```
echo "A, B, C
1, 2, 3
4, 5, 6
7, 8, 9" | liveplot --csv
```

Plot multiple values based on regexes:
```
echo "A=1, B=2
A=2, B=4
A=3, B=6" | liveplot '/*A=([0-9]+).*' '.*B=([0-9]+).*'
```

Same example, but with the input on different lines:
```
echo "A=1
B=2
A=2
B=4
A=3
B=6" | liveplot 'A=([0-9]+)' '.*B=([0-9]+).*'
```

## Using the `--columns` argument:

Plot only part of a CSV document
```
echo "UsefulValueA, UselessHugeValueB, UsefulValueC
1, 2000, 3
4, 5000, 6
7, 8000, 9" | liveplot --csv --columns 1 3
```

Plot multiple parts of the same CSV onto multiple figures
```
echo "SmallValueA, HugeValueB, SmallValueC
1, 2000, 3
4, 5000, 6
7, 8000, 9" | liveplot --csv --columns 1 3 --columns 2
```

Plot only some of the given regexes
```
echo "A=1, B=2
A=2, B=4
A=3, B=6" | liveplot '/*A=([0-9]+).*' '.*B=([0-9]+).*' --columns 1
```


## Using the `--filter` argument for SSV/CSV modes:

The `--filter` argument allows using SSV/CSV modes when the input contains non-ssv/csv lines. This can be useful when plotting data from a verbose program:

```
echo "The progam logs a lot of things
PLOTLINE: A, B, C
And plots periodically
PLOTLINE: 1, 2, 3
PLOTLINE: 4, 5, 6
Even while running" | liveplot --csv --filter 'PLOTLINE: (.*)'
```

This is equivalent to

```
echo "A, B, C
1, 2, 3
4, 5, 6" | liveplot --csv
```

But avoids using `sed`/`awk` on the input, which might add buffering.

## Handling of invalid lines on SSV/CSV modes:

By default, SSV/CSV mode expects to have either full-title line (i.e. lines with no numerical data), or full-data lines (lines with only numerical data). Mixed data/title lines, and lines with empty columns are handled according to the following sample:

```
echo "5, B, 10
1,, 3
Z, 5, 6, 42
7, 8" | liveplot --csv
```
Will generate three plots, named `Z`, `B` and `column3`, with respectively [5, 1, 1, 7], [0, 0, 5, 8] and [10, 3, 6, 6] as their values.

This is what happens on every line:

 - `5, B, 10`: This line defines the number of columns. Since at least one value is convertible to float, it is considered to be a value-line. The plot values after this step is: `{column1: [5], B: [0], column3[10]}`
 - `1,, 3`: Plots 1 & 3 get new values while plot 2 gets a copy of the previous one: `{column1: [5, 1], B: [0, 0], column3[10, 3]}`
 - `Z, 5, 6, 42`: Plot 1 gets a copy of its previous value, but its title is updated. Plots 2 & 3 get new values. The `42` is ignored as the initial line only had three columns: `{Z: [5, 1, 1], B: [0, 0, 5], column3[10, 3, 6]}`
 - `7, 8`: Plots 1 & 2 get new values, while plot 3 has no new value and gets a copy of the preivous one: `{Z: [5, 1, 1, 7], B: [0, 0, 5, 8], column3[10, 3, 6, 6]}`


# TODOs

- Add synchronisation between plots
- Add X/Y plotting ?
