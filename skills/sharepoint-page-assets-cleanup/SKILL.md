---
name: sharepoint-page-assets-cleanup
description: |-
  Inspect modern SharePoint pages for image assets and identify unused files in SiteAssets/SitePages/<Page> folders. Present an auditable cleanup inventory, ask before recycling anything, recycle safely, and verify references remain.

  Use when the user says:
    - "list images this SharePoint page uses"
    - "clean up unused page assets"
    - "check this page for images that aren't needed"
    - "find unused files in the SitePages asset folder"
---
# SharePoint page asset cleanup

## When to use
Use when Stefan asks to list images used by a modern SharePoint page or identify and safely clean unused assets in `SiteAssets/SitePages/<Page>` folders.

Do not use page screenshots or the page DOM. Work through SharePoint REST using `mcp__claude-in-chrome__javascript_tool` in a browser tab already on the target site so the signed-in session provides authentication.

## Inputs
- Target site URL
- Target modern page server-relative path
- Optional page-asset folder path; otherwise derive `SiteAssets/SitePages/<Page>` from the page name
- Whether the request is inventory-only or cleanup after confirmation

## Steps

### 1. Read page metadata
1. Read the welcome page with `GET {site}/_api/web/rootfolder?$select=WelcomePage`.
2. Read the target page fields with:
   `GET {site}/_api/web/GetFileByServerRelativeUrl('{serverRelPath}')/ListItemAllFields?$select=Id,Title,BannerImageUrl,CanvasContent1,LayoutWebpartsContent,PageLayoutType`
3. Use `Accept: application/json;odata=nometadata`.
4. Decode HTML entities twice with a textarea element. Normalize `/` and `\/` to `/`.
5. Extract image URLs using this case-insensitive regex:
   `(?:https?:\\/\\/[^\s"'<>()&?]+|\\/sites\\/[^\s"'<>()&?]+)\\.(?:jpe?g|png|gif|svg|webp)`
6. Deduplicate and group images as:
   - Site-hosted in SiteAssets
   - Microsoft stock images from `cdn.hubblecontent.osi.office.net`
   - Banner image
7. Flag a banner still using default `sitepagethumbnail.png`, duplicate uploads such as several `bg-top.png` copies, and timestamped re-uploads.

### 2. Find unused files
1. List direct files and subfolders with:
   `GET {site}/_api/web/GetFolderByServerRelativeUrl('{folder}')/Files?$select=Name,Length,TimeLastModified`
   and the folder's `/Folders` endpoint.
2. Build a reference set across every page on the site, not only the target page:
   `GET {site}/_api/web/lists/getbytitle('Site Pages')/items?$select=FileRef,CanvasContent1,LayoutWebpartsContent,BannerImageUrl&$top=500`
3. Decode each field and lower-case the combined content.
4. Treat an asset as used when either its filename or URL-encoded filename occurs in that blob. Match by filename, never full path, because pages may vary path casing.
5. Keep original images that support Image web part shape crops, for example `123-name.jpeg` supporting `shape-123-name-<timestamp>.png`.
6. Treat extensionless GUID-named cached stock images as used when referenced.

### 3. Return a cleanup proposal before changes
Never modify assets during an inventory scan.

Report total, used, unused, and unused MB. List each unused candidate with name, size, modified date, and rationale. Ask with AskUserQuestion before recycling. Offer to retain icon-style files whose names start with neither a digit nor `shape-`, such as `org-chart@2x.png`, because they may be reusable Quick Links icons.

State the scan limits:
- It cannot see references from other sites, news rollups, or hub navigation.
- It cannot see references in custom CSS or SPFx extensions.
- It cannot account for older page versions that might be restored and then show broken images.

### 4. Recycle only after explicit confirmation
1. Request a form digest with `POST {site}/_api/contextinfo` and read `FormDigestValue`.
2. Recycle every confirmed candidate individually using:
   `POST {site}/_api/web/GetFileByServerRelativePath(decodedurl='{encodeURIComponent(path after doubling apostrophes)}')/recycle()`
3. Send the digest in `X-RequestDigest`.
4. Never call `DELETE` and never empty the recycle bin. Explain that recycled files can be restored for 93 days.
5. Report `recycled n/n`; list every failure with the corresponding HTTP status code.

### 5. Verify
List the folder again. Confirm all files classified as used still exist and report `referenced files missing: 0` when true. State the remaining file count and MB freed in one or two sentences.

## Output format
### Inventory
- **Page:** title and server-relative path
- **Images in use:** site-hosted, stock, banner; include flags
- **Asset folder:** path, total files, used files, unused candidates, unused MB

### Cleanup proposal
| Candidate | Size | Modified | Reason |
|---|---:|---|---|

Include the scan limits and an explicit confirmation question before recycling.

### After recycling
- **Recycled:** n/n
- **Failures:** none, or file names with HTTP status codes
- **Verification:** referenced files missing: n
- **Remaining:** file count and MB freed

## Safety constraints
- Never expose raw `CanvasContent1`, full query strings, or tokens in tool output.
- Store large results in `window.__x`; return slices of about 25 lines, using `~/` for the asset-folder prefix and `[hubble]` for the stock CDN prefix.
- If the browser tool fails or returns empty data, say so plainly and don't invent a cleanup recommendation.
