# chiquito v2024

## Tracers

A step machine is a description of zero knowledge constraints that validate the trace of the execution of a state machine.

In the PSE/Taiko/scroll zk-EVM we have several step machines (sub circuits), that are combined in a Super Circuit, with the sub circuits connected by advice lookup tables. Chiquito implements this approach already. The main drawbacks we have detected are:
 + Lack of abstraction in the witness generation. In zk-EVM we have the bus mapping, the user basically have to generate most of the witness outside the sub-circuit and then just pass it to the sub-circuits. The user would like for sub-circuit to be able to call other sub-ciruits and return some value that can be used in the witness generation of the caller.
 + Not possible to implement this approach in backend without advice lookup tables.

We are devising a new abstraction called tracers, that solves both problems and open new possibilities to combine step machines.

A tracer is a step machine that can be instantiated more than once in the same witness, and tracers can "call" other traces, so the witness of one is included with the other. Chiquito will generate call and return steps in the caller tracer, so it can constraint the input and output of the callee tracer.

For example:

```
tracer callee (forward signal input a, backward output signal b) {
	forward a;
	step callee_first_step() {
		/// ...
	}
	step callee_intermediate_step() {
		/// ...
	}
	step callee_final_step() {
		/// ...
	}

	callee_first_step()
	callee_intermediate_step()
	callee_final_step()

	return a + b;
}

tracer caller () {
	forward a;
	/// ...
	step caller_step() pre {
		/// ...
		var c = callee(a, b)
		/// ...
	} post {
		
	}
	/// ...

	pre_call()
	callee(a, b)
	post_call()
	caller_step()
}
```

```
function fibo(setup n_bits, signal forward n)(x) -> (signal b, signal n) {
	signal a;
	
	#pragma first
	step first(n_value) {
		internal c;
		
		a <== 1;
		b <== 1;
		c <== 2; // a + b
		n <-- n_value;

		a' <== b;
		b' <== c;
		n' <== n;

		step' === fibo_step
	}

	step fibo_step() {
		internal c;
		c <== a + b;

		a' <== b;
		b' <== c;
		n' <== n;
	}

	#pragma last
	step last() {
	}

	first(n_value);
	var a_value = 1;
	var b_value = 2;

	var i = 1

	while (i < n_value) {
		fibo_step(a_value, b_value, n_value);
		var prev_a = a_value;
		a_value = b;
		b_value = prev_a + b_value;
	}
	last(a, b, n_value)
}
```

We need to think how to join the constraints between the caller and the callee. Perhaps the callee could export forward signals of the first step and backward signals of the last step , to be constraint by the caller.

The syntax can be similar to how circom creates components from templates.

## Arbitrary boolean expression to PI compiler

Chiquito should be able to compile any arbitrary boolean expression to a polynomial identity. We already drafted how to implement this.

We can do this very easily if we have the "inv(b)" operator that is the inverse where "inv(0) == 0".  Then we can eliminate the "inv" operator by creating virtual signals and an extra PI constraint, like done in the is_zero gadget.

## Reduction of the degree of PI

Once we have our PI without "inv" we can convert them to more PI but with a smaller degree automatically. This will allow us to target backends with a maximum PI degree, and also for Halo2 optimise the maximum expression degree.

## Front-end parser with familiar syntax

We will create our own textual language with a similar syntax to circom. Ideally it would be a super set of circom, so any circom language would be compilable by chiquito. This language will extend circom to express chiquito abstractions like tracers, steps, forward signals, etc...

## Compatibility with rust and halo2

We should maintain the possibility of combining chiquito with Halo2 circuits. For that we will have a rust DSL based on macros, that allow to introduce statements in chiquito language, while allowing to reference halo2 expressions and columns.


```
tracer! {
	step! {
		code! "
			a === b + c
		"
	} 
}
```


## Backends

Once we have reduction to any degree we can start implementing backends that we had no access to before.
