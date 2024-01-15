```
tracer fibo(signal &in_n) -> (signal &result) {
	signal a, b, n, i;

	step first() {
		signal c;

		i, a, b, c <== 1, 1, 1, 2;

		n &== *in_n;

		a', b', n' <== b, c, n;

		middle();
	}

	step middle() {
		internal c;

		c <== a + b;

		i', a', b', n' <== i + 1, b, c, n;

		if i' == n {
			last();
		} else {
			middle();
		}
	}

	step last() {
		b &== *result;
		i === n;
	}

	first();
}
```

This style is more verbose.
```
tracer fibo(signal &in_n) -> (signal &result) {
	signal a, b, n, i;

	step first(a, b, n, i) {
		signal c;

		i, a, b, c <== 1, 1, 1, 2;

		n &== *in_n;

		a', b', n' <== b, c, n;

		middle(a', b', n', i');
	}

	step middle(a, b, n, i) {
		internal c;

		c <== a + b;

		i', a', b', n' <== i + 1, b, c, n;

		if i' == n {
			last(b', n', i');
		} else {
			middle(state'); // NOTE THIS
		}
	}

	step last(b, n, i) {
		b &== *result;
		i === n;
	}

	first(a, b, n, i);
}
```
