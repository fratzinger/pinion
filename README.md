# Pinion

> A fast and typesafe code generator

Pinion is a CLI toolkit that allows to create flexible, customizable and typesafe code generators in TypeScript.
Creating and maintaining CLI tooling and generators can be a very time consuming task. Pinion helps with this by

- Putting generator templates as close to your project as possible
- TypeScript first to give you full flexibility over what your generators can do and how
- Using JavaScript template strings instead of hard to debug templating syntaxes like EJS or Mustache 
- A functional programming approach that keeps your generators robust and easy to follow

## Installation

Pinion is installed as a development dependency and its command line is available via `npx pinion`:

```
npm install @feathershq/pinion --save-dev
```

## Getting started

By default, the default Pinion CLI looks for a `.pinion/pinion.ts` file as the entrypoint. In your project folder, create a `.pinion/pinion.ts` file with the following content:

```ts
import { PinionContext, generator, runGenerators, prompt } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  // Add the types from prompts and command line arguments here
  name: string
}

export const generate = (context: Context) => generator(context)
  // Ask prompts here (using Inquirer) that should be available to all other generators
  .then(prompt([{
    type: 'input',
    name: 'name',
    message: 'What is the name of your app?'
  }]))
  // Run all *.tpl.ts generators in this folder
  .then(runGenerators(__dirname))
```

Then create a `.pinion/readme.tpl.ts` like this:

```ts
import { generator, renderTemplate, toFile } from '@feathershq/pinion'
import { Context } from './pinion'

// A template to render using JavaScript template strings
const template = ({ name }: Context) =>
`# ${name}

This is a readme generated by Pinion

Copyright (c) ${new Date().getFullYear()}
`

export const generate = (context: Context) => generator(context)
  .then(renderTemplate(template, toFile('readme.md')))
```

Then run

```
npx pinion
```

## Generators

A Pinion generator is a TypeScript (or JavaScript) file that exports a `generate` function. The `generate` function takes a `context` which contains runtime properties (like the current working directory, access to a logger etc.) and properties added by command line arguments, prompts and other operations. A basic generator looks like this:

```ts
import { PinionContext, generator } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  // Add the types from prompts and command line arguments here
  name: string
}

export const generate = (context: Context) => generator(context)
  .then(/* operations run here */)
```

## Command line options

