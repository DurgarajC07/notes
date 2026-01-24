# JavaScript Parser & AST

> **Abstract Syntax Trees, parsing, and code transformation**

---

## 🎯 Overview

An Abstract Syntax Tree (AST) is a tree representation of JavaScript source code. Understanding ASTs is essential for building tools like linters, bundlers, transpilers, and code formatters.

---

## 📚 Core Concepts

### **Concept 1: Parsing Process**

JavaScript parsing converts source code to AST:

```
Source Code → Lexical Analysis (Tokenization) → Syntax Analysis → AST
```

**Example:**

```javascript
// Source code
const sum = 1 + 2;

// Tokens
[
  { type: 'Keyword', value: 'const' },
  { type: 'Identifier', value: 'sum' },
  { type: 'Punctuator', value: '=' },
  { type: 'Numeric', value: '1' },
  { type: 'Punctuator', value: '+' },
  { type: 'Numeric', value: '2' },
  { type: 'Punctuator', value: ';' }
]

// AST (simplified)
{
  type: 'VariableDeclaration',
  kind: 'const',
  declarations: [{
    type: 'VariableDeclarator',
    id: { type: 'Identifier', name: 'sum' },
    init: {
      type: 'BinaryExpression',
      operator: '+',
      left: { type: 'Literal', value: 1 },
      right: { type: 'Literal', value: 2 }
    }
  }]
}
```

### **Concept 2: AST Node Types**

Common AST node types:

```javascript
// Literals
42; // → { type: 'Literal', value: 42 }
("hello"); // → { type: 'Literal', value: 'hello' }

// Identifiers
foo; // → { type: 'Identifier', name: 'foo' }

// Binary Expressions
a + b; // → { type: 'BinaryExpression', operator: '+', left: {...}, right: {...} }

// Function Declarations
function foo() {}
// → { type: 'FunctionDeclaration', id: {...}, params: [], body: {...} }

// Call Expressions
foo(1, 2);
// → { type: 'CallExpression', callee: {...}, arguments: [...] }
```

---

## 🔥 Practical Examples

### **Example 1: Using Babel Parser**

```javascript
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generate = require("@babel/generator").default;

// Parse code
const code = "const x = 1 + 2;";
const ast = parser.parse(code);

// Traverse and modify
traverse(ast, {
  BinaryExpression(path) {
    // Replace + with *
    if (path.node.operator === "+") {
      path.node.operator = "*";
    }
  },
});

// Generate code from AST
const output = generate(ast);
console.log(output.code); // const x = 1 * 2;
```

### **Example 2: Finding All Function Calls**

```javascript
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const code = `
  console.log('hello');
  foo();
  bar.baz();
`;

const ast = parser.parse(code);
const functionCalls = [];

traverse(ast, {
  CallExpression(path) {
    const calleeName =
      path.node.callee.type === "Identifier" ? path.node.callee.name : "method";
    functionCalls.push(calleeName);
  },
});

console.log(functionCalls); // ['log', 'foo', 'method']
```

---

## 🏗️ Best Practices

1. **Use existing parsers** - Don't write your own (use @babel/parser, acorn, esprima)
2. **Handle errors gracefully** - Invalid syntax should be caught

   ```javascript
   try {
     const ast = parser.parse(code);
   } catch (error) {
     console.error("Parse error:", error.message);
   }
   ```

3. **Optimize traversal** - Only visit nodes you need
   ```javascript
   traverse(ast, {
     FunctionDeclaration(path) {
       /* specific node type */
     },
   });
   ```

---

## 🧪 Interview Questions

### **Q1: What is an AST and why is it important?**

**Answer:** An AST is a tree representation of source code structure. It's important for:

- **Linters** (ESLint) - analyzing code quality
- **Transpilers** (Babel) - converting ES6+ to ES5
- **Bundlers** (Webpack) - analyzing dependencies
- **Formatters** (Prettier) - consistent code style
- **Type checkers** (TypeScript) - static analysis

### **Q2: Describe the JavaScript parsing process.**

**Answer:**

1. **Lexical Analysis (Tokenization):** Source code → tokens
2. **Syntax Analysis (Parsing):** Tokens → AST based on grammar rules
3. **Tree Structure:** AST represents program structure as nested nodes

Example: `const x = 1;` → tokens → VariableDeclaration AST node with Identifier and Literal children.

---

## 📚 Key Takeaways

- AST is a tree representation of code structure
- Parsing involves tokenization (lexical analysis) and syntax analysis
- Tools like Babel, ESLint, Webpack all use ASTs
- AST enables code transformation, analysis, and optimization
- Use existing parsers like @babel/parser rather than building your own

---

**Master javascript parser & ast for production-ready code!**
