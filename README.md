# logjam

[![Build Status][gh-actions-badge]][gh-actions] [![LFE Versions][lfe badge]][lfe] [![Erlang Versions][erlang badge]][versions] [![Tags][github tags badge]][github tags]

[![Project Logo][logo]][logo-large]

*A custom formatter for the Erlang logger application that produces human-readable output*

## Why?

The default formatter for Erlang is very difficult to read at a glance making troubleshooting and debugging of applications via log output during development phases rather cumbersome or even difficult.

## Usage

It is recommended that if you are providing a library, you do not add this
project as a dependency. A code formatter of this kind should be added to a
project in its release repository as a top-level final presentational concern.

Once the project is added, replace the formatter of the default handler (or add a custom handler) for structured logging to your `sys.config` file:

```erlang
[
 {kernel, [
    {logger, [
        {handler, default, logger_std_h,
         #{formatter => {logjam, #{
            map_depth => 3,
            term_depth => 50
          }}}
        }
    ]},
    {logger_level, info}
 ]}
].
```

The logging output will then be supported. Calling the logger like:

```erlang
?LOG_ERROR(
    #{type => event, in => config, user => #{name => <<"bobby">>, id => 12345},
      action => change_password, result => error, details => {entropy, too_low},
      txt => <<"user password not strong enough">>}
)
```

Will produce a single log line like:
```
when=2018-11-15T18:16:03.411822+00:00 level=error pid=<0.134.0>
at=config:update/3:450 user_name=bobby user_id=12345 type=event
txt="user password not strong enough" result=error in=config
details={entropy,too_low} action=change_password
```

Do note that the `user` map gets flattened such that `#{user => #{name => bobby}}` gets
turned into `user_name=bobby`, ensuring that various subfields in distinct maps
will not clash.

The default template supplied with the library also includes optional fields for
identifiers as used in distributed tracing framework which can be set in the metadata
for the logger framework, either explicitly or as a process state. The fields are:

- `id` for individual request identifiers
- `parent_id` for the event or command that initially caused the current logging event to happen
- `correlation_id` for groupings of related events


Logs that are not reports (maps) are going to be formatted and handled such that they can be
put inside a structured log. For example:

```erlang
?LOG_INFO("hello ~s", ["world"])
```

Will result in:

```
when=2018-11-15T18:16:03.411822+00:00 level=info pid=<0.134.0>
at=some:code/0:15 unstructured_log="hello world"
```

Do note that if you are building a release, you will need to manually add
the `logjam` dependency to your `relx` configuration, since it is
technically not a direct dependency of any application in your system.

Test
----

    $ rebar3 check

Features
--------

- Printing rules similar to the default Erlang logger formatter, but extended for
  binary values that can be represented as text. I.e. rather than `<<"hello">>`, the
  value `hello` will be output. A non-representable value will revert to `<<...>>`
- Linebreaks are escaped to ensure all logs are always on one line, and strings that
  contain spaces or equal signs (` ` and `=`) are quoted such that
  `"key=name"="hello world"` to be clear.
- Term depth applies on a per-term basis before a data structure is elided with `...`
- Map depth is controllable independently to deal with recursion vs. complexity of terms
- Colored output can be enabled with `colored => true`. One can color certain parts of
  the output using `colored_start` and `colored_end` in `template`. Per-level colors
  can be configured with `colored_{log level}`.

Caveats
-------

- No max line length is enforced at the formatter level, since the ordering of terms
  in maps is not defined and it could be risky to cut logs early. If a max line length
  is to be enforced, you should wrap this formatter into your own.
- Escaping of keys does not carry well to nested maps. I.e. the map
  `#{a_b => #{"c d" => x}}` is not well supported: `a_b_"c d"=x` will be returned, which
  is nonsensical. For nested maps, you have the responsibility of ensuring composability.
- The transformations to the log line format is not lossless; it is not serialization.
  Information is lost regarding whether the initial term was a binary, a string, or an
  atom. Similarly, naming a key `user_password` may make it seem like the `user` map
  leaks a `password` field, but it is an unrelated field that looks similar due to flattening.
  If this is unacceptable, you might want to choose another structured log format such
  as JSON.

Roadmap
-------

- integration tests
- add example basic usage
- add example usage with optional tracing for IDs
- clean up test suites
- incorporating lager's safer truncating logic (might be a breaking change prior to 1.0.0)

Changelog
---------

- 0.1.1: added optionally colored logs (thanks @pfenoll)

<!-- Named page links below: /-->

[logo]: priv/images/logjam-crop-small.png
[logo-large]: priv/images/logjam.jpg
[screenshot]: priv/images/screenshot.png
[org]: https://github.com/lfex
[github]: https://github.com/lfex/logjam
[gitlab]: https://gitlab.com/lfex/logjam
[gh-actions-badge]: https://github.com/lfex/logjam/workflows/ci%2Fcd/badge.svg
[gh-actions]: https://github.com/lfex/logjam/actions
[lfe]: https://github.com/rvirding/lfe
[lfe badge]: https://img.shields.io/badge/lfe-1.3.0-blue.svg
[erlang badge]: https://img.shields.io/badge/erlang-17.5%20to%2022.0-blue.svg
[versions]: https://github.com/lfex/logjam/blob/master/.travis.yml
[github tags]: https://github.com/lfex/logjam/tags
[github tags badge]: https://img.shields.io/github/tag/lfex/logjam.svg
[github downloads]: https://img.shields.io/github/downloads/lfex/logjam/total.svg
