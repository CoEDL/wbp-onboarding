<link rel="stylesheet" type="text/css" href="https://rawgit.com/ivan111/vtree/master/css/vtree.css">
<style>svg.vtree { width:100%; height:500px; box-shadow:none; }</style>

<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.17/d3.min.js"></script>
<script src="https://rawgit.com/ivan111/vtree/master/dist/vtree.js"></script>

<script type="text/javascript">
function displayVTree(json_data, svg_container) {
	var vt   = new VTree(document.getElementById(svg_container)),
	reader   = new VTree.reader.Object(),
	     s   = document.getElementById(json_data).innerText,
	jsonData = JSON.parse(s),
	    data = reader.read(jsonData)

	vt.data(data).update()
}
</script>

## Welcome!

This documentation introduces the Warlpiri dictionary project and the processes followed by its contributors. A 'big picture' view of the project and its various components are given on this landing page; more detailed information can be found by following the various hyperlinks (e.g. [Warlpiri backslash codes](wbp-backslash-codes)).

### Goals

The primary goal is to produce the first print edition of the Warlpiri dictionary, which involves verifying the dictionary data stored in a plain text file and transforming the raw text data to a near print-ready form.

As the raw data are verified and updated, we also want to make sure that these updates are valid (e.g. changing the spelling of a word may invalidate other entries in which this word is referenced), and—as these updates are accepted—continually transform the raw data to make sure the formatted output appear as intended.

An additional goal of the project is to develop and refine an efficient, sustainable method of performing this verification-transformation routine, so that *several* output forms can be regenerated as the raw data changes (e.g. a web version of the dictionary, or subsets such as bird or plant lists).
The flowchart below provides an illustration of the routine, each stage for which we'll give a short description below.

<div class="mermaid">
graph LR
A(Raw data) -->|Test suite| B(Verified data)
B --> |Parser| C(Parsed data)
C --> E{Templating system}
D(Media) --> E
E --> F(Print dictionary)
E --> G(Web dictionary)
E --> H(... Other derived views)
</div>

<script src="https://unpkg.com/mermaid@7.1.0/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>

### Raw text data

The raw data consists of dictionary entries and their attributes listed in a plain text file.
The example below shows a **m**ain **e**ntry (indicated with a 'backslash code' `\me`) of a Warlpiri word `yulpayi`, whose part of speech is a noun `(N)`, and for which the data lists an English definition (`\def`), and two Warlpiri synonyms (`\syn`).

```
\me yulpayi (N)
\def sand typically found in water-course \edef
\syn wulpayi, karru \esyn
\eme
```

Each starting backslash code has a corresponding ending one (`\me ... \eme`).
The [Warlpiri backslash codes](wbp-backslash-codes) page provides a complete of list of the codes with explanatory notes.

### Test suite, and verified data

