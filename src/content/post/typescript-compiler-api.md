---
title: Under the hood of ESLint and Prettier - ASTs and the TypeScript Compiler API
description: “The engine powers the most popular Typescript extensions, and how does it work?”
publishDate: "2026-05-28T11:23:00Z"
---

How TypeScript tools like Prettier and ESLint actually work is often abstracted. A few VS Code actions are configured, and the codebase is perfectly formatted on every save. At some point, that surface-level understanding becomes insufficient. I am quite opposed to relying on black-box abstractions that I cannot debug or replicate. I want to understand the underlying logic, and it turns out, almost all of these tools share the exact same starting point.

### The structure - Abstract Syntax Trees

To manipulate code programmatically, you can't just read it like a regular text document. Before any tool can fix or lint your code, it has to be converted from its current human-readable state (a string of characters) into a machine-readable structure. That is happening with ASTs.
Abstract Syntax Trees are the format that powers almost every tool in the TypeScript ecosystem. ESLint, Prettier, and the VS Code language server all start by converting your code into one.
When you type const user: User = getUser(id);, the parser doesn't see a sentence. It sees a hierarchical map. Each piece of the code becomes a node in a tree, with a specific role.
The following code -

```typescript
const user: User = getUser(id);
```

Turns into the following structure -

```
VariableStatement
└── VariableDeclaration
    ├── Identifier "user"
    ├── TypeReference "User"
    └── CallExpression
        ├── Identifier "getUser"
        └── Identifier "id"
```

This line of code is transformed and read as a tree of nodes. A VariableStatement containing a VariableDeclaration containing a CallExpression. Each node knows its kind, its position in the file, and its children. Written code is just a string; the AST is the structure underneath it.

Tools like Prettier and ESLint can be thought of as Tree traversers -

- ESLint - Traverses the tree looking for specific nodes (e.g., "Find all VariableDeclaration nodes that lack a specific naming convention").

- Prettier: Traverses the tree to re-serialize it. It effectively breaks the tree apart and reassembles it according to a set of style rules.

### Analyzing code - The TypeScript Compiler API

ASTs are the underlying data structure, but there is still a need for a compiler API in order to read, create, and analyze them. For that, you have the TypeScript Compiler Api. It's the programmatic toolbox that allows interaction and manipulation of the data structure. While the AST is the object, the Compiler API is the set of functions used to traverse, query, and modify that object.

A tool worth mentioning is the [TypeScript AST Viewer](https://ts-ast-viewer.com/). It lets you paste in any valid TypeScript code and outputs it as an AST. It’s great for both learning and development purposes.
For example, pasting in the previous code line immediately visualizes the tree structure discussed above, and interacting with it reveals the raw properties the compiler uses:
![ast explorer demo](20260513-1928-34.8643920.gif)

### Putting it into practice: TS-to-Go

For a recent project, I needed to automate the conversion of TypeScript interfaces into Go structs to keep data models in sync. Instead of manual duplication, I used the TypeScript Compiler API to:

- Parse the source TypeScript files into an AST.

- Walk the tree to find InterfaceDeclaration and PropertySignature nodes (indicating models).

- Map TypeScript types (string, number, boolean) to their Go equivalents.
  for example, mapping primitive TypeScript types to their Go equivalent -

```typescript
/** maps primitive typescript types to go types
 * @example MyType = string; // maps to string
 * MyType = number; // maps to float64
 * MyType = boolean; // maps to bool
 */
export function mapPrimitiveType(typeNode: ts.TypeNode): string | undefined {
	switch (typeNode.kind) {
		case ts.SyntaxKind.StringKeyword:
			return "string";
		case ts.SyntaxKind.NumberKeyword:
			return "float64";
		case ts.SyntaxKind.BooleanKeyword:
			return "bool";
	}
}
```

You can check out the full implementation of the transformer [here](https://github.com/annaadar/ts-to-go).

Getting comfortable with the Compiler API essentially “ruins” the magic of ESLint and Prettier - in a good way. With this knowledge, you can see that nothing in programming is actually magic; it’s just something you have not learned yet.

## Resources

- [Ts-to-Go on Github](https://github.com/annaadar/ts-to-go)
- ["Using the Compiler API" .md on the Typescript Repo](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API)
