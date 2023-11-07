>  NOTE :  extracted from https://mikefarah.gitbook.io/yq/usage/tips-and-tricks

Yaml  files can be surprisingly lenient in what can be parsed as a yaml file. A reasonable way of validation a yaml file is to ensure the top level is a map or array (although it is valid yaml to have scalars at the top level, but often this is not what you want). This can be done by:

yq --exit-status 'tag == "!!map" or tag== "!!seq"' file.txt > /dev/null

## 

Split expressions over multiple lines to improve readability

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#split-expressions-over-multiple-lines-to-improve-readability)

Feel free to use multiple lines in your expression to improve readability.

Use `with` if you need to make several updates to the same path.

. = "newValue" | . style="single") |

. = "cool" | . style="folded")

You can create a bash array named `actions` by:

\> readarray actions < <(yq '.coolActions\[\]' sample.yaml)

For a given yaml file like:

\- arn: "arn:aws:iam::ARN1"

\- arn: "arn:aws:iam::ARN2"

You can loop over the results in a bash loop like:

\# load array into a bash array

\# output each entry as a single line json

readarray identityMappings < <(yq -o=j -I=0 '.identities\[\]' test.yml )

for identityMapping in "${identityMappings\[@\]}"; do

\# identity mapping is a single json snippet representing a single entry

roleArn=$(echo "$identityMapping" | yq '.arn' -)

## 

Set contents from another file

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#set-contents-from-another-file)

Use the [load](https://mikefarah.gitbook.io/yq/operators/load) operator to load contents from another file.

## 

Special characters in strings

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#special-characters-in-strings)

The `strenv` operator is a great way to handle special characters in strings:

VAL\='.a |!@ == "string2"' yq '.a = strenv(VAL)' example.yaml

`yq` doesn't have a way of updating multiple files in a single command (yet?) - but you can use your shell's built in tools like `find`:

find \*.yaml -exec yq '. += "cow"' -i {} \\;

This will run the `'. += "cow"'` expression against every matching file, and update it in place (`-i`).

## 

String blocks and newline issues

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#string-blocks-and-newline-issues)

There are a couple of tricks to getting the right string representation, take a look at [string operators](https://mikefarah.gitbook.io/yq/operators/string-operators#string-blocks-bash-and-newlines) for more details:

## 

Quotes in Windows Powershell

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#quotes-in-windows-powershell)

Powershell has its [own](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules?view=powershell-7.1) way of handling quotes:

PS \> yq \-n '.test = ""something""'

## 

Merge / combine all documents into one

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#merge-combine-all-documents-into-one)

To merge all given yaml files into one, use the `reduce` operator with the `*` (multiply) operator. Note the use of `ea` or `eval-all` to load all files into memory so that they can be merged.

yq ea '. as $item ireduce ({}; . \* $item )' file1.yml file2.yml ...

## 

Merge - showing the source file and line

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#merge-showing-the-source-file-and-line)

To see the original source file and line number of your merged result, you can pre-process the files and add that information in as line comments, then perform the merge.

yq ea '(.. lineComment |= filename + ":" + line) | select(fi==0) \* select(fi==1)' data1.yaml data2.yaml

## 

Merge an array of objects by key

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#merge-an-array-of-objects-by-key)

See [here](https://mikefarah.gitbook.io/yq/operators/multiply-merge#merge-arrays-of-objects-together-matching-on-a-key) for a working example.

See [here](https://mikefarah.gitbook.io/yq/operators/path#set-path-to-prune-deep-paths) for a working example.

## 

Creating a new file / working with blank documents

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#creating-a-new-file-working-with-blank-documents)

To create a new `yaml` file simply:

yq -n '.someNew="content"' > newfile.yml

The best way to run a diff is to use `yq` to normalise the yaml files and then just use diff. Here is a simple example of using pretty print `-P` to normalise the styling and running diff:

diff <(yq -P 'sort\_keys(..)' -o=props file1.yaml) <(yq -P 'sort\_keys(..)' -o=props file2.yaml)

This way you can use the full power of `diff` and normalise the yaml files as you like.

You may also want to remove all comments using `... comments=""`

## 

Reading multiple streams (STDINs)

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#reading-multiple-streams-stdins)

Like `diff` and other bash commands, you can use `<(exp)` to pipe in multiple streams of data into `yq`. instance:

yq '.apple' <(curl -s https://somewhere/data1.yaml) <(cat file.yml)

## 

Updating deeply selected paths

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#updating-deeply-selected-paths)

### 

or why is yq only returning the updated yaml

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#or-why-is-yq-only-returning-the-updated-yaml)

The most important thing to remember to do is to have brackets around the LHS expression - otherwise what `yq` will do is first filter by the selection, and then, separately, update the filtered result and return that subset.

yq '(.foo.bar\[\] | select(name == "fred) | .apple) = "cool"'

## 

Combining multiple files into one

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#combining-multiple-files-into-one)

In order to combine multiple yaml files into a single file (with `---` separators) you can just:

## 

Multiple updates to the same path

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#multiple-updates-to-the-same-path)

You can use the [with](https://mikefarah.gitbook.io/yq/operators/with) operator to set a nested context:

yq 'with(.a.deeply ; .nested = "newValue" | .other= "newThing")' sample.yml

The first argument expression sets the root context, and the second expression runs against that root context.

## 

Logic without if/elif/else

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#logic-without-if-elif-else)

`yq` has not yet added `if` expressions - however you should be able to use `with` and `select` to achieve the same outcome. Lets use an example:

Now, if you were using good ol' jq - you may have a script with `if`s like so:

if (.animal == "cat") then

elif (.animal == "dog") then

Using `yq` - you can get the same result by:

with(select(.animal == "cat");

with(select(.animal == "dog");

with(select(.noise == null);

Note that the logic isn't quite the same, as there is no concept of 'else'. So you may need to put additional logic in the expressions, as this has for the 'else' logic.

## 

yq adds a !!merge tag automatically

[](https://mikefarah.gitbook.io/yq/usage/tips-and-tricks#yq-adds-a-merge-tag-automatically)

The merge functionality from yaml v1.1 (e.g. `<<:`has actually been removed in the 1.2 spec. Thankfully, `yq` underlying yaml parser still supports that tag - and it's extra nice in that it explicitly puts the `!!merge` tag on key of the map entry. This tag tells other yaml parsers that this entry is a merge entry, as opposed to a regular string key that happens to have a value of `<<:`. This is backwards compatible with the 1.1 spec of yaml, it's simply an explicit way of specifying the type (for instance, you can use a `!!str` tag to enforce a particular value to be a string.

Although this does affect the readability of the yaml to humans, it still works and processes fine with various yaml processors.
