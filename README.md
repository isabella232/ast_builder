# AstBuilder

AstBuilder (AST Amended) is a tool to make it easier to work with and compose S-Expressions and other AST related tasks in Ruby.

It relies heavily on RuboCop functionality, most notably of which the [`NodePattern`][0] meta-language for AST construction and matching.

## Usage

### Building a Matcher

AstBuilder supports direct strings, which is mostly done to generate ASTs more quickly from static strings.

```ruby
AstBuilder.build('1 + 1')
```

The more typical usage for AstBuilder involves passing it a block:

```ruby
AstBuilder.build { s(:node_type, s(:child_node, '...')) }
=> #<AstBuilder::Builder:0x00007fe2fba18390 @ast=s(:node_type, s(:child_node, "..."))>
```

AstBuilder comes with several extensions to the standard `AST::Sexp` syntax's `s` method, as we'll be going over here.

#### Expanded Nodes

Ruby constant strings and code that's mostly static can be a bit cumbersome to write out:

```ruby
puts AstBuilder.build('A::B::C = 1')
(casgn
  (const
    (const nil :A) :B) :C
  (int 1))
=> nil
```

If you wanted to actually capture or use wildcards from [`NodePattern`][0] in that statement, it becomes more difficult:

```ruby
puts AstBuilder.build('A::B::C = ...')
AstBuilder::InvalidCode: The following node is invalid:
  'A::B::C = ...'
from /Users/baweaver/Development/ast_builder/lib/ast_builder/builder.rb:189:in 'parse'
```

AstBuilder isn't quite smart enough to be able to tell the difference between a meta-character from [`NodePattern`][0] and a regular Ruby token. This is why the builder blocks are used, but in typical construction you would need to write out the expression by hand.

With AstBuilder you can keep those larger nodes as normal Ruby:

```ruby
puts AstBuilder.build { s(:casgn, expand('A::B'), :C, expand('1')) }
(casgn
  (const
    (const nil :A) :B) :C
  (int 1))
=> nil
```

#### Literal Tokens

If [`NodePattern`][0] tokens aren't valid Ruby, how does one evaluate them into an s-expression tree? `RuboCop::AST::Node`, when coercing to a `String`, will call `inspect` on items it doesn't know how to coerce.

`AstBuilder::LiteralToken` defines this in such a way to not have quotation marks, allowing for a psuedo-interpolation of the meta-language:

```ruby
puts AstBuilder.build { s(:casgn, expand('A::B'), :C, literal('...')) }
(casgn
  (const
    (const nil :A) :B) :C ...)
=> nil
```

#### Captures

If you want to capture a node, you would use `$` in NodePattern. In AstBuilder we use `capture` to simulate the same:

```ruby
puts AstBuilder.build { s(:casgn, expand('A::B'), :C, capture(literal('...'))) }
(casgn
  (const
    (const nil :A) :B) :C $...)
=> nil
```

There's the shorter version, `capture_children`, for this same task:

```ruby
puts AstBuilder.build { s(:casgn, expand('A::B'), :C, capture_children) }
(casgn
  (const
    (const nil :A) :B) :C $(...))
=> nil
```

#### Matching

`matching` will allow you to go beyond traditional [`NodePattern`][0] expressions and start inlining arbitrary functions to your matches, allowing more fine-grained control of matches.

It supports both block functions and items responding to `===` like `Integer` and instances of `RegExp`:

```ruby
# Matching with a block
builder = AstBuilder.build {
  assigns(:value, s(:str, capture_matching { |s| s.length > 3 }))
}

builder.match('value = "abc123"')
=> "abc123"

# Non-matching block
builder = AstBuilder.build {
  assigns(:value, s(:str, capture_matching { |s| s.length > 6 }))
}

builder.match('value = "abc123"')
=> nil

# Matching with something that responds to `===`:
builder = AstBuilder.build {
  assigns(:value, s(:str, capture_matching(/^abc/)))
}

builder.match('value = "abc123"')
=> "abc123"
```

