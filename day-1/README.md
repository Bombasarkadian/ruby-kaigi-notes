# Day 1:
## Matz - Keynote
### Notes
Names are important (what was the proverb?) - especially in programming

Programming is not physical, it's made of concepts, behaviors. Good naming make a code more usable. Bad example would be `yield_self`
```ruby
def yield_self
  yield self
end
```
The name literally describes what it does. Take a look at [feature #14594](https://bugs.ruby-lang.org/issues/14594) about better name for `yield_self`. How about `then`? It would chain nicely. (Matz doesn't really commit into the c ruby too often - he does for MRI - this was first commit in 5 years apart from versions bumps)

Native Americans believe that knowing something/someone true name grants you control over them. The name has a certain power to it and is very important. Project's name can be really important. It can make people love the project. What if Ruby wasn't Ruby? What if Matz didn't come up with "ruby" in 1993... Nowadays ruby will not be a good name. Now we have google and ruby is already occupied by precious stones so people will have trouble finding it. It wasn't issue in 1993. But there are worse examples: "go" (great idea google), "swift" - they have very low "googlability".

The latest trend for better "googlability" is combining words to form a name. Take a look at "Ruby on Rails", "tensor flow" they are unique. Tweaking spelling is another way to achieve this "streem", "jupyter". Some also use exotic names like "hanami" or "kaminari" - you don't know what it does but at least it's easy to find.

Second proverb is "Time is Money" or rather "time is value" - value can often be converted into money. We only have 24 hours every day but majority will be wasting time. We can't avoid things like sleeping, but what about being unproductive? Prioritizing things is very important. Different things are important for everyone.

Ruby helps you use your time better. It is powerful because it has a lot of useful methods. If you need something you usually have a method for this. There are also lots of libraries and frameworks that give you ever more extensions. Ruby community is also big, which gives you a lot of mentors - people who can help you. Ruby is also conscience. On one hand it has a really short syntax but it is easily abusable. We can create very compact programs. "Succinctness is power" ~Paul Graham. It leads us in a direction of much more conscience programs for example through meta programming.

In the ideal world all programs are small enough to grasp. But reality is different. It has to be. But we can leverage some tools to help us. There will be talks on this conference which should touch this topic (for example about rubocop).

Other thing people want from ruby is performance. But it was rewritten to be faster in 1.9. One of the features upcoming in ruby 2.6 is JIT Compiler. It's already being worked on and should make Ruby even faster. Nowadays we all use multi core CPUs. That's why GIL was a major obstacle for boosting ruby performance. That's why people are working on improving concurrency into ruby (Eric Wong). Since time is money we want ruby to be as fast as possible.

Third proverb is from "an old man and a horse" - 塞翁が馬 (is it that one?). Things tend to end up good - life has it's ups and downs. When ruby was first released people didn't know what it was about. Then Ruby on Rails came into existence and ruby was getting a lot of traction. It peaked around 2012 when people were all ecstatic about how ruby is amazing. Now, a few years later the hype stopper. People are saying that ruby is dead, because it's slower and less portable than other languages.

People were complaining that Ruby had it's compatibility gap between 1.8 and 1.9. But the same thing happened to python 2 and 3 and it's been over 10 years since and version 2 is still not deprecated (but will be in the future). Let's say that ruby 3 will be released at some point - there will be some division in community, but we can't get stoped by every obstacle. If you just stop worrying and keep moving forward it makes thing easier.

With all things mentioned earlier ruby gives you high productivity and a great community. It's not something provided by one person for other people but is something created by community from community. The next 3 days of Ruby Kaigi will let us see this.

### Q&A:
1. **Q**: What about type definition? Look at TypeScript and JS. I don't like it. How will this be for ruby?

   **A**: There will never be typing in ruby language definition. There is type annotation in python. Typing is useful for compiled languages, because it helps with finding bugs, but types can actually be recognized by computers automatically, so you won't have to define static typing in language specification.

1. **Q**: Are there plans for high level constructs for ruby? Something like macros?

   **A**: We have to care about compatibility so such things probably won't come before 3.0, but there are no concrete plans for such things.

1. **Q**: Why is it exactly that we don't want type annotations?

   **A**: Because in the future type annotations will most likely be unnecessary and become obsolete so there is no point adding them.

## Aaron Patterson - "Analyzing and Reducing Ruby Memory Usage"
### Intro
Practicing Japanese = 👏

Practicing Japanese on stage = 🙅
### Notes
We'll talk about two memory optimizations in Ruby:
- Memory cache
- Stack VM

Let's start by finding memory usage issues. Since Ruby is written in C we'll look at C programs first. The first way (not very good one) is to look at the code. Better way is to use malloc stack tracing.

In Ruby we can divide memory in two spaces. We've got GC allocated and ObjectSpace allocated memory. There is allocation_tracer for ruby which allows tracking Ruby allocated memory, but not anything outside. For this we can use Malloc Stack Logging.
Malloc logging can grow really fast, as it contains information about all allocations and memory freeing. We can analyze those logs to know how much memory the program was using at the given time. We can also find out what was using the most memory.

We can combine two techniques: "insterumentation" + "" to find out more about moemory usage.

But first let's look at shared string optimization. This allows two different strings to reuse the same part of memory but only if they have the same end. Otherwise there will be a string copy created. Therefore if it's possible it's better to copy the string until the end and not trim it.

Second thing to look at is `$LOADED_FEATURES`. It allows Ruby to keep track of files that were already required. But how do we know that two files are the same ones? We can use different paths for the same location. Also, if it's an array then searching it will be slow. But there is some smart caching going on that speeds up the process.

![cache_generation](media/cache_generation.jpg)

The problem was that the original algorithms was taking a substring. It actually had to create 8 new ruby objects for lookup of simple path like `/a/b/c.rb`. We can use shared strings to speed it up. We can even go further and get rid of ruby objects completely. Since the hash is already written in C we won't use Ruby hash, but instead point directly to C strings.

With this change we've achieved 40% reduction in ruby objects creation. In real world Rails application we've saved over 4% of memory. This change will be available in Ruby 2.6

[Issue #14460](https://bugs.ruby-lang.org/issues/14460). The same thing was proposed in [#8158](https://bugs.ruby-lang.org/issues/8158) (submitted 5 years ago) - always search for existing issues

So how about the stack VM? How do we actually get instructions to execute?

The code has to go through processing made out of parsing and compiling (where the bytecode is generated). There can be optimizations applied in the meantime. The first step is converting code into abstract syntax tree which is actually implemented with ruby objects. We then have to convert those into linked list of operations. The linked list is much easier to work with so there can be more optimizations applied here. The last step will be converting into bytecode which is as simple as representing our list nodes as a list of instructions (in a form of numbers, so we can't use ruby objcts here - we can use their addresses).

We now know how compilation and code execution work.

Since we're using ruby objects addresses in a program we can accidentially modify the original objects. To be sure we don't do this we can duplicate the objects. That's what `frozen_string_literal` doesn. It tells the VM that the strings can't get modified.

We also have to be aware of another problem. Since the strings are both C strings and Ruby objects they can be garbage collected while the C internals still need those. That's why we have mark arrays which keep track if objects are still used. But it duplicates some information, because both the array and instructions list point to the object. Also, the more objects we track the bigger the aray have to be and there can leave us with some unused memory space for the lifetime of the program.

Caould we change array into something else? We can actually mark objects directly from the VM. It's called Direct Marking Technique [#14370](https://bugs.ruby-lang.org/issues/14370). How does in impact our programs. It reduces number of allocated arrays, and number of allocated objects. Overall it saves you around 6% of memory (depending on how much code you have). Also available in Ruby 2.6.

Upgrade to Ruby 2.6!

### Q&A

[No time for Q&A]

## Dmitry Petrashko, Paul Tarjan, Nelson Elhage - "A practical type system for Ruby at Stripe"

### Intro

Focusing at developer productivity. No Rails, a few macroservices, new code in existing services. Bulding upon existing type checkers (such as DRuby, PRuby, RubyDust).

Stripe's type checker is still not open source but there are plans to open source in the future. It's being tested internally. Check it out now at https://sorbet.run

### Notes

How do we start designing a type system for language like Ruby?

It has to be explicit but useful at the same time. It should be rewarding for developers to use it. It's actually easy to build a complex type system, but it's worth trying to make is as simple as possible.

Another important point is compatibility. We'd like our type system to be ruby based and work seamlessly with existing core and libraries. This would also alow gradual adaptation.

Another problem is that we want the type checker to be truthful. Due to Ruby nature some objects can for example become nillable or non-nillable at some point. We should be aware of this. The same goes for dead, unreachable code. Or for union types. Stripe's solution covers all those cases.

Ok, so how do we declare it? Keep in mind we want to use a valid Ruby syntax.

![type_check_compatibility](media/type_check_compatibility.jpg)

There are obviously different strictness levels. The lowest one will only look for syntax errors while the highest one will not allow to execute code with possible types mismatch. It could be used for some critical parts of application.

So does this actually work and is it useful?

It's currently being adopted into Stripe codebase. There were some bugs and suggestions found along the way:

- Typos in error handling - that's sometimes tricky to find in a real application, because your error will not be handled where you want it to. Type ckecking helps with this

- nil checks - some methods require non-nil object so we have to remember to check for nil before using them. Type ckecker helps finding this pattern.

- Using instance variables from static methods - think something like this:

  ```ruby
  class Foo
    def initialize
      @bar = 'bar'
    end

    def self.baz
      @bar
    end
  end
  ```

  It obviously won't work but is a common error

- Unreachable code in unexpected places - sometimes by accident you can write code that won't ever be executed due to how language works. Type checker can find a lot of those

Stripe's type checker can check up to 100k loc/s which is much faster than other similar tools (also in other languages). How was this speed achieved? The checker was actually written in C++ not in Ruby. There is a speed gain but at the cost of loosing metaprogramming suport. Some basic patterns have been reimplemented, but for others the reflection mechanism is used.

Right now there is only a demo mentioned above. Once the final open source version is released it will be notified on the blog. If you have some feedback or ideas send it to sorbet@stripe.com

### Takeaways

* Working type ckecker
* Fast and easy to use
* Will be released soon

### Q&A

1. **Q**: Many Ruby programmers use C where you define type in method definition. Should we have something similar for Ruby?

   **A**: Not like in C

2. **Q**: How does your type annotation go with what Matz talked about type annotations becoming obsolete?

   **A**: Type annotations are useful NOW, it's no point waiting.

3. **Q**: How about typing based on YARD?

   **A**: There is a tool to translate YARD declarations into type annotations

4. **Q**: How about Rails? How about libraries with no type annotations?

   **A**: A lot of community code used in codebase. That's where reflection mechanism can help. It provide at least some partial support until type annotations get added there.

## Bozhidar Batsov - All About RuboCop

### Intro

Great stuff about Bulgaria 🇧🇬

Ruby & Rails Community Style Guides

Actually the first time speaking about RuboCop

### Notes

Why should you use something like rubocop?

* Keeping codebase consistent
* Saving developers time
* Helping Ruby language and community
  * Efficient way to update codebase (cops for finding obsolete code)

>  Lint tools **are not** a replacement for common sense

Over 300 open issues - volunteers needed 💪

@bbatsov's first gem ever - learning to work with RubyGems int the beginning

![learning_rubygems](media/learning_rubygems.jpg)

First versions of rubocop were using regular expressions, but were ultimately updated so it uses proper language parsing now. There is a tool called Ripper in ruby which lets you see how ruby reads your code but it is tied to MRI, not cross platform and had missing documentation. Since it was tied to MRI it was also tied to specific ruby version. It doesn't go well with rubocop tying to be cross platform.

When struggling with Ripper quirks finding `parser` gem, which solved a lot of problems. It was cross-platform, had good documentation and supported a lot of ruby versions. It showed nicely formatted syntax and had an option of rewriting code. But it was a new project with no real world usage yet. Rubocop used parser since version 0.8

Since then it got a lot of useful features like caching, parallel execution etc. It now has very rich configuration options for all available cops which lets you use it in any project you want. And everything is described in the [documentation](http://rubocop.readthedocs.io).

Rubocop has evolved from simple linter into a full featured code formatter.

Writing cops is as simple as writing a few methods so it's easy to add ones you need for your internal projects.

What needs to happen before rubocop 1.0 will be released?

- Removing rails specific cops (and maybe performance ones) from rubocop core
- Moving repository from bbatsov into rubocop hq organization on github
- Reviewing all configs and their defaults
- Comming up with a better process of cops updates (curently, `mry` can be used to ease the process)
- Resolving all potentil breaking changes in cops API

### Q&A

[No time left for Q&A]

## Shibata Hiroshi - RubyGems 3 & 4

### Intro

Ruby core and rubygems developer

Asakusa.rb - Rubyis group in Japan

Slides are [available here](https://www.slideshare.net/hsbt/rubygems-3-4)

### Notes

We'll start with RubyGems version 2.7. Previous versions only have security fixes, while 2.7 is a stable version and gets both bugs and security fixes.

RubyGems supports Ruby down to 1.8, which is a really obsolete version by now. There are a lot of Ruby versions to support, so there is a big build matrix for rubygems on CI.

Support for ruby 1.8 is mainly done by using `respond_to` in a lot of places. There are also two versions of rubygems: independend and bundled in ruby. The second one needs additional checks to make sure that things like bundler are available.

How rubygems installs bundler? If you look at `update_rubygems/setup.rb` you'll se how it adds bundler to default gems.

In the end sometimes rubygems and bundler are installed together while at other times they can be installed separately (depending on versions). This used to cause a lot of problems in certain environments (such as Travis CI). This was ultimately fixed in colaboration with travis team.

RubyGems security is handled by hackerone - 3 people handle vulnerability issues.

Since the project is tightly coupled to specific versions of other projects in causes some problems with maintaining code on git.

What's coming with rubygems 3.0?

Introduction of `Gem::Deprecate`, an easy way to show deprecation warnings to gem's end-users.

Way of searching code in gems with `gemsearch`. Using `all-ruby` to check ruby version compatibility. Those tools can be used to get rid of workarounds for older ruby versions.

Used tools have been updated and support for missing bundler have been added - it could be installed automatically now.

What's up with rubygems 4?

Changes to how gems are installed. Better support for standard libraries defaults.

Rubygems 4 will install to `~/.gem` by default so there will no longer be errors about not having root access. But what about compatibility with things like rbenv?

 ### Q&A

[No time for Q&A]

## Anton Davydov - Architecture of hanami applications

### Intro

Ask questions on mail@davydovanton.com with RubyKaigi2k18 in title

You can read more about [those concepts here](https://github.com/davydovanton/hanami-architecture)

### Notes

Let's start by talking about problems. There are 3 types we can find: management, performance and maintenance. We'll focus on the last one.

It's a hard topic, because things like "good architecture" are hard to measure and agree upon. Let's look at ways to improve existing code by introducing some levels of abstraction to it.

Following good patterns lets us test code much easier. For example by avoiding global state we can use dependency injection

![mvc_patterns](media/mvc_patterns.jpg)

But with abstractions we start to get a lot of objects and a lot of nested names. How do we handle them? And do we really have to initialize those objects every time?

This is solved by next level of abstraction - containers. We can use `dry-containers` for this. This way alll global state is managed from a single place and allows some memorization to improve performance.

But we can improve upon it with interactors and autoinject.

Then, we have models. We don't want logic and validation in models. They will simply save and read the data.

Even higher layer is application. We can actually have different applications that reuse same models and services. Think about separate applications for administrator panel. In typocal hanami approach you'll have different applications sharing common libraries. Instead of actions caling your interactors and executing some logic, you can actoally have different applications doing this. With such approach it's easier to create something like admin panel. But we have to remember about other apps using the same code so we don't break compatibility.

There are some more things you can look on, but your application will be good without them:

* dry-systems
* domains

Your typical hanami project will consist of many applications interacting with some domains which have access to your private code.

The last idea we'll talk about is an event sourcing patern. When creating separate applications the won't access the database directly. Thel will only broadcast events that will be stored in event log. Then, everything from event log will be processed - such action can be read multiple times by different services. For extracting the data we cen use CQRS (Command Query Responsibility Segregation)

> At its heart is the notion that you can use a different model to update information than the model you use to read information
>
> ~Martin Fowler

Hanami implements above ideas which may help writing cleaner, more maintainable code. But it is not a silver bullet and can be really hard to get started with.

### Takeaways

Isolate your business logic

Don't use things you don't undestand (only use what you know you need)

Look at [dry-rb](http://dry-rb.org) for some helpful libraries

### Q&A

1. **Q**: Why do we need repositories in Rails?

   **A**: Rails models have too much logic in them. It would be better to extract this logic elsewhere

2. **Q**: What are pros and cons of repositories?

   **A**: It's easier to find implementation and test things. You can simply stub out repository with whatever you want. On the other side you add one more abstraction to your application. It's also hard to add another concept like this into rails application.

## Lightning talks

### From String#undump to String#unescape

`#undump` is a new method in Ruby 2.5. It's a reverse of `#dump`

`#undump` is useful because it uses the same notation as other languages. It can be used for example for reading nginx logs, because it uses escaped urls there. But undump will not actually work for your regular strings because it must be wrapped in double quotes - otherwise it will raise na exception.

To work around the issue let's make a simple `string_unescape` gem that hacks around this issue and properly wraps sting to make it work with `undump`

### Create libcsv based ruby compatible CSV library

Using libcsv is faster and is a widely used standard. Making it ruby/csv compatible should allow backawards compatibility and keep the code easy to understand. CSV files are used quite often so it's important to work on it's performance.

Due to libcsv implementation, increasing amount of columns will drastically reduce performance, while increasing amount of rows won't have so big of an impact.

Libcsv only supports ASCI, it lacks support for multibyte characters.

### Rib console and plugins to make you happier

Rib is replacement for irb. Written only in Ruby and maintained for 7 years now. It's faster than irb (or at least not slower - looking at you pry). It has some plugins for extra stuff.

`beef` - starting rails console takes a long time. `beef` will make a sound when it starts

It also has support for things like multiline history 👏

It also lets you to seamlessly use your editor (like vim or emacs) for writing longer pieces of code

Some plugins give some basic debugging support.

Find it at https://github.com/godfat/rib

### Improving JSON methods performance

(Not sure what was the actual title)

Parsing json with `JSON.parse` will always return keys as strings. Internally freezing objects let's you remove some memory pressure from strings duplication. It gived 19% performance improvement with a very simple fix.

Other method was `JSON.generate`. There were some unnecesary conversions done internally (changing hash keys to arrays to iterate them). Some other optimizations were added to skip unnecesary methods calls and objects conversion (no need to convert a string that's already ASCII). This change gived 3.5x speed boost.

Those improvements are still waiting to be merged in pull requests [#345](https://github.com/flori/json/pull/345) and [#346](https://github.com/flori/json/pull/346)

### Improve Red Chainer and Numo::NArray performance

[Red Chainer](https://github.com/red-data-tools/red-chainer) was using numpy under the hood. And simply by making them use same types at same places allowed 1.7x speed improvement. Moving where some operations are executed between the libraries allowed further speedup.

### Using Tamashii Connect - Real World with Chatbot

[Tamashii Bluetooth](https://github.com/tamashii-io/tamashii-bluetooth) helps building IoT with ruby and Raspberry Pi.

### Find out potential dead codes from diff

Finding unusable code is hard. There are soultions like debride, but they have a lot of false-positives and there is no DSL for this. We'll only focus on reducing false-positives.

First we'll create list of potentially unused methods and after some changes we'll have another list. We can extract only ones that were introduced by new commit (new list - old list). It makes it easy to use in CI environment. On the other hand we won't be able to check the whole project because it's fitted for regular checks.

It actually also helps with finding refactoring mistakes. If you rename methods and forget to change one name, it should find those methods.

By using diff we can build upon existing solution and reduce false-positives rate

https://github.com/riseshia/deathnote

### Test asynchronous functions with RSpec

Writing and testing synchronous code is easy. Butwhen we start using websockets we enter the asynchronous world. this can be approached with event loop model architecture (EventMachine for ruby). But it brings a new problem. Spending too long in one function will block all subsequent events.

Asynchronous functions run in threads and aren;t blocking the main thread. But how do we test it with RSpec? RSpec has `async_func` keyword which lets us do exaclty this. But it doesn't always work as expected... [out of time]

### To refine or not to refine

Let's talk about Ruby refinements. They allow extending a class locally. They've been an experimental feature until Ruby 2.1.

They can be used to reduce some code dependencies. It let's you avoid monkey patching without changing your existing code. Another use case is adding methods available in new versions to older versions of libraries or language.

But they have some strange behaviors and can cause different problems.

Refinements can be useful when writing gems but not so much for application code.

### 5-Minute Recipe of Todo-app

How do you create a todo app in 5 minutes? Well it's almost impossible so we'll already have an existing app to get started and will copy snippets from there.

[Demo of creating a web application with [hyalite](https://github.com/youchan/hyalite)]

### Symbolic Execution of Ruby Program

Type checkers require extra work while developers are easy and want to write less code. We can extract type definitions from tests, but then we have to write tests. Maybe we can auto generate test cases?

That's what symbolic execution could give us. It builds constraints for all possible cases and then tries to solves for them. [KLEE](https://klee.github.io) - symbolic execution made at Stanford. Using it requires us to compile ruby program into LLVM bytecode. We can then use this to use KLEE on our code.

### Schrödinger branches

[Got a bit lost here 🙉]

Development branch on Ruby has lots of comments that can actualy revert each other. Every Christmas there will be a new branch created and new versions will be created on every one of them. It get's hard to maintain quickly so we have to slowly end life of old branches. 2 versions behind become security maintained only and older ones become EOL. This means that some ruby is indeed dying every year. But how do we know if the old version is actually dead if we never get a commit that indicates this.
