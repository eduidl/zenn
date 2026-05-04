---
name: zenn-publishing
description: Use when editing or publishing content in a Zenn CLI repository, especially when handling article images. Prefer Zenn-managed repository images over GitHub raw links, place image files under `/images`, reference them with absolute `/images/...` paths, and follow Zenn CLI image constraints and preview workflow.
metadata:
  short-description: Zenn CLI article and image workflow
---

# Zenn Publishing

Use this skill when working in a Zenn CLI repository that contains `articles/` and optionally `books/`.

## Image policy

When an article or book uses images, do not leave GitHub raw URLs in the markdown if the image should be managed by Zenn's GitHub repository integration.

Instead:

1. Put image files under the repository root `images/` directory.
2. Organize per article when useful, for example `images/my-article/image1.png`.
3. Reference images from markdown with absolute paths that start with `/images/`.

Examples:

```md
![](/images/my-article/overview.png)
![](/images/shared/logo.png)
```

Do not use:

```md
![](../images/foo.png)
![](https://raw.githubusercontent.com/...)
```

## Constraints

- Supported extensions: `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`
- Maximum file size: 3MB per image
- Relative paths are invalid for Zenn CLI local image preview

## Workflow

1. If the article currently points at GitHub raw images, download or copy them into `images/`.
2. Update markdown references to `/images/...`.
3. Confirm the files exist and satisfy the size and extension limits.
4. Use `npx zenn preview` to verify rendering when needed.
5. Remember that pushing the repository uploads these images to Zenn through GitHub repository integration.

## Operational notes

- Deleting an image from the GitHub repository also removes it from Zenn.
- Replacing an image can take around a minute to fully refresh after deploy.
- Local preview serves repository images from the `/images/` path.
