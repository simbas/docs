---
layout: ~/layouts/MainLayout.astro
title: Content Collections (Experimental)
description: Content collections help organize your Markdown and type-check your frontmatter with schemas.
i18nReady: false
---

Content collections help organize your Markdown or MDX and type-check your frontmatter with schemas. You may reach for collections if you:
- Have a medium-to-large number of documents to manage and fetch (example: a blog with 50+ posts).
- Want to enforce frontmatter fields, and fail if fields are missing (example: every blog post should have a title and description).
- Plan to use content in multiple areas of your site (landing pages, footers, navigation, etc).

## Glossary

- **Schema:** a way to codify the "structure" of your data. In this case, frontmatter data
- **Collection:** a set of data that share a common schema. In this case, Markdown and MDX files
- **Entry:** A piece of data (Markdown or MDX file) belonging to a given collection

## The `src/content/` directory

This RFC introduces a new, reserved directory for Astro to manage: `{srcDir}/content/`. This directory is where all collections and schema definitions live, relative to your configured source directory.

## The `.astro` cache directory

Since we will preprocess your post frontmatter separate from Vite ([see background](#background)), we need a new home for generated metadata. This will be a special `.astro` directory **generated by Astro** at build time or on dev server startup.

We expect `.astro` to live at the base of your project directory. This falls inline with generated directories like `.vscode` for editor tooling and `.vercel` for deployments.

To clarify, **the user will not view or edit files in the `.astro` directory.** However, since it will live at the base of their project, they are free to check this generated directory into their repository.

## The `astro:content` module

Content Schemas introduces a new virtual module convention for Astro using the `astro:` prefix. Since all types and utilities are generated based on your content, we are free to use whatever name we choose. We chose `astro:` for this proposal since:
1. It falls in-line with [NodeJS' `node:` convention](https://2ality.com/2021/12/node-protocol-imports.html).
2. It leaves the door open for future `astro:` utility modules based on your project's configuration.

The user will import helpers like `getCollection` and `getEntry` from `astro:content` like so:

```tsx
import { getCollection, getEntry } from 'astro:content';
```

Users can also expect full auto-importing and intellisense from their editor.


## Creating a collection

All entries in `src/content/` **must** be nested in a "collection" directory. This allows you to get a collection of entries based on the directory name, and optionally enforce frontmatter types with a schema. This is similar to creating a new table in a database, or a new content model in a CMS like Contentful.

What this looks like in practice:

```bash
src/content/
  # All newsletters have the same frontmatter properties
  newsletters/
    week-1.md
    week-2.md
    week-3.md
  # All blog posts have the same frontmatter properties
  blog/
    columbia.md
    enterprise.md
    endeavour.md
```

### Nested directories

Collections are considered **one level deep**, so you cannot nest collections (or collection schemas) within other collections. However, we *will* allow nested directories to better organize your content. This is vital for certain use cases like internationalization:

```bash
src/content/
  docs/
    # docs schema applies to all nested directories 👇
    en/
    es/
    ...
```

All nested directories will share the same (optional) schema defined at the top level. Which brings us to...

## Adding a schema

Schemas are an optional way to enforce frontmatter types in a collection. To configure schemas, you can create a `src/content/config.{js|mjs|ts}` file. This file should:

1. `export` a `collections` object, with each object key corresponding to the collection's folder name. We will offer a `defineCollection` utility similar to `defineConfig` in your `astro.config.*` today (see example below).
2. Use a [Zod object](https://github.com/colinhacks/zod#objects) to define schema properties. The `z` utility will be built-in and exposed by `astro:content`.

For instance, say every `blog/` entry should have a `title`, `slug`, a list of `tags`, and an optional `image` url. We can specify each object property like so:

```ts
// src/content/config.ts
import { z, defineCollection } from 'astro:content';

const blog = defineCollection({
  schema: {
    title: z.string(),
    slug: z.string(),
    // mark optional properties with `.optional()`
    image: z.string().optional(),
    tags: z.array(z.string()),
  },
});

export const collections = { blog };
```

You can also include dashes `-` in your collection name using a string as the key:

```ts
const myNewsletter = defineCollection({...});

export const collections = { 'my-newsletter': myNewsletter };
```

### Why Zod?

We chose [Zod](https://github.com/colinhacks/zod) since it offers key benefits over plain TypeScript types. Namely:
- specifying default values for optional fields using `.default()`
- checking the *shape* of string values with built-in regexes, like `.url()` for URLs and `.email()` for emails

```tsx
...
// "language" is optional, with a default value of english
language: z.enum(['es', 'de', 'en']).default('en'),
// allow email strings only. i.e. "jeff" would fail to parse, but "hey@blog.biz" would pass
authorContact: z.string().email(),
// allow url strings only. i.e. "/post" would fail, but `https://blog.biz/post` would pass
canonicalURL: z.string().url(),
```

You can [browse Zod's documentation](https://github.com/colinhacks/zod) for a complete rundown of features.

## Fetching content

Astro provides 2 functions to query collections:

- `getCollection` - get all entries in a collection, or based on a filter
- `getEntry` - get a specific entry in a collection by file name

These functions will have typed based on collections that exist. In other words, `getCollection('banana')` will raise a type error if there is no `src/content/banana/`.

```tsx
---
import { getCollection, getEntry } from 'astro:content';
// Get all `blog` entries
const allBlogPosts = await getCollection('blog');
// Filter blog posts by entry properties
const draftBlogPosts = await getCollection('blog', ({ id, slug, data }) => {
  return data.status === 'draft';
});
// Get a specific blog post by file name
const enterprise = await getEntry('blog', 'enterprise.md');
---
```

### Return type

Assume the `blog` collection schema looks like this:

```tsx
// src/content/config.ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  schema: {
    title: z.string(),
    slug: z.string(),
    image: z.string().optional(),
    tags: z.array(z.string()),
  },
});

export const collections = { blog };
```

`await getCollection('blog')` will return entries of the following type:

```tsx
{
  // parsed frontmatter
  data: {
    title: string;
    slug: string;
    image?: string;
    tags: string[];
  };
  // unique identifier file path relative to src/content/[collection]
  // example below would reflect the file names in your project
  id: 'file-1.md' | 'file-2.md' | ...;
	// URL-ready slug computed by stripping the file extension from `id`
	slug: 'file-1' | 'file-2' | ...;
  // raw body of the Markdown or MDX document
  body: string;
}
```

:::note
The `body` is the *raw* content of the file. This ensures builds remain performant by avoiding expensive rendering pipelines. See [“Moving to `src/pages/`"](#mapping-to-srcpages) to understand how a `<Content />` component could be used to render this file, and pull in that pipeline only where necessary.
:::

### Nested directories

[As noted earlier](#nested-directories), you may organize entries into directories as well. The result will **still be a flat array** when fetching a collection via `getCollection`, with the nested directory reflected in an entry’s `id`:

```tsx
const docsEntries = await getCollection('docs');
console.log(docsEntries)
/*
-> [
	{ id: 'en/getting-started.md', slug: 'en/getting-started', data: {...} },
	{ id: 'en/structure.md', slug: 'en/structure', data: {...} },
	{ id: 'es/getting-started.md', slug: 'es/getting-started', data: {...} },
	{ id: 'es/structure.md', slug: 'es/structure', data: {...} },
	...
]
*/
```

This is in-keeping with our database table and CMS collection analogies. Directories are a way to organize your content, but do *not* effect the underlying, flat collection → entry relationship.

## Mapping to `src/pages/`

We imagine users will want to map their collections onto live URLs on their site. This should be  similar to globbing directories outside of `src/pages/` today, using `getStaticPaths` to generate routes dynamically.

Say you have a `docs` collection subdivided by locale like so:

```bash
src/content/
	docs/
		en/
			getting-started.md
			...
		es/
			getting-started.md
			...
```

We want all `docs/` entries to be mapped onto pages, with those nested directories respected as nested URLs. We can do the following with `getStaticPaths`:

```tsx
// src/pages/docs/[...slug].astro
import { getCollection } from 'astro:content';

export async function getStaticPaths() {
	const blog = await getCollection('docs');
	return blog.map(entry => ({
		params: { slug: entry.slug },
	});
}
```

This will generate routes for every entry in our collection, mapping each entry slug (a path relative to `src/content/docs`) to a URL. 