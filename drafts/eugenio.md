# Eugenio: A translator for arbitrary boolean statements to polynomial identities

## Abstract

A software engineer view of a set of invariants over some data will naturally be thought and expressed as boolean statements (BS). A zero knowledge proof (ZKP) circuit can be seen as a set of invariants  over some data, the witness. However, ZKP arithmetizations are expressed as polynomial identities (PI) over the witness elements. Currently, a lot of the work of software engineer building ZKP circuits it is to manually convert boolean statements into polynomial identities, this is time consuming and can be difficult to do without introducing soundness bugs.

We propose a translator that automatically can convert any boolean statement into a set of polynomial identities that are zero if and only if the boolean statement is truth for values in the variables.

## Basis of operation

The translator works in two steps. First the boolean statements are converted to polynomial identities with an added operator MI (for multiplicative inverse). In a second phase the operator is eliminated by creating virtual signals.

The MI operator is `MI(x) = x^-1 when x != 0, 0 when x = 0`.

## Boolean representation in PI

The main representation of a BS in a PI is what we call anti-booly. The PI will be zero if and only if the statement is true. This is the main representation because this is what ZKP arithmetizations expect.

For example, for the BS `a == b` the anti-booly PI is `a - b`

However, there is another representation that is used in sub-expressions of a PI. We call it OneZero, where the PI is one if the boolean statement is true, and zero if false, other values are invalid.

For example for a BS `if a then b` it is more practical to convert to a PI if `a`is in OneZero representation. Assuming that, `a*b` would be a PI for that BS, and the final representation will be in the representation of `b`.

Having the `MI` operator, we could convert easily between representations.

 + if `a` is in a anti-booly representation, the OneZero Representation will be `1 - (a * MI(a))`.
 + if `a` is in a OneZero reprentation, the anti-booly representation will be simply `1 - a`.

## Translator algorithm

We will present a `pseudo-rust`implementation of the translator

Having a source language, with the following operators:
	+ `eq`
	+ `neq`
	+ `and`
	+ `or`
	+ `not`
	+ `if a then b`
	+ `if a then b else c`
	+ `+`
	+ `-`
	+ `*`
	+ `select`, ternary operator, like ` a ? b : c` in C language

And the target language being PI with `MI` operator.

We will use the `S`type for an AST of the source language and `T` for a type for an SAT of the target language.

### First phase

We present the algorithms to translate each of the source operators to the target language. The translator for the boolean expressions will return both representations, anti-booly and OneZero of the resulting PI. For the arithmetic expressions with root in `+`, `-`, `*` or select it will return an arithmetic equivalant representation.

The target type `T` has methods to cast between representations.

#### Equality

```
fn translate_eq(
        lhs: S,
        rhs: S,
    ) -> CompilationResult {
        assert!(lhs.is_arith());
        assert!(rhs.is_arith());

        let lhs = self.translate_arith(lhs);
        let rhs = self.translate_arith(rhs);

        let expr = lhs - rhs;

        CompilationResult {
            anti_booly: expr,
            one_zero: expr.cast_one_zero(),
        }
    }
```

#### Inequality

```
fn translate_neq(
        &self,
        lhs: S,
        rhs: S,
    ) -> CompilationResult {
        assert!(lhs.is_bool());
        assert!(rhs.is_bool());

        let lhs = self.translate_arith(lhs);
        let rhs = self.translate_arith(rhs);

        let expr = 1 - ((lhs - rhs)*MI(lhs - rhs));

        CompilationResult {
            anti_booly: expr,
            one_zero: 1 - expr,
        }
    }
```

#### And

```
fn translate_and(
        &self,
        lhs: S,
        rhs: S,
    ) -> CompilationResult {
        assert!(lhs.is_bool());
        assert!(rhs.is_bool());

        let lhs = self.translate_bool(lhs);
        let rhs = self.translate_bool(rhs);

        let expr = lhs.one_zero * rhs.one_zero;

        CompilationResult {
            anti_booly: expr.cast_anti_booly(),
            one_zero: expr,
        }
    }
```

#### Or

```
fn translate_or(
        lhs: S,
        rhs: S,
    ) -> CompilationResult {
        assert!(lhs.is_bool());
        assert!(rhs.is_bool());

        let lhs = self.translate_bool(lhs);
        let rhs = self.translate_bool(rhs);

        let expr = lhs.anti_booly * rhs.anti_booly;

        CompilationResult {
            anti_booly: expr,
            one_zero: expr.cast_one_zero(),
        }
    }
```

