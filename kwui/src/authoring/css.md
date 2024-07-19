# CSS in kwui

**kwui** has a builtin CSS parser and layout engine, implements a subset of [CSS 2.2 Visual Formatting Model](https://www.w3.org/TR/CSS22/visuren.html).

## Supported CSS attributes

- **position**: static, relative, absolute
- **display**: none, inline, inline-block, block
- **margin-\***, **border-\***, **padding-\***
- **width**, **height**
- **background-color**, **background-image**
- **border-radius**
- **color**
- **text-align**, **line-height**, **vertical-align**
- **line-height**

## Supported CSS selectors

- **tag, class, id**: `div {}`, `.class1 {}`, `#id`
- **combinators**: `div span {}`, `div + span {}`, `p, span {}`

## Style Scoped

Full, scoped and component-friendly CSS support for JSX, inspired by [styled-jsx](https://github.com/vercel/styled-jsx).

- Locally-scoped styles
- Co-location
- JavaScript variables in styles

```javascript
function StyleComponent(props) {
   return (<div class="style-root">
        <style jsx>{`
            div.style-root {
                font-size: 24px;
            }
            span {
                color: blue;
            }
        `}</style>
        <span>blue text</span>
    </div>);
}
```

**kwui** provide native support for scoped style.
When your components render, static styles are cached for better performance.
CSS reparse only occurs when you are using JavaScript variables in styles. 

> Note: scoped style feature was added in `v0.3.0`.

