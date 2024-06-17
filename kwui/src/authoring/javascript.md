# JavaScript in kwui

## assets/entry.js

**kwui** ScriptEngine load and run the JavaScript entry javascript: `assets/entry.js`. It shows a `Dialog` with parameters, such as dialog window width, height, and script module path of gui root component.

```javascript
app.showDialog({
	title: "Hello kwui",
    width: 1280,
    height: 720,
	modulePath: "./hello.js"
});

```

## assets/hello.js

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
