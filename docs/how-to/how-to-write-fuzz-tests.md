# How to write fuzz tests

This document introduces how to write fuzz tests in GreptimeDB.

## What is a fuzz test
Fuzz test is tool that leverage deterministic random generation to assist in finding bugs. The goal of fuzz tests is to identify inputs generated by the fuzzer that cause system panics, crashes, or unexpected behaviors to occur. And we are using the [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz) to run our fuzz test targets. 

## Why we need them
- Find bugs by leveraging random generation
- Integrate with other tests (e.g., e2e)

## Resources
All fuzz test-related resources are located in the `/tests-fuzz` directory.
There are two types of resources: (1) fundamental components and (2) test targets.

### Fundamental components 
They are located in the `/tests-fuzz/src` directory. The fundamental components define how to generate SQLs (including dialects for different protocols) and validate execution results (e.g., column attribute validation), etc.

### Test targets
They are located in the `/tests-fuzz/targets` directory, with each file representing an independent fuzz test case. The target utilizes fundamental components to generate SQLs, sends the generated SQLs via specified protocol, and validates the results of SQL execution.

Figure 1 illustrates the fundamental components of the fuzz test provide the ability to generate random SQLs. It utilizes a Random Number Generator (Rng) to generate the Intermediate Representation (IR), then employs a DialectTranslator to produce specified dialects for different protocols. Finally, the fuzz tests send the generated SQL via the specified protocol and verify that the execution results meet expectations.
```
                            Rng                                 
                             |                                  
                             |                                  
                             v                                  
                       ExprGenerator                            
                             |                                  
                             |                                  
                             v                                  
               Intermediate representation (IR)                 
                             |                                  
                             |                                  
      +----------------------+----------------------+           
      |                      |                      |           
      v                      v                      v           
MySQLTranslator    PostgreSQLTranslator   OtherDialectTranslator
      |                      |                      |           
      |                      |                      |           
      v                      v                      v           
SQL(MySQL Dialect)         .....                  .....         
      |
      |
      v
  Fuzz Test

```
(Figure1: Overview of fuzz tests)

For more details about fuzz targets and fundamental components, please refer to this [tracking issue](https://github.com/GreptimeTeam/greptimedb/issues/3174).

## How to add a fuzz test target

1. Create an empty rust source file under the `/tests-fuzz/targets/<fuzz-target>.rs` directory.

2. Register the fuzz test target in the `/tests-fuzz/Cargo.toml` file.

```toml
[[bin]]
name = "<fuzz-target>"
path = "targets/<fuzz-target>.rs"
test = false
bench = false
doc = false
```

3. Define the `FuzzInput` in the `/tests-fuzz/targets/<fuzz-target>.rs`.

```rust
#![no_main]
use libfuzzer_sys::arbitrary::{Arbitrary, Unstructured};

#[derive(Clone, Debug)]
struct FuzzInput {
    seed: u64,
}

impl Arbitrary<'_> for FuzzInput {
    fn arbitrary(u: &mut Unstructured<'_>) -> arbitrary::Result<Self> {
        let seed = u.int_in_range(u64::MIN..=u64::MAX)?;
        Ok(FuzzInput { seed })
    }
}
```

4. Write your first fuzz test target in the `/tests-fuzz/targets/<fuzz-target>.rs`.

```rust
use libfuzzer_sys::fuzz_target;
use rand::{Rng, SeedableRng};
use rand_chacha::ChaChaRng;
use snafu::ResultExt;
use sqlx::{MySql, Pool};
use tests_fuzz::fake::{
    merge_two_word_map_fn, random_capitalize_map, uppercase_and_keyword_backtick_map,
    MappedGenerator, WordGenerator,
};
use tests_fuzz::generator::create_expr::CreateTableExprGeneratorBuilder;
use tests_fuzz::generator::Generator;
use tests_fuzz::ir::CreateTableExpr;
use tests_fuzz::translator::mysql::create_expr::CreateTableExprTranslator;
use tests_fuzz::translator::DslTranslator;
use tests_fuzz::utils::{init_greptime_connections, Connections};

fuzz_target!(|input: FuzzInput| {
    common_telemetry::init_default_ut_logging();
    common_runtime::block_on_write(async {
        let Connections { mysql } = init_greptime_connections().await;
            let mut rng = ChaChaRng::seed_from_u64(input.seed);
            let columns = rng.gen_range(2..30);
            let create_table_generator = CreateTableExprGeneratorBuilder::default()
                .name_generator(Box::new(MappedGenerator::new(
                    WordGenerator,
                    merge_two_word_map_fn(random_capitalize_map, uppercase_and_keyword_backtick_map),
                )))
                .columns(columns)
                .engine("mito")
                .if_not_exists(if_not_exists)
                .build()
                .unwrap();
            let ir = create_table_generator.generate(&mut rng);
            let translator = CreateTableExprTranslator;
            let sql = translator.translate(&expr).unwrap();
            mysql.execute(&sql).await
    })
});
```

5. Run your fuzz test target

```bash
    cargo fuzz run <fuzz-target> --fuzz-dir tests-fuzz
```

For more details, please refer to this [document](/tests-fuzz/README.md).