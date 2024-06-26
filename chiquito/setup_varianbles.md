The goal of this is to discuss how we can help the user of chiquito to distinguish between variables used to construct the constraints and variables used to constraint the witness.

There are three categories (for not using type as the type of the value they contain) in a circuit description: signals, setup variables and witness generation variables. This exists independently of the strategy of the ZK-lang used to describe them.

If we distinguish: Setup time (ST) and Witness generation time (WGT)

|  | signals | setup | wit gen |
| ---- | ---- | ---- | ---- |
| ST: Value known | no | yes | no |
| WGT: Value known | yes | yes | yes |
| ST: Can be assigned | no | yes | no |
| WGT: Can be assigned | yes | yes | yes |
| ST: Can be used in a assign expression | no | yes | no |
| WGT: Can be used in a assign expression | yes | yes | yes |
| ST: Can be use in a assertion or if or while that contains assertions? | yes | yes | no |
|  |  |  |  |
In circom both setup and witgen variables are defined with `var`,  the compiler will see if the values of the variable can be calculated at ST, and then it will be considered `known`, otherwise `unknown`. I think this is bad because the user has the information about which are which not clearly presented.

## Suggestions

### Setup category

We define `signal a; var b; setup c;`

### Use symbol before name

It seems that the capabilities of wg vars are more limited. We could use the symbol `$` to prefix all wg variables and still defined with `var`.

### wg blocks

We could define `wg` blocks which are the only place where the `wg` vars can be used. These blocks would not allow to add constriants.

### Use arrow to assign setup vars

We could use the arrow `<--` operator to assign setup vars, and protest if a variable is assigned with `=` but is used where a `wg` var cannot be used.

### Use `:=`for assign

We would have `<--` for signals, `=` for wg vars and `:=` for setup vars.

Example:

I know the question if RLC is zero perhaps doesn't make much sense.

```
const setup NUM_ITEMS: field = 10; // I think all consts should be setup, so perhaps "setup" can be omitted.

machine example(signal input[NUM_ITEMS]: field) (signal output: field) {
	challenge X; // or signal X: challenge; ??
	signal intermediate[NUM_ITEMS]: field;
	signal rlc_value;
	signal rlc_is_zero;

	// other steps here ...

	step rlc_is_zero() {
		setup rlc_expr: expr := 1;
		setup i: field := 0;
		var rlc_inv_value;
		signal rlc_inv;

		while (i < NUM_ITEMS) {
			rlc_expr := rlc_expr * X * intermediate[i];
			i++;
		}

		rlc_value' <== rlc_expr;

		rlc_inv_value = 1 \\ rlc_value';
		rlc_inv <- rlc_inv_value;
		
		0 === rlc_value' * (1 - (rlc_value' * rlc_inv));
		rlc_is_zero' === 1 - (rlc_value' * rlc_inv);
	}
}
```