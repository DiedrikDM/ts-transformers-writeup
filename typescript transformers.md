# TypeScript Transformers: Transform and Rise Up!

I'm going to start with a major disappointment: this article has nothing to do with the eternal war of Autobots vs Decepticons! 
Alright, now that's out of the way, let's look at TypeScript Transformers.

## What is a TypeScript Transformer?

TypeScript uses a compiler to get turned into normal JavaScript. What if we could hook into that compiler to change some code into our own JavaScript result? Well that's exactly what a Transformer can do!

The TypeScript team has exposed a compiler API that we can use to create custom transformations at compile time. With a Transformer we will traverse the AST (Abstract Syntax Tree) and change the end result. These changes can be used to add, replace and remove statements.

## Why would I use this?

If you're building applications with certain frameworks (like Angular), you are already using them! At build time these frameworks look at your TypeScript code and introduce extra JavaScript or optimize it.

A TypeScript Transformer can be used to look at your existing code and introduce new behavior to your end result. Consider a Decorator that can be added to your code. If you want to use Decorators, you still need to add polyfills to be able to use them. A Transformer can do this in a nicer way!

## Let's look at an example of a Transformer

In TypeScript we can import and export functionality from one file to another in a very simple way.

```TS
import {sum} from './math';

console.log(sum(20,22));
```
```TS
export function sum(a: number, b: number): number{
  return a + b;
}
```

When we need functionality in a file, we need to import that piece of code, otherwise you cannot use it. For TypeScript the compiler will automatically translate these import and export statements into calls for a chosen Module Loader. TypeScript supports five options: System, UMD, AMD, CommonJS, ES2015. When your browser tries to execute the JavaScript result of four of these options it will only work with the help of an extra library.

__The good news is, one option is natively supported!__

Starting from 2017 browsers started to support the ES2015 module loading. _Well the language feature by itself was already supported, but the functionality to load modules dynamically only came later._ This is a nice feature! 

The TypeScript compiler on the other hand has a couple of issues in combination with this feature...

## What's the problem?

If we look at the TypeScript from before and compile it, here's the end result in JavaScript.

```JS
import { sum } from "./math";
console.log(sum(20, 22));

```
Everything looks fine, except if you try it out in your browser, you'll get an error message:

IMAGE OF ERROR MESSAGE - CANNOT LOAD

The problem is that when you compile TypeScript modules - which normally don't take an extension - it will not output an extension either. Which is a problem for the Native Module Loading of the browser.

__How can we solve this?__
1. We can add the .js file extension to the import statements in our TypeScript files. Even though the files are originally .ts files, this will work! It is the quickest solution, but ofcourse means you need to do this in every file importing functionality from a TypeScript file.

2. We'll use a TypeScript Transformer to add the .js file extension for use at runtime!

## Adding a TypeScript Transformer

If you want to create your own TypeScript Transformer, you need to know the API.
The two most important types that are used:

```TS
type TransformerFactory<T extends Node> = (context: TransformationContext) => Transformer<T>;

type Transformer<T extends Node> = (node: T) => T;
```
The Factory needs to be called when we start the compilation process. It is a function that will create the actual Transformer which is another function. The Transformer will take any kind of node of the AST and needs to return a node by itself (the same or another).

What kind of nodes might we encounter?
  - CallExpression _like sum(1,2)_
  - BinaryExpression _like name = createName()_
  - ClassDeclaration _like class MyComponent {}_
  - ImportDeclaration _like import {sum} from ‘./math’_
  - VariableStatement _like const a = 42_
  - MethodDeclaration
  - ….

Now for the purpose of this example, we'll need to use __ImportDeclaration__.

