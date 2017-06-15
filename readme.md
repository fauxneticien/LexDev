# About

1-minute video summary: See live changes in a list of invalid parts of speech items (`invalid-ps.csv`), as the user makes changes to a backslash-coded dictionary file (`rotokas.dic`), and the documentation file of the parts of speech values used in the dictionary (`parts-of-speech.md`).

![Video summary](http://g.recordit.co/mXbEwJWopv.gif)

# Set up

## Mac

```Shell
# Download software into ~/Downloads/tmp
mkdir -p ~/Downloads/tmp
cd ~/Downloads/tmp
curl -O -L https://github.com/stedolan/jq/releases/download/jq-1.5/jq-osx-amd64
curl -O -L https://github.com/johnkerl/miller/releases/download/v5.2.0/mlr.macosx
curl -O -L https://github.com/go-task/task/releases/download/v1.3.1/task_1.3.1_macOS_x64.tar.gz
tar xzf task_1.3.1_macOS_x64.tar.gz

# Make executable
chmod +x jq-osx-amd64 mlr.macosx task

# Move into correct path
mv jq-osx-amd64 /usr/local/bin/jq
mv mlr.macosx /usr/local/bin/mlr
mv task /usr/local/bin/task

# Delete /tmp directory
cd ~/Downloads
rm -R /tmp
```

# Part I: Getting things out of plain text formats with `jq`

We're going to use [`jq`](https://stedolan.github.io/jq/), a "lightweight and flexible command-line JSON processor", to parse various plain text data (dictionary data, documentation files) into a structured format. One major advantage of using `jq` is being able to use [`jqplay.org`](https://jqplay.org/) with some test data to quickly come up with the parsing script.

## 1. Using `jq` to parse a backslash-coded file

### Input

Shown below is the first headword from the Rotokas dictionary (`rotokas.dic` openly available from the [NLTK corpora](https://github.com/nltk/nltk_data/tree/gh-pages/packages/corpora) in `toolbox.zip`).

```
\lx kaa
\ps V
\pt A
\ge gag
\tkp nek i pas
\dcsv true
\vx 1
\sc ???
\dt 29/Oct/2005
\ex Apoka ira kaaroi aioa-ia reoreopaoro.
\xp Kaikai i pas long nek bilong Apoka bikos em i kaikai na toktok.
\xe Apoka is gagging from food while talking.
```

### Output

Using `jq`, we can parse the backslash-coded data (which are essentially line-separated key-value pairs in the format: `\key value`) into individual JSON objects:

```JSON
{
  "key": "lx",
  "value": "kaa"
}
{
  "key": "ps",
  "value": "V"
}
{
  "key": "pt",
  "value": "A"
}
```

### Commands

#### Basic usage

The `jq` command below will produce the key-value JSON objects output given the `rotokas.dic` data as an input, as you can see here: [https://jqplay.org/s/1PlNwqyVhC](https://jqplay.org/s/1PlNwqyVhC)

***Note.*** the flag `--raw-input` needs to be passed to `jq` to indicate we're not ingesting JSON data, which is what `jq` assumes by default.

```Shell
jq --raw-input 'capture("\\\\(?<key>[a-z]+) (?<value>.*)")' rotokas.dic
```

#### Production usage

We're going to modify the usage above with two more bits to make the output data useful in a production setting:

1. Add line numbers to each object, so we know from where `rotokas.dic` the key-value pair originated.
2. Remove all unnecessary whitespace in the output

In the command below, #1 is achieved by piping the result of the capture (using `|`) into another `jq` expression: `.line_number = input_line_number`. This appends the key `line_number` into each object, assigning it the value `input_line_number`, which is made available by `jq` (Manual reference: [I/O](https://stedolan.github.io/jq/manual/v1.5/#IO)). We achieve #2 by setting the flag `--compact-output`

```Shell
jq --raw-input --compact-output \
	'capture("\\\\(?<key>[a-z]+) (?<value>.*)")
	| .line_number = input_line_number' rotokas.dic > rotokas.json
```

##### Production output (`rotokas.json` in JSON lines format: see [http://jsonlines.org/](http://jsonlines.org/))

```JSON
{"key":"lx","value":"kaa","line_number":1}
{"key":"ps","value":"V","line_number":2}
{"key":"pt","value":"A","line_number":3}
{"key":"ge","value":"gag","line_number":4}
{"key":"tkp","value":"nek i pas","line_number":5}
{"key":"dcsv","value":"true","line_number":6}
```

## 2. Using `jq` to parse a headers in a Markdown file (Exercise)

### Input

Suppose we have a Markdown file, called `parts-of-speech.md`, which has the following content:

```Markdown
# Rotokas parts of speech

## About

This document describes the part of speech values used in the Rotokas dictionary. 

## ADV: Adverb

## N: Noun

## V: Verb
```

### Output

We want to capture the initial part of some of the lines starting two or more `# `, then followed by a `:`, rendering the output below:

```JSON
{"value":"ADV"}
{"value":"N"}
{"value":"V"}
```

### Command

Exercise: use the `capture()` function from `jq` to process the Markdown headers, filling in the appropriate regular expression in the link: [https://jqplay.org/s/YqGkGYKaeI](https://jqplay.org/s/YqGkGYKaeI) (Answer given [here](https://jqplay.org/s/dWeikiBJKj)).

# Part II: Using Miller (`mlr`) to work with both JSON and CSV data

While `jq` is great for reading in raw plain-text data into structured *JSON* objects, not all data are originally in JSON, and not all subsequent steps down the line will be able to work with JSON.

## 1. The `filter` verb  (basic `mlr` usage)

### Input (rotokas.json)

The following data are selected lines from the `rotokas.dic` backslash-coded data which have been parsed into a JSON lines format (see Part I, Section 1).

```JSON
{"key":"lx","value":"kaa","line_number":1}
{"key":"ps","value":"V","line_number":2}
...
{"key":"lx","value":"kaa","line_number":17}
{"key":"ps","value":"V","line_number":18}
...
{"key":"lx","value":"kaa","line_number":35}
{"key":"ps","value":"N","line_number":36}
...
```

### Output

We can use the `filter` verb from `mlr` to keep only the parts of speech lines (i.e. JSON objects where the `key` is `ps`).

```JSON
{"key":"ps","value":"V","line_number":2}
{"key":"ps","value":"V","line_number":18}
{"key":"ps","value":"N","line_number":36}
...
```

### Command 

***Note.*** The `--json` flag which indicates to `mlr` that both the input and output should be in JSON (same as `mlr --ijson --ojson ...`).

```Shell
mlr --json filter '$key == "ps"' rotokas.json
```

#### CSV output

With Miller, we can very easily get CSV (or other output, see documentation [http://johnkerl.org/miller/doc/file-formats.html](http://johnkerl.org/miller/doc/file-formats.html)) by switching the flag:

```Shell
mlr --ijson --ocsv filter '$key == "ps"' rotokas.json
```
which gives us:

```CSV
key,value,line_number
ps,V,2
ps,V,18
ps,N,36
...
```

## 2. Find all invalid parts of speech values (anti-join pipeline with `mlr`)

### Input

Recall the following data:

1. `ps-whitelist.json`: JSON-lines data extracted from a Markdown document (See Part I, Section 2). Let's call this the data "on the left" (will be evident later).

	```
	{"value":"ADV"}
	{"value":"N"}
	{"value":"V"}
	...
	```
2. Output from `mlr ... filter ...` from the previous section (i.e. Part II, Section 1), and call this the data "on the right".

	```
	{"key":"ps","value":"V","line_number":2}
	...
	{"key":"ps","value":"???","line_number":78}
	...
	{"key":"ps","value":"CLASS","line_number":1057}
	...
	```

Notice the both files share a common field called `value`. We can use `mlr` to query for all lines in #2 (right data) which do ***not*** have a match in #1 (left data), and output this query as CSV data.

### Output

```CSV
key,value,line_number
ps,???,78
ps,CLASS,1057
```

### Command

Given an excerpt of the usage information from `mlr` verb called `join` (`mlr join help`):

<pre>
Usage: mlr join [options]
Joins records from specified left file name with records from all file names
at the end of the Miller argument list. Functionality is essentially the same as the system "join" command,
but for record streams.
Options:
  <b>-f</b> 	{left file name}
  <b>-j</b> 	{a,b,c}   Comma-separated join-field names for output
  ...
  <b>--np</b>   Do not emit paired records
  --ul		Emit unpaired records from the left file
  <b>--ur</b>   Emit unpaired records from the right file(s)
</pre>

The following command:

1. Filters the dictionary data, keeping only objects with `ps` values, then pipes this data to the next command which
2. Joins `ps-whitelist.json` to the left of the incoming data (i.e. from #1) and outputs/emits only data that are:
	a. Non-paired between the left and right data (`--np`), and
	b. Originated from the right data (`--ur`)
3. Formats the emitted data as comma-separated values (`--ocsv`)

<pre>
mlr --json filter '$key == "ps"' rotokas.json |
mlr --ijson --ocsv \
    join  <b>--np --ur</b> \
          -j "value" \
          -f ps-whitelist.json
</pre>

# Part III: Streamlining and automating command-line tools with `task`

`task` is a cross-platform task runner written in Go: [https://github.com/go-task/task](https://github.com/go-task/task), and takes a YAML file called `Taskfile.yml` as its input.

## 1. Basic usage of `task`

### Input

```YAML
say-hello:
    cmds:
    	- echo "hello"
```

### Output

```Shell
hello
```

### Command

```Shell
task say-hello
```

## 2. Backslash file conversion with `jq` and `task`

By using `task`, we can give make a frequently-used command, or a set of commands, into a task and give it a name (e.g. `backslash-to-json`). For each task in a Taskfile, you can also (optionally) give it a description of what the task achieves. The commands associated with the task are then defined as a list of commands using the `cmds` key.

### Input (`Taskfile.yml`)

```YAML
backslash-to-json:
  desc: "Converts rotokas.dic to a JSON-lines file"
  cmds:
    - cat rotokas.dic |
      jq --raw-input --compact-output
         'capture("\\\\(?<key>[a-z]+) (?<value>.*)")' > rotokas.json
```

### Output (`rotokas.json`)

```JSON
{"key":"lx","value":"kaa"}
{"key":"ps","value":"VFR"}
{"key":"pt","value":"A"}
{"key":"ge","value":"gag"}
{"key":"tkp","value":"nek i pas"}
{"key":"dcsv","value":"true"}
{"key":"vx","value":"1"}
{"key":"sc","value":"???"}
...
```

### Command

```Shell
task backslash-to-json
```

## 3. Automation with `task`

We can modify the `Taskfile.yml` from the previous section, giving the `backslash-to-json` task an explicit list of files in the `sources` and `generates` keys. When `task` is run with the `--watch` option, it will monitor the source file(s) and automatically re-run the task when it detects changes to the source file(s).

### Input (`Taskfile.yml`)

```YAML
backslash-to-json:
  desc: "Converts rotokas.dic to a JSON-lines file"
  sources:
    - rotokas.dic
  generates:
    - rotokas.json
  cmds:
    - cat rotokas.dic |
      jq --raw-input --compact-output
         'capture("\\\\(?<key>[a-z]+) (?<value>.*)")' > rotokas.json
```

### Output (`rotokas.json`)

Same as section above.

```Shell
task --watch backslash-to-json
```

# Part IV: continuous testing of dictionary data with `jq`, `mlr`, and `task`

We'll now combine all the parts above to demonstrate how we can continuously re-generate a report of invalid parts of speech data, given a dictionary (e.g. `rotokas.dic`), and a set of valid values defined through a documentation file, written in Markdown (e.g. parts-of-speech.md).

## Inputs

1. `rotokas.dic`: A backslash-coded dictionary file
2. `parts-of-speech.md`: A Markdown document, with valid parts of speech defined in its headers (e.g. `## V: Verb`)
3. `Taskfile.yml`: a YAML document containing various tasks to run as files #1 and #2 are modified

    ```YAML
    backslash-to-json:
      desc: "Converts rotokas.dic to a JSON-lines file"
      sources:
        - rotokas.dic
      generates:
        - rotokas.json
      cmds:
        - cat rotokas.dic |
          jq --raw-input --compact-output
             'capture("\\\\(?<key>[a-z]+) (?<value>.*)")
             | .line_number = input_line_number' > rotokas.json

    parse-ps-whitelist:
      desc: "Extracts valid parts of speech from headers in parts-of-speech.md"
      sources:
        - parts-of-speech.md
      generates:
        - ps-whitelist.json
      cmds:
        - cat parts-of-speech.md |
          jq --raw-input --compact-output
             'capture("^#{2,}\\s(?<value>[A-Z]+):\\s")' > ps-whitelist.json 

    test-ps-values:
      desc: "Returns invalid parts of speech data from the dictionary"
      sources:
        - rotokas.dic
        - parts-of-speech.md
      generates:
        - invalid-ps.csv
      cmds:
        - ^backslash-to-json
        - ^parse-ps-whitelist
        - cat rotokas.json |
          mlr --json filter '$key == "ps"' |
          mlr --ijson --ocsv
              join  --np --ur
                    -j "value"
                    -f ps-whitelist.json > invalid-ps.csv
    ```
    
## Outputs

```CSV
key,value,line_number
ps,VFR,5
ps,???,78
ps,???,214
ps,???,308
ps,???,350
ps,???,544
ps,???,713
ps,???,742
ps,???,1046
ps,???,1081
ps,???,2004
ps,???,2425
...
```

## Command

```Shell
task --watch test-ps-values
```
