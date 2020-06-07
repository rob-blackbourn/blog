---
layout: post
title:  "JavaScript Datascience"
categories: JavaScript data frame
---

# Data Science with Javascript

Let's say we want to do the following in JavaScript and have the code run at
native speed. What would we need?

```javascript
df['bodymass'] = df['weight'] / (df['height'] ** 2)
```

I think three things are required:

1. Operator overloading to enable multiplying arrays.
2. Property indexing, so df['bodymass'] if different from df['columns'].
3. Vector arithmetic implemented in WebAssembly.