The import fixing code is in credit of this github repository: [https://github.com/Zoltu/typescript-transformer-append-js-extension](https://github.com/Zoltu/typescript-transformer-append-js-extension).

__1. Check if it's necessary to fix the import__

In the following piece of code we'll check if the Import Module Specifier actually needs to be fixed. We will only update it if it is indeed an Import or ExportDeclaration, if it is a String, if it is a relative path and if it does not end with a file extension.
```TS
function shouldMutateModuleSpecifier(node: typescript.Node): node is (typescript.ImportDeclaration | typescript.ExportDeclaration) & { moduleSpecifier: typescript.StringLiteral } {
  if (!typescript.isImportDeclaration(node) && !typescript.isExportDeclaration(node)) return false

  if (node.moduleSpecifier === undefined) return false

  // only when module specifier is valid
  if (!typescript.isStringLiteral(node.moduleSpecifier)) return false

  // only when path is relative
  if (!node.moduleSpecifier.text.startsWith('./') && !node.moduleSpecifier.text.startsWith('../')) return false

  // only when module specifier has no extension
  if (path.extname(node.moduleSpecifier.text) !== '') return false

  return true
}
```

__2. Update the Import/Export Declaration__

After we checked whether an update is required, we'll do the actual update. The fix is pretty simple, we need to add the .js file extension - so appending .js to the filename is sufficient.

``` TS
function visitNode(node: typescript.Node): typescript.VisitResult<typescript.Node> {
  if (shouldMutateModuleSpecifier(node)) {
    if (typescript.isImportDeclaration(node)) {
      const newModuleSpecifier = typescript.createLiteral(`${node.moduleSpecifier.text}.js`)
      return typescript.updateImportDeclaration(node, node.decorators, node.modifiers, node.importClause, newModuleSpecifier)
    } else if (typescript.isExportDeclaration(node)) {
      const newModuleSpecifier = typescript.createLiteral(`${node.moduleSpecifier.text}.js`)
      return typescript.updateExportDeclaration(node, node.decorators, node.modifiers, node.exportClause, newModuleSpecifier)
    }
  }

  return typescript.visitEachChild(node, visitNode, transformationContext)
}
```

__3. Put everything in the TransformerFactory__

Now we need to wrap everything into a function that will be used to create the Transformer. In this Transformer we should include the _visitNode_ function and the _shouldMutateModuleSpecifier_ function on the position of the ellipses (...).

```TS
import * as typescript from 'typescript'
import * as path from 'path'

const transformer = (_: typescript.Program) => (transformationContext: typescript.TransformationContext) => (sourceFile: typescript.SourceFile) => {
  
  // ...

	return typescript.visitNode(sourceFile, visitNode)
}

export default transformer

```

## Using the Transformer

Now it's great that we have the actual Transformer, but we need to call it during the compilation process. The problem is, there's no way to just hook it into the standard __tsc__ compiler. The TypeScript team has something in the pipeline to fix this ([TypeScript issue 14419](https://github.com/Microsoft/TypeScript/issues/14419#issuecomment-484162367)). However as of this writing, there are two possible solutions:

1. Create our own build executable that will run our Transformer step. The problem with this - which is already a thing - is that every tool/bundler/builder will have its own way of working without standardization. And we're now adding another one to that collection.

2. Use an existing third-party tool called __ttypescript__ - yes with two T's. This tool wraps around the standard __tsc__ and allows to add a Transformer to the tsconfig.json file.

To bind everything together, we need to do the following things:

1. Install the typescript and ttypescript modules. I will also install the Transformer from npm, because it already exists anyways
```
npm install --save-dev typescript
npm install --save-dev ttypescript
npm install --save-dev @zoltu/typescript-transformer-append-js-extension
```
2. Add the Transformer to your tsconfig.json file
```
{
	"compilerOptions": {
		"module": "es2015",
		"plugins": [
			{
				"transform": "@zoltu/typescript-transformer-append-js-extension/output/index.js",
				"after": true,
			}
		]
	},
}
```

3. Compile using __ttsc__
```
npx ttsc
```

And this will make sure the .js file extension gets added to our import statement!

``` JS
import { sum } from "./math.js";
console.log(sum(20, 22));
```

That concludes this blog post on TypeScript Transformers. Thanks for the read and have a good continuation of your day!
