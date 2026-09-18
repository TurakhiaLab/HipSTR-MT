# Contributing to HipSTR-MT

Thank you for your interest in HipSTR-MT. We welcome bug reports, feature
requests, documentation improvements, and code contributions.

HipSTR-MT is a performance fork of [gymrek-lab/HipSTR](https://github.com/gymrek-lab/HipSTR)
that parallelizes HipSTR's per-region processing while producing the same
genotype calls as upstream HipSTR. That guarantee is the core of the project,
and much of this guide is about keeping it.

## Where to ask what

| You have...                                                              | Go to                                                                                   |
|--------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| A bug, crash, or wrong result in HipSTR-MT                               | [HipSTR-MT issues](https://github.com/TurakhiaLab/HipSTR-MT/issues)                     |
| A feature request for this fork (threading, build, performance, packaging) | [HipSTR-MT issues](https://github.com/TurakhiaLab/HipSTR-MT/issues)                   |
| A question about the genotyping model or algorithm, which HipSTR-MT shares with upstream | [HipSTR issues](https://github.com/tfwillems/HipSTR/issues) or hipstrtool@gmail.com |
| A usage question                                                         | The [tutorial](https://hipstr-tool.github.io/HipSTR-tutorial), `./HipSTR-MT --help`, then an issue here |

Please search existing issues before you open a new one.

## Reporting a bug

A useful bug report includes:

- The output of `./HipSTR-MT --version`
- The exact command line, including `--threads`
- Your operating system and compiler (`g++ --version`), and whether you built
  with `make` or installed another way (for example, Bioconda)
- The error message or the relevant part of the log, or, for a wrong result,
  the affected VCF records
- The smallest region BED file that reproduces the problem (one locus is
  ideal), and whether the input data can be shared

Two quick checks make a report much easier to act on:

1. **Does it reproduce with `--threads 1`?** If not, it is probably a
   parallelization bug.
2. **Does it reproduce with upstream HipSTR?** If so, the problem is probably
   in the genotyping code that HipSTR-MT shares with upstream, and it may be
   better reported there.

## Contributing code

1. For anything larger than a small fix, open an issue first so we can agree
   on an approach before you spend time on it.
2. Fork the repository and create a branch from `main`.
3. Build and run the tests:

   ```
   make
   test/run_tests.sh
   ```

4. Open a pull request against `main`. CI builds the project and runs the test
   suite on every pull request, and it must pass before we can merge.

Keep each pull request focused on one change. Code, tests, and documentation
for the same change belong together; unrelated changes belong in separate pull
requests. Write commit messages that explain why a change was made, not only
what changed.

### The correctness requirement

HipSTR-MT must produce the same genotype calls as unmodified upstream HipSTR,
at every thread count. Any change to genotyping, alignment, or numerical code
must keep this true.

- `test/check_correctness.sh` runs HipSTR-MT on the fixtures in `test/` at
  several thread counts and compares each output with a golden VCF using
  `test/compare_vcf_tolerant.py`. It fails on any genotype change, and on any
  derived statistic that drifts by more than 1e-3 (relative). To test other
  thread counts, pass them as arguments: `test/check_correctness.sh 1 16 64`.
- If you fix a correctness bug, add a fixture that fails without your fix, as
  `test/homopolymer/` does for the locus that exposed the `fast_exp_sum`
  accumulator bug. Keep fixtures small (subset the BAM to the affected region),
  and only use data you are allowed to redistribute. A golden VCF should be the
  output of unmodified upstream HipSTR on the same fixture.
- If a change could affect output across the genome, also compare against
  upstream HipSTR on a larger region set, and include the comparator summary in
  your pull request.

### Floating-point code

Several past correctness bugs in this fork came from tiny floating-point
differences. At loci where two alignment paths are almost equally likely, such
a difference can change which candidate alleles are found, and with them the
genotype call. Please:

- Do not narrow `double` values to `float` in log-probability, alignment, or
  accumulation code. The comments on `MatrixChannel` in
  `src/SeqAlignment/HapAligner.h` and on `fast_exp_sum` in `src/mathops.cpp`
  describe two cases where this changed results.
- Keep `-ffp-contract=off` on functions compiled with `target_clones`, so that
  every instruction-set variant gives identical results.
- Prefer vectorizations that reproduce the scalar result exactly. If a change
  alters the order of a floating-point sum, verify it against upstream as
  described above.

### Performance changes

Report wall-clock times before and after the change, with the dataset, region
set, thread count, and hardware. Timings on shared machines can vary by several
percent from run to run, so repeat the measurements and report the spread.
Performance changes must still pass `test/check_correctness.sh`.

### Dependencies and portability

Third-party libraries (htslib, mimalloc, libdeflate, and Taskflow) are vendored
under `lib/` and `taskflow/` so that a plain clone builds. Please discuss new
dependencies in an issue before adding them. HipSTR-MT currently targets Linux
on x86_64 (see Requirements in the [README](README.md)). The build must keep
honoring `CXXFLAGS`, `CPPFLAGS`, and `LDFLAGS` from the environment, which
packaging tools such as Bioconda rely on.

## Relationship to upstream HipSTR

Changes to HipSTR's statistical model belong in
[gymrek-lab/HipSTR](https://github.com/gymrek-lab/HipSTR), not here. We plan to
contribute this fork's changes back upstream, so keeping behavior compatible
with upstream makes that easier.

## AI-assisted contributions

You may use AI coding assistants, but you are responsible for every line you
submit: review it, test it, and be able to explain it during review. Disclose
AI assistance by adding an `Assisted-by:` line to each affected commit message
and to your pull request description, naming the tool and the model version.
For example:

    Assisted-by: Claude:claude-opus-5

This follows the AI policy of htslib (see `lib/htslib/CONTRIBUTING.md`).

## License

HipSTR-MT is licensed under the GNU General Public License v2 (see
[LICENSE](LICENSE)). By submitting a contribution, you agree that it is
licensed under the same terms.
