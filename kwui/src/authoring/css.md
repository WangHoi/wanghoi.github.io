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
