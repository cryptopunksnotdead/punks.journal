# I Apologize  - Sorry, Sorry, Sorry

>  I'm seeing from you is that we should not consider people's feelings when criticizing their work. [...]
>  Please take time to sit down [..] and offer an apology to the author of the CSV library.


Don't get me wrong - the author or the authors of the standard CSV library 
deserve our hugs and thank yous for the great work and many hours (for sure many unpaid and volunteered)
put into the CSV library. We are all standing on the shoulders of giants.
 
Now it's 2019 and the standard CSV library is an unloved and uncared orphan that is no longer
"state-of-the-art" or the best it can be. At it's core is a `string#split` kludge:

``` ruby
 parts =  parse.split(@col_sep_split_separator, -1)
      if parts.empty?
        if in_extended_col
          csv[-1] << @col_sep   # will be replaced with a @row_sep after the parts.each loop
        else
          csv << nil
        end
      end

      # This loop is the hot path of csv parsing. Some things may be non-dry
      # for a reason. Make sure to benchmark when refactoring.
      parts.each do |part|
        if in_extended_col
          # If we are continuing a previous column
          if part.end_with?(@quote_char) && part.count(@quote_char) % 2 != 0
            # extended column ends
            csv.last << part[0..-2]
            if csv.last.match?(@parsers[:stray_quote])
              raise MalformedCSVError.new("Missing or stray quote",
                                          lineno + 1)
            end
            csv.last.gsub!(@double_quote_char, @quote_char)
            in_extended_col = false
          else
            csv.last << part << @col_sep
          end
        elsif part.start_with?(@quote_char)
          # If we are starting a new quoted column
          if part.count(@quote_char) % 2 != 0
            # start an extended column
            csv << (part[1..-1] << @col_sep)
            in_extended_col =  true
          elsif part.end_with?(@quote_char)
            # regular quoted column
            csv << part[1..-2]
            if csv.last.match?(@parsers[:stray_quote])
              raise MalformedCSVError.new("Missing or stray quote",
                                          lineno + 1)
            end
            csv.last.gsub!(@double_quote_char, @quote_char)
          elsif @liberal_parsing
            csv << part
          else
            raise MalformedCSVError.new("Missing or stray quote",
                                        lineno + 1)
          end
        elsif part.match?(@parsers[:quote_or_nl])
          # Unquoted field with bad characters.
          if part.match?(@parsers[:nl_or_lf])
            message = "Unquoted fields do not allow \\r or \\n"
            raise MalformedCSVError.new(message, lineno + 1)
          else
            if @liberal_parsing
              csv << part
            else
              raise MalformedCSVError.new("Illegal quoting", lineno + 1)
            end
          end
        else
          # Regular ole unquoted field.
          csv << (part.empty? ? nil : part)
        end
      end
```
(Source: [`ruby/csv.rb#L1255`](https://github.com/ruby/csv/blob/master/lib/csv.rb#L1255))


CSV is and never has been and never will be a single well-defined format, thus, the core
assumption in the library that the world needs to change all files to get read with the standard CSV
library is a kind of hybris.

> CSV maintains a pretty strict definition of CSV taken directly from
> {the RFC}[http://www.ietf.org/rfc/rfc4180.txt].  I relax the rules in only one
> place and that is to make using this library easier.  CSV will parse all valid
> CSV.
>
> (Source: [`ruby/csv.rb#L72`](https://github.com/ruby/csv/blob/master/lib/csv.rb#L72))


Anyways, the point is the CSV library is broken, broken, broken and I do NOT blame the author or authors
I blame all of you. Yes, YOU, all you free-loaders waiting for a miracle.

Again to repeat the author or the authors of the standard CSV library 
deserve our hugs and thank yous for the great work and many hours (for sure many unpaid and volunteered)
put into the CSV library. We are all standing on the shoulders of giants.
 

Now that the giants have left the building any volunteers on adopting the standard CSV library and making it great again? 
Why not start with a custom-built parser that handles more edges cases 
for escaping and quoting rules or more flexible rules / conventions for leading and trailing whitespaces
and so on. Anyone? You might be giant.
