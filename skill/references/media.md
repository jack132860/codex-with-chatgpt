# ChatGPT-generated media


The connector remains read-only. It can view supported PNG/JPEG/GIF/WebP/SVG
files with `read_image`, but it cannot write into the repository or retrieve a
browser download by itself.

When the user asks ChatGPT web to generate an image or video:

1. Generate it in the workspace's saved ChatGPT conversation using the same
   built-in browser tab and connector rules above.
2. Activate the finished asset's actual Download control through the visible
   ChatGPT UI. Browser screenshots are navigation evidence only; never save,
   crop, rename, or import a screenshot as the requested asset.
3. Import the original download through the local execution harness:
   `c2c asset import -w <ws> --from <downloaded-file> --to <new-workspace-relative-path> --json`.
4. The destination must be new and project-relative. The importer validates
   PNG/JPEG/GIF/WebP/SVG/MP4/MOV/WebM content, rejects active SVG and path
   escapes, and never overwrites an existing file. Include the imported path in
   EXECUTED/review.
