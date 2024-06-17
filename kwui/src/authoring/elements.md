# Composite Pattern

## Primitive Elements

### Text

```javascript
function ExampleTextComponent() {
    return (<p id="text-container">
        anonymous text span
        <span id="simple-text">simple text span</span>
        <span>{"escaped text: \u{f111}\u{f192}"}</span>
    </p>);
}
var textStyle = css`
#text-container {
    text-align: center;
}
#simple-text {
    color: blue;
    font-size: 24px;
}
```

### Image

```javascript
function ImageComponent() {
    return <div id="img_a"></div>;
}
var imageStyle = css`
#id {
    width: 32px;
    height: 32px;
    background-image: url(':/sample.png');
}
`;
```

### Button

```javascript
function ButtonComponent(props, kids) {
    return <button onclick={() => console.log("clicked")}>Button</button>;
}
var buttonStyle = css`
button { margin: 10px; padding: 4px; background-color: orange; }
button:hover { background-color: orangered; }
`;
```

### LineEdit

```javascript
function LineEditComponent(props, kids) {
    return <line_edit id="line-edit" value="abc"></line_edit>;
}
var buttonStyle = css`
#line-edit {
	display: inline-block;
	width: 200px;
	height: 32px;
}`;
```
