#  Why the CSV standard library is broken, broken, broken (and how to fix it)

What's broken (and wrong, wrong, wrong)
with the CSV standard library? Let's count the ways.



## CSV Format - Strict Out-of-the-Box CSV Format - No Batteries Included

Let's say you want to read in a CSV datafile
with comments and skipping blank lines
and stripping all leading and trailing whitespaces in values [^1]
and with the utf8-encoding:

Example - `beer11.csv`:

```
#######
# try with some comments
#   and blank lines even before header (first row)

Brewery,City,Name,Abv
Andechser Klosterbrauerei,Andechs,Doppelbock Dunkel,7%
Augustiner Bräu München,München,Edelstoff,5.6%

Bayerische Staatsbrauerei Weihenstephan,  Freising,  Hefe Weissbier,   5.4%
Brauerei Spezial,                         Bamberg,   Rauchbier Märzen, 5.1%
Hacker-Pschorr Bräu,                      München,   Münchner Dunkel,  5.0%
Staatliches Hofbräuhaus München,          München,   Hofbräu Oktoberfestbier, 6.3%
```

Let's start with the CSV configuration extravaganza.
We need two regular expressions (regex) -
one for skipping comments and one for skipping blank lines with spaces only:

``` ruby
COMMENTS_REGEX = /^\s*#/
BLANK_REGEX    = /^\s*$/   ## skip all whitespace lines - note: use "" for a blank record
SKIP_REGEX = Regexp.union( COMMENTS_REGEX, BLANK_REGEX )
```

Next we need a custom converter for stripping leading and trailing spaces
from values:

``` ruby
##  register our own converters
CSV::Converters[:strip] = ->(field) { field.strip }
```

and finally all-together lets setup the csv option hash:

``` ruby
csv_opts = {
  skip_lines:  SKIP_REGEX,
	skip_blanks: true,    ## note: skips lines with no whitespaces only!! (e.g. line with space is NOT blank!!)
  :converters => [:strip],
	encoding: 'utf-8'
}
```

Now try:

```
pp CSV.read( "beer11.csv", csv_opts )
```

Resulting in:

``` ruby
[["Brewery", "City", "Name", "Abv"],
 ["Andechser Klosterbrauerei", "Andechs", "Doppelbock Dunkel", "7%"],
 ["Augustiner Br\u00E4u M\u00FCnchen", "M\u00FCnchen", "Edelstoff", "5.6%"],
 ["Bayerische Staatsbrauerei Weihenstephan", "Freising", "Hefe Weissbier", "5.4%"],
 ["Brauerei Spezial", "Bamberg", "Rauchbier M\u00E4rzen", "5.1%"],
 ["Hacker-Pschorr Br\u00E4u", "M\u00FCnchen", "M\u00FCnchner Dunkel", "5.0%"],
 ["Staatliches Hofbr\u00E4uhaus M\u00FCnchen", "M\u00FCnchen", "Hofbr\u00E4u Oktoberfestbier", "6.3%"]]
```

Why not make the human format the default?
So everybody can use it out-of-the-box with zero-configuration? E.g.

``` ruby
pp CSV.read( "beer11.csv" )
```

How many people do you think care enough to configure the CSV standard library
before reading?

Note ^1: Stripping all leading and trailing whitespaces in values will NOT
work with quoted values - the CSV standard library has no purpose-built parser
that handles stripping of whitespaces, and, thus, the strip converter
will also strip "escaped" spaces inside quoted values e.g.:

```
  1," ","2 ",3   
```

resulting in

``` ruby
["1","","2","3"]
```

instead of the expected (preserved whitespace in quoted values):

```
["1"," ","2 ","3"]
```



## CSV Format - Strict Conventions vs Human Conventions

The CSV standard library was born as fastercsv. The claim was that the
new fastercsv library is faster than the old CSV standard library.
What's the (dirty) trick? The fastercsv library uses a
very, very, very strict CSV format so it can be faster by
using `line.split(",")` that runs on native c-code.

Let's try to read / parse:

```
1, "2","3" ,4
```

Example:

``` ruby
require 'csv'

CSV.parse( %{1, "2"} )
# => Illegal quoting in line 1. (CSV::MalformedCSVError).

CSV.parse( %{"3" ,4} )
# => Unclosed quoted field on line 1. (CSV::MalformedCSVError)
```

Just adding leading or trailing spaces to quoted values
leads to format errors / exceptions. And, sorry, the custom `:strip` converter
only gets called AFTER parsing, thus, it won't help or fix anything.

What to do? Sorry, there's no easy shortcut.
Yes, we need a (new) purpose built-parser for handling all the (edge) cases
in the CSV format. Using the "super-fast" `line.split(",")` kludge will NOT work.  



## CSV Format - New Rule - (Unquoted) Empty Values Are `nil`

In the CSV format all values are by default strings. Remember: the CSV
format is a text format.
The CSV standard library makes up the "ingenious" new rule
that if you use empty quoted values (`"","",""`) you will get empty strings
(as expected) but if you use empty "unquoted" values (`,,`) - surprise, surprise -
you will get `nil`s. Example:

``` ruby
CSV.parse( %{"","",,} )
```

resulting in:

``` ruby
["", "", nil, nil]
```

instead of the expected (all string values):

``` ruby
["", "", "", ""]
```

The right way: All values are always strings. Period.

If you want to use `nil`s you MUST configure a string (or strings)
such as `NA`, `n/a`, `\N`, or similar that map to `nil`.




## Let' Welcome CsvReader :-) - The CSV Standard Library Alternative

Reads tabular data in the comma-separated values (csv) format the right way, that is, uses best practices out-of-the-box with zero-configuration.
Everything works as expected :-). Example:


``` ruby
require 'csvreader'

pp CsvReader.parse_line( %{1, "2"} )
# => ["1", "2"]

pp CsvReader.parse_line( %{"3" ,4} )
# => ["3","4"]

pp CsvReader.parse_line( %{"","",,} )
# => ["", "", "", ""]

pp CsvReader.read( "beer11.csv" )
# => [["Brewery", "City", "Name", "Abv"],
#     ["Andechser Klosterbrauerei", "Andechs", "Doppelbock Dunkel", "7%"],
#     ["Augustiner Br\u00E4u M\u00FCnchen", "M\u00FCnchen", "Edelstoff", "5.6%"],
#     ["Bayerische Staatsbrauerei Weihenstephan", "Freising", "Hefe Weissbier", "5.4%"],
#     ["Brauerei Spezial", "Bamberg", "Rauchbier M\u00E4rzen", "5.1%"],
#     ["Hacker-Pschorr Br\u00E4u", "M\u00FCnchen", "M\u00FCnchner Dunkel", "5.0%"],
#     ["Staatliches Hofbr\u00E4uhaus M\u00FCnchen", "M\u00FCnchen", "Hofbr\u00E4u Oktoberfestbier", "6.3%"]]
```