### Matching Values and ASTs

An `AstBuilder::Builder` can be coerced into a `RuboCop::NodePattern`, which can be used in the same fashion.

> Note: RuboCop has some slight inconsistencies with how it represents `nil`, which is why when
> coercing to a RuboCop::NodePattern syntax they're replaced with `nil?`.

```ruby
AstBuilder.build { s(:casgn, expand('A::B'), :C, capture_children) }.to_cop
=> #<RuboCop::NodePattern:0x00007fe2fca004d8>
```

This means that you can use `match` just the same as you would on a `NodePattern`, but AstBuilder surfaces this functionality as we'll see in the next section.

#### match

AstBuilder can directly match by coercing its internal state into a NodePattern:

```ruby
ast_builder_build = AstBuilder.build { s(:casgn, expand('A::B'), :C, capture_children) }

# Using AstBuilder to build a quick AST mock:
match_data = ast_builder_build.match(AstBuilder.build('A::B::C = 1').to_ast)
=> s(:int, 1)

# Against a string:
ast_builder_build.match('A::B::C = 1')
=> s(:int, 1)
```

This can also be used against nodes in a RuboCop rule match:

```ruby
module RuboCop
  module Cop
    # Grouping name of the Cop
    module Deprecations
      # Name of the check
      class AbcDeprecation < RuboCop::Cop::Cop

        # RuboCop takes a default message for errors
        MSG = '`A::B::C` is deprecated, use `D::E::F` instead.'

        # Saving the matcher as a constant allows for easier reuse if you use
        # autocorrect later, as well as giving a consistent theme across your
        # matchers.
        AST_MATCH = AstBuilder.build { s(:casgn, expand('A::B'), :C, capture_children) }

        # [...]

        # Node matches work with the node type you're planning to capture. In this
        # case we're trying to capture a `casgn` from the top type of the expression:
        #
        #     (casgn
        #       (const
        #         (const nil :A) :B) :C
        #       (int 1))
        #
        # RuboCop matchers are all on_(type of node) for method names.
        def on_casgn(node)
          # Unless our node matches that expression, bail out.
          return false unless AST_MATCH.match(node)

          # If it did, add an offense so RuboCop knows it's bad.
          add_offense(node)
        end

        def autocorrect(node)
          # RuboCop passes this function to a batch that runs all the given
          # correctors for the code, hence returning a lambda here.
          -> corrector {
            matches = AST_MATCH.match(node)

            # 1. Matches are wrapped in an Enumerator, as there can be multiple
            # 2. Then the value you want may be in an S-Expression
            # 3. An S-Expression can have multiple children, so it's returned as an Array
            # 4. Getting the first value specifically gives us 1, the set value
            #
            #       [s()] -> s() -> [1]  ->  1
            value = matches.first.children.first

            # Now we could always use the actual s-expression from step 2 here
            # with `matches.first.location.source`, which is handy when we're
            # not dealing with only one integer.
            new_code = "D::E::F = #{value}"

            # You can use a few things here, like `insert_before` or after or
            # other expressions. See the corrector source for more ideas:
            #
            #   https://github.com/rubocop-hq/rubocop/blob/master/lib/rubocop/cop/corrector.rb
            #
            # In this case we're replacing the entire node where it's expression is,
            # or rather the entire thing, with our new code.
            corrector.replace(node.location.expression, new_code)
          }
        end
      end
    end
  end
end
```

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'ast_builder'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ast_builder

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/square/ast_builder. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## Code of Conduct

Everyone interacting in the AstBuilder project’s codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/square/ast_builder/blob/master/CODE_OF_CONDUCT.md).

[0]: https://www.rubydoc.info/gems/rubocop/RuboCop/NodePattern "RuboCop NodePattern"

## License

Copyright 2019 Square, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
