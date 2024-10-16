

## Nargo version:
nargo version = 0.35.0

noirc version = 0.35.0+999071b80e61a37cb994a4e359eabbac27cd53f1

(git version hash: 999071b80e61a37cb994a4e359eabbac27cd53f1, is dirty: false)

## bb version:
bb --version   

0.56.0%

## About

There are 3 main files: one for each branch of the bignum lib that I'm comparing. I've been manually renaming the one that I want to test to be `main.nr`.

Currently:

- main.nr -> mc/refactor
- main.nr.old -> Zac's approach on main
- main.nr.alt -> mc/alt-refactor (where I tried to make sure that the params were definitely evaluated at compile time, by riddling the codebase with an annoying `MOD` numeric generic to convey the modulus_bits of the bignum).


The main files each contain 3 versions of a `main()` function, which I was using to compare gate counts.

The refactored `BigNum` has the same gate counts as Zac's approach.

The refactored `RuntimeBigNum` has much higher gate counts than Zac's runtime bignum approach (which in Zac's impl uses a struct called `BigNumInstance`). THIS GATE DISCREPENCY THE THING I'D LIKE TO UNDERSTAND.

- A `BN254_Fq` `BigNum` main(). Not needed for this debugging. The constraints match Zac's constraints.
- A 2048-bit `BigNum` main(). These constraints match Zac's constraints.
- A 2048-bit `RuntimeBigNum` main().
    - The constraints of this main() should match its `BigNum` counterpart. **This is probably the easiest avenue for debugging the high constraint counts, because it avoids needing to switch between branches of bignum and between the different `main.nr` files in this repo.**
    - The constraints of this main() should _also_ match the `BigNumInstance` approach of Zac's.

Currently the 3rd `main()` is the commented-out one in all files.

## Flow to compare gate counts:

Using the currently-named `main.nr`:

```noir
// THIS IS THE RUNTIME-BIGNUM VERSION

type MyRuntimeBigNum = RuntimeBigNum<18>;

fn main(x: u8) {
    let params = MY_RUNTIME_BIGNUM_PARAMS;
    println(params);
    let _a: MyRuntimeBigNum = RuntimeBigNum::__derive_from_seed(params, [x,2,3,4]);
    let mut a = _a;
    // let mut b = a;
    // let mut c = a;
    for _i in 0..10 {
        // a = unsafe { a.__add(b) };
        // b = unsafe { a.__mul(c) };
        // c = unsafe { c.__mul(c) };
        // RuntimeBigNum::evaluate_quadratic_expression(params, [[a]], [[false]], [[b]], [[false]], [c], [true]);
        a = a.mul(a);
        // b = b.mul(c);
        // c = b.mul(a);
    }
}

#[test]
fn test_2048() {
    main(1);
}
```

Run:

`nargo compile --silence-warnings --force`

**Don't worry about any `bug` warnings: that's because this main() file is badly written.**

`bb gates -b target/use.json -h`


You should see a gate count of `16577`. That's too high. It should be around `11185`.


Then comment-out that test and instead uncomment the 2nd main() in the file:

```noir
// THIS IS THE BIGNUM VERSION

type BigNum_2048 = BigNum<18, Test2048Params>;

fn main(x: u8) {
    let _a: BigNum_2048 = BigNum::__derive_from_seed([x,2,3,4]);
    let mut a = _a;
    // let mut b = a;
    // let mut c = a;
    for _i in 0..10 {
        // a = unsafe { a.__add(b) };
        // b = unsafe { a.__mul(c) };
        // c = unsafe { c.__mul(c) };
        // BigNum::evaluate_quadratic_expression([[a]], [[false]], [[b]], [[false]], [c], [true]);
        a = a.mul(a);
        // b = b.mul(c);
        // c = b.mul(a);
    }
}

#[test]
fn test_2048() {
    // let a: BigNum_2048 = BigNum::__derive_from_seed([1,2,3,4]);
    main(1);
}
```

Do the same:

Run:
`nargo compile --silence-warnings --force`

`bb gates -b target/use.json -h`

This should give a gate count of `11185`. This matches the Zac branch version of BigNum too.