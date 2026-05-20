# proj-coach-schemas

## To install dependencies:

```bash
bun install
```

To run:

```bash
bun run index.ts
```

This project was created using `bun init` in bun v1.2.16. [Bun](https://bun.sh) is a fast all-in-one JavaScript runtime.


## To publish with legacy pipeline

1. create a tag locally. First make sure to be on main branch tip, then check version in package.json (need to bump in each release)

`git tag v${version}`

2. push tag

`git push origin v${version}`

## Publish with new pipeline (temporarily broken)

1. Create a PR without a version bump.
2. When merged to main, the GitHub action will bump the version and release automatically.