A generator can define command line options using [Yargs](https://yargs.js.org/) and exporting a `command` variable:

```ts
import { Argv } from '@feathershq/pinion'

export const command = (yargs: Argv) => yargs
  .usage('Usage: $0 <command> [options]')
  .command('app', 'Generate my awesome app', () => {})
  .help()
```

## Operations

Operations perform actions and possibly update the context with information that can be used in the next step. Most operations take their arguments in two forms. Either as plain values like `renderTemplate('This is the template string')` or functions that get called with the current `context` and return the value, e.g. `renderTemplate(context => \`This is a dynamic template for ${context.name}\`)`. All functions can also be asynchronous.

### prompt

`prompt(options|context => options)` takes a list of questions using [inquirer.js](https://github.com/SBoudrias/Inquirer.js) and updates the context with results. A context callback can be used to only ask prompts conditionally (e.g. skipping when they have already been provided from the command line).

```ts
import { PinionContext, generator, prompt } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  // Add the types from prompts and command line arguments here
  name: string
}

export const generate = (context: Context) => generator(context)
  .then(prompt(context => [{
    type: 'input',
    name: 'name',
    message: 'What is the name of your app?',
    // Only show prompt if there is no name
    when: !context.name
  }]))
```

### renderTemplate

`renderTemplate(text|context => text, toFile, writeOptions)` renders a string to a target file. `writeOptions` can be `{ force: true }` to skip prompting if an an existing file should be overwritten.

To put together file names dynamically, the `toFile` helper can be used"

```ts
import { PinionContext, generator, prompt, renderTemplate, toFile } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  // Add the types from prompts and command line arguments here
  name: string
}

export const template = ({ name } : Context) =>
`# ${name}

This is a readme generated by Pinion

Copyright (c) ${new Date().getFullYear()}
`


export const generate = (context: Context) => generator(context)
  .then(prompt(context => [{
    type: 'input',
    name: 'name',
    message: 'What is the name of your app?',
    // Only show prompt if there is no --name CLI argument
    when: !context.name
  }]))
  .then(renderTemplate(template, toFile('readme.md')))
```

### when

`when(boolean|context => boolean, operation)` evaluates a condition and runs the operation when it returns true.

```ts
import { PinionContext, generator, prompt } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  // Add the types from prompts and command line arguments here
  name: string
}

export const generate = (context: Context) => generator(context)
  .then(prompt(context => [{
    type: 'input',
    name: 'name',
    message: 'What is the name of your app?',
    // Only show prompt if there is no name
    when: !context.name
  }]))
  .then(when<Context>(
    ({ name }) => name === 'David',
    renderTemplate('I\'m afraid I can\'t do that Dave', toFile('greeting.md'))
  )
```

### inject

`inject(text|context => text, location, toFile)` injects a template at a specific location into an existing file. The location functions can be used as follows:

- `before(text|context => text)` inject at the line before `text`
- `after(text|context => text)` inject at the line after `text`
- `prepend()` inject ad the beginning of the file
- `append()` inject at the end of the file

```ts
import { PinionContext, inject, before, prepend, toFile } from '@feathershq/pinion'

export const generate = (context: PinionContext) => generator(context)
  .then(inject('Injected before copyright notice', before('Copyright (c)'), toFile('readme.md')))
  .then(inject('Appended hello world', append(), toFile('readme.md')))
```

### copyFiles

`copyFiles(fromFile, toFile, options)` recursively copies all files from a location to a destination. It will prompt to overwrite if a file already exists. `writeOptions` can be `{ force: true }` to skip prompting if an an existing file should be overwritten.

```ts
import { PinionContext, copyFiles, fromFile, toFile } from '@feathershq/pinion'

export const generate = (context: PinionContext) => generator(context)
  .then(copyFiles(fromFile(__dirname, 'static'), toFile('.')))
```

### writeJSON

`writeJSON(data|context => data, toFile, writeOptions)` write JSON data to a file. `writeOptions` can be `{ force: true }` to skip prompting if an an existing file should be overwritten.

```ts
import { PinionContext, writeJSON, toFile } from '@feathershq/pinion'

export const generate = (context: PinionContext) => generator(context)
  .then(writeJSON({ description: 'Something' }, toFile('package.json')))
```

### mergeJSON

`mergeJSON(data|context => data, toFile, writeOptions)` merges new data into an existing file.

### readJSON

`readJSON(fromFile, converter, fallback?)` loads a JSON file, parses it and extends the `context` with the data returned by `converter`. 

```ts
import { PackageJson } from 'type-fest'
import { PinionContext, writeJSON, readJSON fromFile, toFile } from '@feathershq/pinion'

// The main types of your generator
export interface Context extends PinionContext {
  package: PackageJson
}

export const generate = (context: Context) => generator(context)
  // Load package.json, fall back to an empty object if it doesn't exist
  .then(readJSON(fromFile('package.json'), package => ({ package }), {}))
  // Merge existing package.json and write an updated description
  .then(writeJSON<Context>(
    ({ package }) => ({...package, description: 'Overwritten description' }),
    toFile('package.json')
  ))
```

### install

`install(dependencies[]|context => dependencies[], dev, packageManager = 'npm')` dependencies using the `npm` or `yarn` package manager.

```ts
import { PinionContext, generator, install } from '@feathershq/pinion'

export const generate = (context: Context) => generator(context)
  .then(install(['@feathersjs/feathers']))
  .then(install(['ts-node'], true))
```

### runGenerators

`runGenerators((filePart|context => filePart)[])` will run all `*.tpl.ts` or `*.tpl.js` generators in the given path alphabetically in sequence, passing the current context.

```ts
import { PinionContext, generator, runGenerators } from '@feathershq/pinion'

export const generate = (context: Context) => generator(context)
  .then(runGenerators(__dirname, 'templates')
```

## License

Copyright (c) 2022 Feathers Cloud Inc.

Licensed under the [MIT license](./LICENSE).
