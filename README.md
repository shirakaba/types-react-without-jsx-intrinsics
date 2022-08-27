# types-react-without-jsx-intrinsics

`@types/react`, with an empty `JSX.IntrinsicElements` interface, for custom React renderers not based on DOM.

## Background

### What's an intrinsic element?

In JSX, [any elements starting with a lower-case letter](https://reactjs.org/docs/jsx-in-depth.html#user-defined-components-must-be-capitalized) are taken to be "intrinsic elements" rather than user-defined components. The means that when they're transpiled, [JSX factory calls](https://www.typescriptlang.org/tsconfig#jsxFactory) like `React.createElement()` will use the tag name as the first argument, rather than passing some in-scope function with that name. In other words, this:

```jsx
export const HelloWorld = () => <h1>Hello world</h1>;
```

... becomes:

```js
import React from 'react';
export const HelloWorld = () => React.createElement("h1", null, "Hello world");
```

These intrinsic elements are a great feature of React DOM, as the alternative would be to have to import library-defined components, as is the case in React Native:

```js
import { Text } from 'react-native';
export const HelloWorld = () => <Text>Hello world</Text>;
```

Importing library-defined components is honestly comparatively cumbersome as you end up having to do it *a lot*, usually with some linter screaming at you for having either a missing or an unused import while you're in the middle of editing something.

In fact, it gets even more cumbersome if you're building a React custom renderer that has both UI elements and corresponding React wrappers around those:

```jsx
import React from 'react';
import { TextView } from "@nativescript/core";
import { TextView as TextViewComponent } from "react-nativescript";

export const HelloWorld = () => {
    const ref = React.useRef<TextView>();

    // The design here is that the React component exposes a ref to the
    // underlying UI element that it manages, allowing the consumer to call APIs
    // imperatively. However, we need to make some compromise on the naming of
    // the component vs. the element to avoid a name clash, and every solution
    // involves passing a burden of extra typing onto the user.
    return <TextViewComponent ref={ref}>Hello world</TextViewComponent>;
};
```

Intrinsic elements solve this name-clash, and a lot of typing, beautifully - by avoiding the import of a component altogether:

```jsx
import React from 'react';
import { TextView } from "@nativescript/core";

export const HelloWorld = () => {
    const ref = React.useRef<TextView>();

    return <textView ref={ref}>Hello world</textView>;
};
```

So it would be attractive, when building a React custom renderer, if we could base our library on intrinsic elements just like React DOM does.

### The problem

`@types/react` is polluted with typings for React DOM. I have been complaining about this wherever I can for years. The consequence of this is that anyone writing a custom renderer will find that their IntelliSense suggests that all the HTML elements are available at intrinsic elements. You can define the intrinsic elements your library supports by writing to the same interface (`JSX.IntrinsicElements`), but you can only add, not remove - this means that the moment you have a name-clash with a HTML element (most UI libraries will name-clash upon generic names like `<button>`, `<label>`, `<image>`), the TypeScript compiler will start screaming.

Up until now, I've been working around this using `patch-package`, but it's horrible to manage and generally necessitates restarting the TypeScript language server.

Now, there are some options with TypeScript's newer `jsxFactory` option (and indeed, both Svelte Native and Svelte NodeGUI make use of it), but they pass on a burden of extra typing onto the user - they'll often need to import both React itself, for things like `useState()`, and your JSX factory (for the implicit `createElement()` powering the JSX) in every file with UI code. I just want the same first-class treatment that React DOM gets.

### The solution

My hand has been forced. The most user-friendly option I could think up was to republish patched versions of `@types/react` with the `JSX.IntrinsicElements` interface emptied. Users can then install those typings in place of `@types/react` via the relatively new npm [package aliasing](https://github.com/npm/rfcs/blob/main/implemented/0001-package-aliases.md) feature (or any of a number of tsconfig tricks for referencing types).

No more `patch-package`, and no compromises except for having to ensure that any starter templates for your projects have that npm aliasing in place.

## Usage

If you'd normally have installed, say, `@types/react@16.9.49`, then install the equivalent version of `types-react-without-jsx-intrinsics` instead, by following the below instructions. I only support a handful of versions and don't have the motivation to set up automation to handle all possible versions, so please just pick the closest version to the one you need.

The version number of the package *should* exactly mirror that of the corresponding `@types/react` package. If we ever make any mistakes in publishing, the best we can do is increment the patch version and make a note of warning here.

### Installing from npm

You can run this CLI command:

```sh
npm install --save-dev npm:types-react-without-jsx-intrinsics@16.9.49
```

... or equally write this field into your `package.json` and run `npm install`:

```json
"devDependencies": {
    "@types/react": "npm:types-react-without-jsx-intrinsics@16.9.49"
}
```

### Installing from disk (for dev purposes)

You can run this CLI command:

```sh
# Assuming the types-react-without-jsx-intrinsics repo is in the same directory
# as your package.json:
npm install --save-dev ./types-react-without-jsx-intrinsics/16.9.49
```

... or equally write this field into your `package.json` and run `npm install`:

```json
"devDependencies": {
    "@types/react": "file:types-react-without-jsx-intrinsics/16.9.49"
}
```

## Contributing

### Adding another version of `@types/react`

- First, install the desired version of `@types/react` into some temporary project.
- Copy the files out of the temporary project's `node_modules/@types/react` folder into the `packages` directory in this repo.
- Rename the copied directory to reflect the package's version number.
- Edit the package's `package.json`:
  - `name`: `"types-react-without-jsx-intrinsics"`
  - `repository.url`: `"https://github.com/shirakaba/types-react-without-jsx-intrinsics.git"`
  - `repository.directory`: `"packages/16.9.49"` (replace the version number with that of the package in question),
  I personally don't bother editing the `contributors` section as it's just more editing to do and I get attribution through the repository URL anyway.
- Edit the package's `index.d.ts` file such that the interface `JSX.IntrinsicElements` is empty. Again, I personally don't edit the contributors list in the comments.

Now it's ready to be published.

### Publishing the packages

There's no magic tooling; this is a completely manual process. We just change directory to that of the package of interest, then run `npm publish`.

Of course you'll need permissions to publish the package; get in contact if interested!

## Troubleshooting

The answer is always to restart the TypeScript language server in whatever IDE you're using.
