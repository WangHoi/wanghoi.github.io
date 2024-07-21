# JavaScript in kwui

## Entrypoint

### assets/entry.js

**kwui** ScriptEngine load and run the JavaScript entry javascript: `assets/entry.js`. It shows a `Dialog` with parameters, such as dialog window width, height, and script module path of gui root component.
```javascript
app.showDialog({
	title: "Hello kwui",
    width: 1280,
    height: 720,
	modulePath: "./hello.js"
});

```

### assets/hello.js

`assets/hello.js` export the `builder` function, which invoked by **kwui**, to build root gui component and stylesheet on-demand.

```javascript
// JSX syntax to define root component
function Hello(props, kids) {
	return <div style="margin: 16px;">
		<span id="hello">Hello kwui</span>
	</div>;
}

// `css` is a builtin JavaScript template literal, to define CSS-in-JS
var hello_css = css`
#hello {
    color: orangered;
}
`;

// root script module need to export the `builder` function
export function builder(moduleParams) {
	return {
		root: <Hello />,
		stylesheet: hello_css,
	}
}
```

## Dialog

A dialog window is a top-level window mostly used for present graphics visuals and brief communications with the user.

## app.showDialog()
```javascript
let dialogId = app.showDialog({
	title: "Hello kwui",
    width: 1280,
    height: 720,
	modulePath: "./hello.js"
});
```
Create and show a new dialog window on Windows, replace root view content on Android.

## app.closeDialog()

Close the specific dialog on Windows, on effects on Android.

```javascript
app.closeDialog(this.dialogId);
```

## Component Hooks

### useState()

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

### useEffect()

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

### useContext()

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

### Context properties

In component function, you can access creation context, from `this` object.

- `this.dialogId` - the current dialog id where the Component is located.

