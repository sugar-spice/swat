# ABSTRACT

SWAT is Simple Web Application Test ( Tool )

# SYNOPSIS

SWAT is Simple Web Application Test ( Tool )

    $  swat examples/google/ google.ru
    /home/vagrant/.swat/reports/google.ru/00.t ..
    # start swat for google.ru//
    # try num 2
    ok 1 - successfull response from GET google.ru/
    # data file: /home/vagrant/.swat/reports/google.ru///content.GET.txt
    ok 2 - GET / returns 200 OK
    ok 3 - GET / returns Google
    1..3
    ok
    All tests successful.
    Files=1, Tests=3, 12 wallclock secs ( 0.00 usr  0.00 sys +  0.02 cusr  0.00 csys =  0.02 CPU)
    Result: PASS

# WHY

I know there are a lot of tests tool and frameworks, but let me  briefly tell _why_ I created swat.
As devops I update a dozens of web application weekly, sometimes I just have _no time_ sitting and wait 
while dev guys or QA team ensure that deploy is fine and nothing breaks on the road. 
So I need a **tool to run smoke tests against web applications**. 
Not tool only, but the way to **create such a tests from the scratch in way easy and fast enough**. 

So this how I came up with the idea of swat. If I was a marketing guy I'd say that swat:

- is easy to use and flexible tool to run smoke tests against web applications
- is [curl](http://curl.haxx.se/) powered and [TAP](https://testanything.org/) compatible
- leverages famous [prove](http://search.cpan.org/perldoc?prove) utility
- has minimal dependency tree  and probably will run out of the box on most linux environments, provided that one has perl/bash/find/curl by hand ( which is true  for most cases )
- has a simple and yet powerful DSL allow you to both run simple tests ( 200 OK ) or complicated ones ( using curl api and perl functions calls )
- is daily it/devops/dev helper with low price mastering ( see my tutorial )
- and yes ... swat is fun :)

# Tutorial

## Install swat

### stable release

    sudo cpan install swat

### developer release

    # developer release might be untested and unstable
    sudo cpanm --mirror-only --mirror https://stratopan.com/melezhik/swat-release/master swat

Once swat is installed you have **swat** command line tool to run swat tests, but before do this you need to create them.

## Create tests

    mkdir  my-app/ # create a project root directory to contain tests

    # define http URIs application should response to

    mkdir -p my-app/hello # GET /hello
    mkdir -p my-app/hello/world # GET /hello/world

    # define the content the expected to return by requested URIs

    echo 200 OK >> my-app/hello/get.txt
    echo 200 OK >> my-app/hello/world/get.txt

    echo 'This is hello' >> my-app/hello/get.txt
    echo 'This is hello world' >> my-app/hello/world/get.txt

## Run tests

    swat ./my-app http://127.0.0.1

# DSL

Swat DSL consists of 2 parts. Routes and Swat Data.

## Routes

Routes are http resources a tested web application should have.

Swat utilize file system to get know about routes. Let we have a following project layout:

    example/my-app/
    example/my-app/hello/
    example/my-app/hello/get.txt
    example/my-app/hello/world/get.txt

When you give swat a run

    swat example/my-app 127.0.0.1

It will find all the _directories with get.txt|post.txt files inside_ and "create" routes:

    GET hello/
    GET hello/world

When you are done with routes you need to set swat data.

## Swat data

Swat data is DSL to describe/generate validation checks you apply to content returned from web application.
Swat data is stored in swat data files, named get.txt or post.txt. 

The process of validation looks like:

- Swat recursively find files named **get.txt** or **post.txt** in the project root directory to get swat data.
- Swat parse swat data file and _execute_ entries found. At the end of this process swat creates a _final check list_ with 
["Check Expressions"](#check-expressions).
- For every route swat makes http requests to web application and store content into text file 
- Every line of text file is validated by every item in a _final check list_

_Objects_ found in test data file are called _swat entries_. There are _3 basic type_ of swat entries:

- Check Expressions
- Comments
- Perl Expressions and Generators

### Check Expressions

This is most usable type of entries you  may define at swat data file. _It's just a string should be returned_ when swat request a given URI. Here are examples:

    200 OK
    Hello World
    <head><title>Hello World</title></head>

Using regexps

Regexps are check expresions with the usage of <perl regular expressions> instead of plain strings checks.
Everything started with `regexp:` marker would be treated as perl regular expression.

    # this is example of regexp check
    regexp: App Version Number: (\d+\.\d+\.\d+)

### Comments

Comments entries are lines started with `#` symbol, swat will ignore comments when parse swat data file. Here are examples.

    # this http status is expected
    200 OK
    Hello World # this string should be in the response
    <head><title>Hello World</title></head> # and it should be proper html code

### Perl Expressions

Perl expressions are just a pieces of perl code to _get evaled_ by swat when parsing test data files.

Everything started with `code:` marker would be treated by swat as perl code to execute.
There are a _lot of possibilities_! Please follow [Test::More](https://metacpan.org/pod/search.cpan.org#perldoc-Test::More) documentation to get more info about useful function you may call here.

    code: skip('next test is skipped',1) # skip next check forever
    HELLO WORLD


    code: skip('next test is skipped',1) unless $ENV{'debug'} == 1  # confitionally skip this check
    HELLO SWAT

# Generators

Swat entries generators is the way to _create new swat entries on the fly_. Technically specaking it's just a perl code which should return an array reference:
Generators are very close to perl expressions ( generators code is alos get evaled ) with maijor difference:

Value returned from generator's code should be  array reference. The array is passed back to swat parser so it can create new swat entries from it. 

Generators entries start with `:generator` marker. Here is example:

    # Place this in swat pattern file
    generator: [ qw{ foo bar baz } ]

This generator will generate 3 swat entries:

    foo
    bar
    baz

As you can guess an array returned by generator should contain _perl strings_ representing swat entries, here is another example:
with generator producing still 3 swat entites 'foo', 'bar', 'baz' :

    # Place this in swat pattern file
    generator: my %d = { 'foo' => 'foo value', 'bar' => 'bar value' }; [ map  { ( "# $_", "$data{$_}" )  } keys %d  ] 

This generator will generate 3 swat entities:

    # foo
    foo value
    # bar
    bar value

There is no limit for you! Use any code you want with only requiment - it should return array reference. 
What about to validate web application content with sqlite database entries?

    # Place this in swat pattern file
    generator: \
    
    use DBI; \
    my $dbh = DBI->connect("dbi:SQLite:dbname=t/data/test.db","",""); \
    my $sth = $dbh->prepare("SELECT name from users"); \
    $sth->execute(); \
    my $results = $sth->fetchall_arrayref; \
    
    [ map { $_->[0] } @${results} ]

See examples/swat-generators-sqlite3 for working example

# Multiline expressions

Sometimes code looks more readable when you split it on separate chunks. When swat parser meets  `\` symbols it postpone entity execution and
and next line to buffer. Once no `\` occured swat parser _execute_ swat entry.

Here are some exmaples:

    # Place this in swat pattern file
    generator:                  \
    my %d = {                   \
        'foo' => 'foo value',   \
        'bar' => 'bar value',   \
        'baz' => 'baz value'    \
    };                          \
    [                                               \
        map  { ( "# $_", "$data{$_}" )  } keys %d   \
    ]                                               \

    # Place this in swat pattern file
    generator: [            \
            map {           \
            uc($_)          \
        } qw( foo bar baz ) \
    ]

    code:                                                       \
    if $ENV{'debug'} == 1  { # confitionally skip this check    \
        skip('next test is skipped',1)                          \ 
    } 
    HELLO SWAT

Multiline expressions are only allowable for perl expressions and generators 

# Post requests

Name swat data file as post.txt to make http POST requests.

    echo 200 OK >> my-app/hello/post.txt
    echo 200 OK >> my-app/hello/world/post.txt

You may use curl\_params setting ( follow ["Swat Settings"](#swat-settings) section for details ) to define post data, there are some examples:

- `-d` - Post data sending by html form submit.

         # Place this in swat.ini file or sets as env variable:
         curl_params='-d name=daniel -d skill=lousy'

- `--data-binary` - Post data sending as is.

         # Place this in swat.ini file or sets as env variable:
         curl_params=`echo -E "--data-binary '{\"name\":\"alex\",\"last_name\":\"melezhik\"}'"`
         curl_params="${curl_params} -H 'Content-Type: application/json'"

# Generators and Perl Expressions Scope

Swat uses _perl string eval_ when process generators and perl expressions code, be aware of this. 
Follow [http://perldoc.perl.org/functions/eval.html](http://perldoc.perl.org/functions/eval.html) to get more on this.

# Swat Settings

Swat comes with settings defined in two contexts:

- Environmental Variables
- swat.ini files

## Environmental Variables

Defining a proper environment variables will provide swat settings.

- `debug` - set to `1` if you want to see some debug information in output, default value is `0`
- `debug_bytes` - number of bytes of http response  to be dumped out when debug is on. default value is `500`
- `ignore_http_err` - ignore http errors, if this parameters is off (set to `1`) returned  _error http codes_ will not result in test fails, 
useful when one need to test something with response differ from  2\*\*,3\*\* http codes. Default value is `0`
- `try_num` - number of http requests  attempts before give it up ( useless for resources with slow response  ), default value is `2`
- `curl_params` - additional curl parameters being add to http requests, default value is `""`, follow curl documentation for variety of values for this
- `curl_connec_timeout` - follow curl documentation
- `curl_max_time` - follow curl documentation
- `port`  - http port of tested host, default value is `80`

## Swat.ini files

Swat checks files named `swat.ini` in the following directories

- **~/swat.ini**
- **$project\_root\_directory/swat.ini**
- **$route\_directory/swat.ini**

Here are examples of locations of swat.ini files:

     ~/swat.ini # home directory swat.ini file
     my-app/swat.ini # project_root directory swat.ini file
     my-app/hello/get.txt
     my-app/hello/swat.ini # route directory swat.ini file ( route hello )
     my-app/hello/world/get.txt
     my-app/hello/world/swat.ini # route directory swat.ini file ( route hello/world )

Once file exists at any location swat simply **bash sources it** to apply settings.

Thus swat.ini file should be bash file with swat variables definitions. Here is example:

    # the content of swat.ini file:
    curl_params="-H 'Content-Type: text/html'"
    debug=1

## Settings priority table

Here is the list of settings/contexts  in priority ascending order:

    | context                 | location                | priority  level |
    | ------------------------|------------------------ | --------------- |
    | swat.ini file           | ~/swat.ini              |               1 |
    | environmental variables | ---                     |               2 |
    | swat.ini file           | project root directory  |               3 |
    | swat.ini file           | route directory         |               4 |

Swat processes settings _in order_. For every route found swat:

- Clear all settings
- Apply settings from environmental variables ( if any given )
- Apply settings from swat.ini file in home directory ( if any given )
- Apply settings from swat.ini file in project root directory ( if any given )
- And finally apply settings from swat.ini file in route directory ( if any given )

# TAP

Swat produce output in [TAP](https://testanything.org/) format , that means you may use your favorite tap parsers to bring result to
another test / reporting systems, follow TAP documentation to get more on this. Here is example for converting swat tests into JUNIT format

    swat $project_root $host --formatter TAP::Formatter::JUnit

See also ["Prove settings"](#prove-settings) section.

# Command line tool

Swat is shipped as cpan package, once it's installed ( see ["Install swat"](#install-swat) section ) you have a command line tool called **swat**, this is usage info on it:

    swat project_dir URL <prove settings>

- **URL** - is base url for web application you run tests against, you need defined routes which will be requested against URL, see DSL section.
- **project\_dir** - is a project root directory

## Prove settings

Swat utilize [prove utility](http://search.cpan.org/perldoc?prove) to run tests, so all the swat options _are passed as is to prove utility_.
Follow [prove](http://search.cpan.org/perldoc?prove) utility documentation for variety of values you may set here.
Default value for prove options is  `-v`. Here is another examples:

- `-q -s` -  run tests in random and quite mode

# Examples

./examples directory contains examples of swat tests for different cases. Follow README.md files for details.

# Dependencies

Not that many :)

- perl 
- curl 
- bash
- find
- head

# AUTHOR

[Aleksei Melezhik](mailto:melezhik@gmail.com)

# Thanks

To the authors of ( see list ) without who swat would not appear to light

- perl
- curl
- TAP
- Test::More
- prove
