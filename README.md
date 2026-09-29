# Let's Be Rational for QuantLib

A port of Peter Jäckel's *Let's Be Rational* algorithm to QuantLib. It computes Black (lognormal) implied volatility to machine precision in two iterations.

QuantLib already used Jäckel's method for Bachelier (normal) implied vol (`bachelierBlackFormulaImpliedVol`, PR #2028). Its Black solver, `blackFormulaImpliedStdDev`, is a Newton solver whose accuracy target is set in *price* terms. That target breaks down deep out of the money and at very low volatility.

The code lives on [azrabano23/QuantLib@lets-be-rational](https://github.com/azrabano23/QuantLib/tree/lets-be-rational). `lbr.patch` here applies cleanly to QuantLib `master`.

## Results

**Accuracy.** Measured on 570 well-conditioned cases: log-moneyness from ±1e-6 to ±30, total std dev from 1e-8 to 5, calls and puts. Each case is checked against a 50-digit mpmath reference.

| Solver | Failures | Median rel. vol error | Max rel. vol error | Cases with error > 1e-6 |
|---|---|---|---|---|
| **Let's Be Rational (this)** | 0 | 1.5e-16 | 3.5e-14 | 0 |
| `blackFormulaImpliedStdDev` (default accuracy) | 0 | 4.6e-8 | 1.0 | 149 |
| `blackFormulaImpliedStdDev` (accuracy 1e-12) | 0 | 1.2e-13 | 1.0 | 49 |
| Li–Radoičić–Stefanica approximation (default settings) | 84 | 6.0e-9 | 2.2e-2 | 83 |

**Speed.** About 600–800 ns per call, against about 2,000 ns for the existing default (roughly 3× faster). These were measured at -O2 on a loaded shared VM, so read them as ratios. 611 of the 621 cases converge in exactly 2 iterations.

## What's in the implementation

- The normalised Black function, with four evaluation branches: asymptotic expansion, small-t expansion, direct formula, and erfcx.
- A four-region rational-cubic initial guess.
- A safeguarded Householder(3) iteration, capped at two steps.
- Log-moneyness computed with `log1p` near the money. Plain `log(F/K)` loses about 1e-11 there.
- Displacement (shifted lognormal) support, consistent with the existing API.

## Tests

`test-suite/blackformula.cpp` gets six new cases:

- a reference check of the normalised Black function in every branch;
- a round trip over the full grid;
- displacement cases;
- edge cases: intrinsic value, the upper bound, deep OTM, tiny std dev;
- checks that the result moves with every input;
- a comparison against the existing solver.

I mutation-tested the suite. Each of these deliberate breakages makes a test fail:

- swapping Householder for Newton or Halley;
- dropping the erfcx branch;
- a naive intrinsic value;
- reverting to `log(F/K)`.

## Reproduce

```bash
git clone https://github.com/azrabano23/QuantLib -b lets-be-rational && cd QuantLib
cmake -B build -DQL_BUILD_EXAMPLES=OFF -DQL_BUILD_BENCHMARK=OFF && cmake --build build -j --target ql_test_suite
./build/test-suite/quantlib-test-suite --run_test=QuantLibTests/BlackFormulaTests
cd ../quantlib-lets-be-rational/bench && python gen_cases.py && ./build.sh   # see results.txt
```

## Files

- `lbr.patch`: the change, as `git format-patch` output.
- `bench/`: the case generator (mpmath), the C++ benchmark, and the raw results.
- `TALKING-POINTS.md`: the maths, design decisions and trade-offs, with Q&A.
- `upstream/`: the issue and PR text for lballabio/QuantLib.

## Credits

The algorithm is by Peter Jäckel ([Let's Be Rational, Wilmott 2015](https://onlinelibrary.wiley.com/doi/abs/10.1002/wilm.10395)). The ported portions keep his original license notice.
