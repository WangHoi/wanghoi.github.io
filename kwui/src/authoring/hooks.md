# JavaScript Component Hooks

## useState

```javascript
import { useState, useContext, createContext } from "Keact";

export function UseStateExample(props) {
	let [n, setN] = useState(0);
	return <button onclick={() => setN(n + 1)}>{`Click ${n} times`}</button>;
}

export function builder() {
	return {
		root: <UseStateExample></UseStateExample>,
		stylesheet: css`
button { margin: 10px; padding: 4px; background-color: orange; }
button.dark { background-color: black; color: white; }
button:hover { background-color: orangered; }
`
	};
}
```

## useEffect

```javascript
import { useState, useEffect } from "Keact";

export function UseEffectExample(props) {
	let [n, setN] = useState(0);
	useEffect(() => {
		console.log("useEffect setup, n:", n);
		return () => console.log("useEffect cleanup, n:", n);
	}, [n]);
	return <button onclick={() => setN(n + 1)}>{`Click ${n} times`}</button>;
}

export function builder() {
	return {
		root: <UseEffectExample></UseEffectExample>,
		stylesheet: css`
button { margin: 10px; padding: 4px; background-color: orange; }
button.dark { background-color: black; color: white; }
button:hover { background-color: orangered; }
`
	}
}
```

## useContext

```javascript
import { useState, useContext, createContext } from "Keact";

let [Theme, ThemeProvider] = createContext();

function Sub1(props, kids) {
	return kids;
}

function Sub2(props, kids) {
	let theme = useContext(Theme);
	return <button class={theme}>Hello</button>
}

function ThemeExample() {
	return <ThemeProvider value="dark">
		<Sub1>
			<Sub2></Sub2>
		</Sub1>
	</ThemeProvider>
}

export function builder() {
	return { root: <ThemeExample /> };
}
```