#### Not

```
fn translate_not(
        sub: S,
    ) -> CompilationResult {
        assert!(sub.is_bool());

        let sub = self.translate_bool(sub);

        let expr = lhs.anti_booly * rhs.anti_booly;

        CompilationResult {
            anti_booly: sub.one_zero, // 0F -> 0T, 1T -> 1F
            one_zero: (1 - sub.one_zero),
        }
    }
```

#### If Then

```
fn translate_if_then(
        cond: S,
        when_true: S,
    ) -> CompilationResult {
        assert!(cond.is_bool());
        assert!(when_true.is_bool());

        let cond = self.translate_bool(cond);
        let when_true = self.translate_bool(when_true);

        CompilationResult {
            anti_booly: cond.one_zero * when_true.anti_booly,
            one_zero: cond.one_zero * when_true.one_zero,
        }
    }
```

#### If Then Else

```
fn translate_if_then_else(
        cond: S,
        when_true: S,
        when_false: S,
    ) -> CompilationResult {
        assert!(cond.is_bool());
        assert!(when_true.is_bool());
        assert!(when_false.is_bool());

        let cond = self.translate_bool(cond);
        let when_true = self.translate_bool(when_true);
        let when_false = self.translate_bool(when_false);

		let one_zero_true = cond.one_zero * when_true.one_zero;
		let one_zero_false = (1 - cond.one_zero) * when_true.one_zero;

        CompilationResult {
            anti_booly: (one_zero_true * one_zero_true).cast_anti_booly(),
            one_zero: one_zero_true * one_zero_true,
        }
    }
```

#### Select

```
fn translate_select(
        cond: S,
        when_true: S,
        when_false: S,
    ) -> T {
        assert!(cond.is_bool());
        assert!(when_true.is_arith());
        assert!(when_false.is_arith());

        let cond = self.translate_bool(cond);
        let when_true = self.translate_arith(when_true);
        let when_false = self.translate_airth(when_false);

        cond.one_zero * when_true + (1 - cond.one_zero) * when_false
    }
```

#### Arithmetic operators

The standard arithmetic operators `+`, `-` and `*` are trivialy translated to the target language equivalent.

### Second phase

At this point we have PI with MI operator, we need to eliminate the MI in order to have a ZKP arithmetization.

For this we need to create a virtual signal for each MI operator, and have a constraint for it.

The `ConstrDecomp` contains the field `auto_signals` with a vector of expressions that calculate the correct witness value for that virtual signal. It also contains the field `constrs` with the PI that are necessary to constraint the value of the virtual signals.

The algorithm recursively go over the target AST, and when it finds a MI operator do the following operation:

```
fn mi_elimination_recursive(
    decomp: &mut ConstrDecomp,
    constr: T,
    signal_factory: &mut SF,
) -> T {
    use Expr::*;

    match constr {
        // other operators are trivial using recursion
        Expr::MI(se) => {
            let se_elim = mi_elimination_recursive(decomp, *se.clone(), signal_factory);

            let virtual_mi = signal_factory.create();
            let constr_inv = Query(virtual_mi.clone());

            decomp.auto_signals.insert(virtual_mi.clone(), MI(se));

            let virtual_constr = se_elim.clone() * (Const(F::ONE) - (se_elim * Query(virtual_mi)));

            decomp.constrs.push(virtual_constr);
            constr_inv
        }
    }
}
```

The algorithm is based on the is_zero gadget classical circuit.

For `MI(se)`  first we eliminate `MI` operators in `se` by using the algorithm recursively, resulting in `se_elim`.

Then we create a virtual signal `virtual_mi` that will substitute `MI(se_elim)`. To make sure than `virtual_mi` is constraint to have the same value that `MI(se_elim)` would have. We add the PI constraint:

`se_elim * (1- se_elim * virtual_mi)` 

It is clear that this expression is zero if and only if  `virtual_mi == MI(se_elim)`.

We add the pair `(virtual_mi, MI(se))`  to `auto_signals` so the right value is assigned during witness generation.

### Optimisation and performance

We do not claim that this algorithm will produce the optimal PIs given a BS. But, rather that this automatic translation is feasible.

There are many ways to optimise this translation and also to simplify the resulting PIs. In the same way that compilers that target CPUs have increased the level of optimisation of the resulting code, ZKP compilers will develop new types of optimisations to produce the best PIs and number of signals.


