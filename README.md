# timing_attack

Profile web applications, sorting inputs into two categories based on
discrepancies in the application's response time.

If you need a known-vulnerable application for testing and/or development, see
[Camelflage](https://github.com/ffleming/camelflage).

## Installation

```bash
% gem install timing_attack
```

## Usage

```
timing_attack [options] -u <target> <inputs>
    -u, --url URL                    URL of endpoint to profile
    -n, --number NUM                 Requests per input (default: 50)
    -c, --concurrency NUM            Number of concurrent requests (default: 15)
    -t, --threshold NUM              Minimum threshold, in seconds, for meaningfulness (default: 0.025)
    -p, --post                       Use POST, not GET
    -q, --quiet                      Quiet mode (don't display progress bars)
        --brute_force                Brute force mode
        --parameters STR             JSON hash of parameters.  'INPUT' will be replaced with the attack string
        --body STR                   JSON of body paramets to be sent to Typhoeus.  'INPUT' will be replaced with the attack string
        --percentile NUM             Use NUMth percentile for calculations (default: 3)
        --mean                       Use mean for calculations
        --median                     Use median for calculations
    -v, --version                    Print version information
    -h, --help                       Display this screen
```

Note that setting concurrency too high can add significant jitter to your results.  If you know that your inputs contain elements in both long and short response groups but your results are bogus, try backing off on concurrency.  The default value of 15 is a good starting place for robust remote targets, but you might need to dial it back to as far as 1 (especially if you're attacking a single-threaded server)

For the `url`, `body`, and `parameters` options, the string `INPUT` can be included.  It will be replaced with the current test string.

The `body` and `parameters` options take objects serialized with JSON.

### Enumeration

Consider that we we want to gather information from a Rails server running
locally at `http://localhost:3000`.  Let's say that we know the following:
* `charles@poodles.com` exists in the database
* `invalid@fake.fake` does not exist in the database

And we want to know if `bactrian@dev.null` and `alpaca@dev.null` exist in
the database.

We execute (using `-q` to suppress the progress bar)
```bash
% timing_attack -q -u 'http://localhost:3000/timing/conditional_hashing?login=INPUT&password=123' \
                bactrian@dev.null alpaca@dev.null \
                charles@poodles.com invalid@fake.fake
```
```
Short tests:
  invalid@fake.fake             0.0031
  alpaca@dev.null               0.0033
Long tests:
  bactrian@dev.null             0.1037
  charles@poodles.com           0.1040
```

Note that you don't need to know anything about the database when attacking.  It
is, however, nice to have a bit of information as a sanity check.

### Brute Forcing

Consider that we know the endpoint
`http://localhost:3000/timing/string_comparison` is vulnerable to a timing
attack due to an early return in string comparison.  We can attack it with
```bash
timing_attack -u http://localhost:3000/timing/string_comparison \
              --parameters '{"password":"INPUT"}' \
              --brute_force
```
This will attempt a brute-force timing attack against against the `password`
parameter.

## How it works

The various inputs are each thrown at the endpoint `--number` times.  The
`--percentile`th percentile of each input's results is considered the
representative result for that input.  Inputs are then sorted according to
their representative results and the largest spike in their graph is found.
Results then split into short and long groups according to this spike.

The `--mean` flag uses the average of results for a particular input as its
representative result.  The `--median` flag simply uses the 50th percentile.
According to [Crosby, Wallach, and
Reidi](https://www.cs.rice.edu/~dwallach/pub/crosby-timing2009.pdf), results
with percentiles above ~15, median, and mean are all quite noisy, so you should
probably keep `--percentile` low.

I was very surprised to find that I get correct results against remote targets
with `--num` around 20.  Default is 5, as that has been sufficient in my tests
for LAN and local targets.

## Contributing

Bug reports and pull requests are welcome [here](https://github.com/ffleming/timing_attack).

## Disclaimer

timing_attack is quick and dirty.

Also, don't use timing_attack against machines that aren't yours.

## Todo
* Tests
* More intelligent filtering than nth-percentile + spike detection
  * CW&R's box test
