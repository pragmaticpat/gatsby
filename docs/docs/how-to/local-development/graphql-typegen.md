---
title: GraphQL Typegen
examples:
  - label: Using GraphQL Typegen
    href: "https://github.com/gatsbyjs/gatsby/tree/master/examples/using-graphql-typegen"
---

## Introduction

If you're already using [Gatsby with TypeScript](/docs/how-to/custom-configuration/typescript) and manually typing the results of your queries, you'll learn in this guide how Gatsby's automatic GraphQL Typegen feature can make your life easier. By relying on the types that are generated by Gatsby itself and using autocompletion for GraphQL queries in your IDE you'll be able to write GraphQL queries quicker and safer.

This feature was added in `gatsby@4.15.0`.

## Prerequisites

- A Gatsby project set up with `gatsby@4.15.0` or later.
- The config option [`graphqlTypegen`](/docs/reference/config-files/gatsby-config/#graphqltypegen) set to `true` inside `gatsby-config`.
  ```js:title=gatsby-config.js
  module.exports = {
    graphqlTypegen: true,
  }
  ```
- A `tsconfig.json` in your project with `"include": ["./src/**/*"]` set. See [full `tsconfig.json` example](https://github.com/gatsbyjs/gatsby/tree/master/examples/using-graphql-typegen/tsconfig.json).
- Optional: If you use VSCode you may install the [GraphQL extension](https://marketplace.visualstudio.com/items?itemName=GraphQL.vscode-graphql).

## Using the autogenerated `Queries` types

For this example to work you'll have to have a `title` inside your `siteMetadata` (in `gatsby-config`).

1. Start the development server with `gatsby develop`. Once the server is ready you should see a log message `Generating GraphQL and TypeScript types` at the bottom of your terminal.
1. Gatsby created a type generation file which you should see at `src/gatsby-types.d.ts`. It contains the TypeScript types for your queries. Your `tsconfig.json` should include this file so that you can access the `namespace Queries` everywhere in your project.
1. Create a new page at `src/pages/typegen.tsx` with the following contents:

   ```tsx:title=src/pages/typegen.tsx
   import * as React from "react"
   import { graphql, PageProps } from "gatsby"

   const TypegenPage = ({ data }: PageProps) => {
     return (
       <main style={pageStyles}>
         <p>Site title: TODO</p>
         <hr />
         <p>Query Result:</p>
         <pre>
           <code>{JSON.stringify(data, null, 2)}</code>
         </pre>
       </main>
     )
   }

   export default TypegenPage

   export const query = graphql`
     query TypegenPage {
       site {
         siteMetadata {
           title
         }
       }
     }
   `
   ```

   It is important that your query has a name (here: `query TypegenPage {}`) as otherwise the automatic type generation doesn't work. We recommend naming the query the same as your React component and using [PascalCase](https://en.wiktionary.org/wiki/Pascal_case). You can enforce this requirement by using [`graphql-eslint`](#graphql-eslint).

1. Access the `Queries` namespace and use the `TypegenPageQuery` type in your React component like so:

   ```tsx
   ({ data }: PageProps<Queries.TypegenPageQuery>)
   ```

   When you type out the site title like this you should get TypeScript IntelliSense:

   ```tsx
   <p>Site title: {data.site?.siteMetadata?.title}</p>
   ```

### Configuring the gatsby-config option

Instead of setting a boolean value for the `graphqlTypegen` option in `gatsby-config` you can also set an object to configure it. See all details in the [gatsby-config documentation](/docs/reference/config-files/gatsby-config/#graphqltypegen).

If for example you use `typesOutputPath` to specify a different path, make sure to also update the `"include"` setting in your `tsconfig.json` to include the new path.

### Non-Nullable types

As Gatsby [infers all fields](/docs/glossary#inference) — unless an explicit schema by the user is provided — they are nullable by default. For GraphQL Typegen this means that fields are possibly `null`. You can see this in the example above where you had to type `data.site?.siteMetadata?.title` as both `siteMetadata` and `title` are nullable.

If you're sure that `siteMetadata.title` is always available you can use Gatsby's schema customization API to explicitly type your fields:

```ts:title=gatsby-node.ts
import { GatsbyNode } from "gatsby"

export const createSchemaCustomization: GatsbyNode["createSchemaCustomization"] = ({ actions }) => {
  actions.createTypes(`
    type Site {
      siteMetadata: SiteMetadata!
    }

    type SiteMetadata {
      title: String!
    }
  `)
}
```

Read the [Customizing the GraphQL schema guide](/docs/reference/graphql-data-layer/schema-customization/) to learn how to explicitly define types in your site or source plugin.

### GraphQL fragments

Fragments allow you to reuse parts of GraphQL queries throughout your site and collocate specific parts of a query to individual files. Learn more about it in the [Using GraphQL fragments guide](/docs/reference/graphql-data-layer/using-graphql-fragments/).

In the context of GraphQL Typegen fragments enable you to have individual TypeScript types for nested parts of your queries since each fragment will be its own TypeScript type. You then can use these types to e.g. type arguments of components consuming the GraphQL data.

Here's an example (also used in [using-graphql-typegen](https://github.com/gatsbyjs/gatsby/tree/master/examples/using-graphql-typegen)) with a `Info` component that gets the `buildTime` as an argument. This `Info` component and its `SiteInformation` fragment is used in the `src/pages/index.tsx` file then:

```tsx:title=src/components/info.tsx
import * as React from "react"
import { graphql } from "gatsby"

// highlight-next-line
const Info = ({ buildTime }: { buildTime?: any }) => {
  return (
    <p>
      Build time: {buildTime}
    </p>
  )
}

export default Info

// highlight-start
export const query = graphql`
  fragment SiteInformation on Site {
    buildTime
  }
`
// highlight-end
```

```graphql:title=src/pages/index.tsx
# Rest of the page above...

query IndexPage {
  site {
    ...SiteInformation
  }
}
```

This way a `SiteInformationFragment` TypeScript type will be created that you can use in the `Info` component:

```tsx:title=src/components/info.tsx
const Info = ({ buildTime }: { buildTime?: Queries.SiteInformationFragment["buildTime"] }) => {}
```

### Tips

- When adding a new key to your GraphQL query you'll need to save the file before new TypeScript types are generated. The autogenerated files are only updated on file saves.
- When using `gatsby-plugin-image` (and Image CDN) you'll get the correct TypeScript types for `gatsbyImageData` and `gatsbyImage` automatically.
- We recommend adding `src/gatsby-types.d.ts` to your `.gitignore` as it's machine generated code and duplicated information that is already existent in e.g. your page queries.

## Configuring the VSCode GraphQL plugin

1. Install the [GraphQL extension](https://marketplace.visualstudio.com/items?itemName=GraphQL.vscode-graphql) in your VSCode.
1. Create a `graphql.config.js` at the root of your project with the following contents:

   ```js:title=graphql.config.js
   module.exports = require("./.cache/typegen/graphql.config.json")
   ```

   The VSCode extension will pick up the `graphql.config.js` file and use the autogenerated file from Gatsby's `.cache` directory.
   To learn more about `graphql.config.js` head to the [GraphQL Config documentation](https://www.graphql-config.com/docs/user/user-usage).

1. Restart VSCode so that the GraphQL extension takes effect.
1. Start the development server with `gatsby develop`.
1. Go to any of your queries, e.g. a page query inside `src/pages` and use <kbd>Ctrl + Space</kbd> (or use <kbd>Shift + Space</kbd> as an alternate keyboard shortcut) to get autocompletion like in GraphiQL.

### Multiple GraphQL projects

If your repository has multiple GraphQL projects including Gatsby, you can configure it with the [`projects` key](https://www.graphql-config.com/docs/user/user-usage#projects):

```js:title=graphql.config.js
module.exports = {
  projects: {
    site: require("./.cache/typegen/graphql.config.json"),
    other: {
      // other config
    }
  }
}
```

### Subdirectory

If your Gatsby project is in a subdirectory, e.g. `site` your config should look like this:

```js:title=graphql.config.js
module.exports = require("./site/.cache/typegen/graphql.config.json")
```

## `graphql-eslint`

You can optionally use [`graphql-eslint`](https://github.com/B2o5T/graphql-eslint) to lint your GraphQL queries. It seamlessly integrates with the `graphql.config.js` file you created in the other step.

This guide assumes that you don't have any existing ESLint configuration yet. You'll need to adapt your configuration if you're already using ESLint and refer to [`graphql-eslint`'s documentation](https://github.com/B2o5T/graphql-eslint/tree/master/docs).

1. Install dependencies

   ```shell
   npm install --save-dev eslint @graphql-eslint/eslint-plugin @typescript-eslint/eslint-plugin @typescript-eslint/parser
   ```

1. Edit your `package.json` to add two scripts:

   ```json:title=package.json
   {
     "scripts": {
       "lint": "eslint --ignore-path .gitignore .",
       "lint:fix": "npm run lint -- --fix"
     },
   }
   ```

1. Create a `.eslintrc.js` file to configure ESLint:

   ```js:title=.eslintrc.js
   module.exports = {
     root: true,
     overrides: [
       {
         files: ['*.ts', '*.tsx'],
         processor: '@graphql-eslint/graphql',
         parser: "@typescript-eslint/parser",
         extends: [
           "eslint:recommended",
           "plugin:@typescript-eslint/recommended"
         ],
         env: {
           es6: true,
         },
       },
       {
         files: ['*.graphql'],
         parser: '@graphql-eslint/eslint-plugin',
         plugins: ['@graphql-eslint'],
         rules: {
           '@graphql-eslint/no-anonymous-operations': 'error',
           '@graphql-eslint/naming-convention': [
             'error',
             {
               OperationDefinition: {
                 style: 'PascalCase',
                 forbiddenPrefixes: ['Query', 'Mutation', 'Subscription', 'Get'],
                 forbiddenSuffixes: ['Query', 'Mutation', 'Subscription'],
               },
             },
           ],
         },
       },
     ],
   }
   ```

1. Create a `graphql.config.js` at the root of your project with the following contents:

   ```js:title=graphql.config.js
   module.exports = require("./.cache/typegen/graphql.config.json")
   ```

1. Start the Gatsby development server with `gatsby develop` and check that `.cache/typegen/graphql.config.json` was created.

You can now use `npm run lint` and `npm run lint:fix` to check your GraphQL queries, e.g. if they are named.

## Additional Resources

- [Gatsby with TypeScript](/docs/how-to/custom-configuration/typescript)
- [VSCode GraphQL Plugin](https://marketplace.visualstudio.com/items?itemName=GraphQL.vscode-graphql)
- [IntelliJ GraphQL Plugin](https://plugins.jetbrains.com/plugin/8097-graphql)
- [gatsby-config Option](/docs/reference/config-files/gatsby-config/#graphqltypegen)