We use the [Test Anything Protocol](http://testanything.org/) ([full specification](http://testanything.org/tap-version-13-specification.html)), which specifies a reporting format that is both human- and machine- readable (latter using various [TAP-consumers](http://testanything.org/consumers.html)).
For example, suppose that in the raw data `\me wulpayi` exists, but not `\me karru`. Then, running the 'cross-references verification' on `yulpayi` headword above would result in the following report:

```
TAP version 13
# Check validity of cross references
ok 1 Line 3: 'wulpayi' is a valid reference to an \me item
not ok 1 Line 3: 'karru' is a valid reference to an \me item

1..2
# tests 2
# pass  1
# fail  1
```

### Parser

#### Background

Consider [Phrase Structure Rules](https://en.wikipedia.org/wiki/Phrase_structure_rules) of the form `S -> NP VP`. We can define a grammar with a set of such rules to parse text data such as `Colorless green ideas sleep furiously` into a tree structure of the form:

<img width="300" src="https://upload.wikimedia.org/wikipedia/commons/8/82/Cgisf-tgg.svg">

One machine-readable way of representing the tree above is to use the JavaScript Object Notation (JSON). As shown in the code block below, the JSON object `S`, has 2 items within it (NP and VP), the first of which also has two items within it (A and NP). Thus, the data at `S[0][0].value` is the string `Colorless` (JSON indices start from 0, not 1).

```json
{
	"S" : 
		[
			[
				{ "node": "A", "value": "Colorless" },
				[
					{ "node": "A", "value": "green" },
					{ "node": "NP", "value": "ideas" }
				]
			],
			
			[
				{ "node": "V",   "value": "sleep" },
				{ "node": "Adv", "value": "furiously" }
			]
		]
}
```

Here's an interactive display of the same JSON data (click and drag to explore, use scroll wheel or touch pad to zoom in/out):

<div id="colorless-json" style="display:none">
{"S":[[{"node":"A","value":"Colorless"},[{"node":"A","value":"green"},{"node":"NP","value":"ideas"}]],[{"node":"V","value":"sleep"},{"node":"Adv","value":"furiously"}]]}
</div>
<div id="colorless-viz"></div>
<script>
displayVTree("colorless-json", "colorless-viz")
</script>

#### Nearley grammars in EBNF 

A formalised, machine-readable equivalent of phrase structure rules is the [Extended Backus-Naur Form](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) (EBNF). Using EBNF, we may define a data structure of a form:

```ne
main_entry     -> "\\me" [a-z]:+ "(" part_of_speech" ")"
part_of_speech -> "N" | "V"
``` 

These two production rules above ('rewrite rules' in phrase structure grammars) specify that a `main_entry` consists of an escaped backslash `\\`, followed by `me`, followed by 1 or more lowercase letters (`[a-z]:+`), followed by `part_of_speech`, which may be `N` or `V`.
Defining data structures in such a manner allows us to:

- test whether some data are item under such rules (i.e. make sure that verbs are always specified with `V`, and not `Verb` or `Vrb`, or even lowercase `v`)
- parse valid data into a nested form (e.g. a tree), in which we can access the data in specific positions

### Parsed data

We use the JavaScript parser toolkit [Nearley](https://nearley.js.org/) to transform raw text data such as:

```
\me yulpayi (N)
\def sand typically found in water-course \edef
\syn wulpayi, karru \esyn
\eme
```

into [JavaScript Object Notation](https://www.json.org/) (JSON) data (click and drag to explore data structure):

<div id="tree-json" style="display:none">
{"result":[{"type":"code","value":"me","text":"\\me ","offset":0,"lineBreaks":0,"line":1,"col":1},{"type":"line_data","value":"yulpayi (N)","text":"yulpayi (N)\n","offset":4,"lineBreaks":1,"line":1,"col":5},[{"type":"code","value":"def","text":"\\def ","offset":16,"lineBreaks":0,"line":2,"col":1},{"type":"line_data","value":"sand typically found in water-course","text":"sand typically found in water-course ","offset":21,"lineBreaks":0,"line":2,"col":6},{"type":"code","value":"edef","text":"\\edef\n","offset":58,"lineBreaks":0,"line":2,"col":43}],[{"type":"code","value":"syn","text":"\\syn ","offset":64,"lineBreaks":0,"line":2,"col":49},{"type":"line_data","value":"wulpayi, karru","text":"wulpayi, karru ","offset":69,"lineBreaks":0,"line":2,"col":54},{"type":"code","value":"esyn","text":"\\esyn\n","offset":84,"lineBreaks":0,"line":2,"col":69}],{"type":"code","value":"eme","text":"\\eme\n","offset":90,"lineBreaks":0,"line":2,"col":75}]}</div>
<div id="tree-viz"></div>
<script>displayVTree("tree-json", "tree-viz")</script>

Just as we accessed `Colorless` as `S[0][0].value` previously, we can access the value of the definition at `result[2][1].value`.

## Templating

We use [Embedded JavaScript](http://ejs.co/) (EJS) templates to render the parsed dictionary data into various outputs.
