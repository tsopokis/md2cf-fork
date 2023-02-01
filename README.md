# md2cf

A tool and library to convert documents written in Markdown to Confluence Storage format and upload them to a Confluence instance.

## Features

  - **Convert Markdown documents:** a library implementing a
    [Mistune](https://github.com/lepture/mistune) renderer that outputs
    Confluence Storage Format.
  - **Talk to the Confluence API:** an embedded micro-implementation of
    the [Confluence Server REST
    API](https://developer.atlassian.com/server/confluence/confluence-server-rest-api/)
    with basic support for creating and updating pages.
  - **Upload your documents automatically:** a full-featured command line utility that can automate the
    upload process for you.

## Installation

```bash
pip install md2cf
```

If you're only planning on using `md2cf` as a command to upload documents to Confluence, it's highly recommended to install it via [pipx](https://pypa.github.io/pipx/).

```bash
pipx install md2cf
```

## Getting started

Run `md2cf --help` to see all the options and parameters.

In order to upload a document, you will need to supply at least the following five parameters:

  - The **URL** of your Confluence instance, including the path to
    the REST API (e.g. `http://confluence.example.com/rest/api`)
  - Either
    - The **username** and **password** to log into the instance, or
    - a **personal access token**
  - The **space** in which to publish the page
  - The **files or directories** to be uploaded -- if none are specified, the contents will be read from the standard input

Example basic usage:

```bash
md2cf --host 'https://confluence.example.com/rest/api' --username foo --password bar --space TEST document.md
```

Or, if using a token:

```bash
md2cf --host 'https://confluence.example.com/rest/api' --token '2104v3ryl0ngt0k3n720' --space TEST document.md
```

> :warning: Entering your password (or your token) as a parameter on the command line is [generally a bad idea](https://unix.stackexchange.com/q/78734). If you're running the script interactively, you can omit the `--password` parameter and the script will securely ask you to type it.

> :warning: Tokens work differently [Confluence Cloud](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/) and [self-hosted instances](https://confluence.atlassian.com/enterprise/using-personal-access-tokens-1026032365.html). With Confluence Cloud, you have to use your token **as your password** with the `--username` and `--password` parameters. With self-hosted instances, you should use the `--token` parameter.

You can also supply the hostname, username, password, token, and space as **environment variables**:

- `CONFLUENCE_HOST`
- `CONFLUENCE_USERNAME`
- `CONFLUENCE_PASSWORD`
- `CONFLUENCE_TOKEN`
- `CONFLUENCE_SPACE`.

If you're using self-signed certificates and/or want to **ignore SSL errors**, add the `--insecure` option.

You can specify **multiple files and/or entire folders**. If you specify a folder, it will be traversed recursively and all files ending in `.md` will be uploaded. See [Uploading folders](#uploading-folders) for more information.

If you just want to get a preview of what `md2cf` would do, the `--dry-run` option will print a list of page data but leave Confluence untouched.

## Page information arguments

### Page title

The **title** of the page can come from a few sources, in order of priority from highest to lowest:
* the `--title` command line parameter
* a `title` entry in your document's front matter, i.e. a YAML block delimited by `---` lines at the top of the file
  ```yaml
  ---
  title: This is a title
  ---
  # Rest of the document here
  ```
* the first top-level header found in the document (i.e. the first `#` header)
* the filename if there are no top-level headers.

Note that if you're reading from standard input, you must either specify the title through the command line or have a title in the content as a header or in the front matter.

If you want to strip the top level header from the document, so the title isn't repeated in the body of the page, pass the `--strip-top-header` parameter.

If you're uploading entire folders, you might want to add a prefix to each page title in order to avoid collisions. You can do this using the `--prefix` parameter.

### Removing extra newlines

If your document uses single newlines to break lines, for example if it was typeset with a fixed column width, Confluence Cloud might respect those newlines and produce a document that's difficult to read. Use the `--remove-text-newlines` parameter to replace every newline within a paragraph with a space.

<details>
<summary>Example</summary>
For example, this will turn

```text
This is a document
with hardcoded newlines
in its paragraphs.

It's not that nice
to read.
```

into

```text
This is a document with hardcoded newlines in its paragraphs.

It's not that nice to read.
```
</details>

### Adding a preface and/or postface

The `--preface-markdown`, `--preface-file`, `--postface-markdown`, and `--postface-file` commands allow you to add some text at the top or bottom of each page. This is useful  if you're mirroring documentation to Confluence and want people to know that it's going to be overwritten in an automated fashion.

`--preface-markdown` and `--postface-markdown` allow you to specify Markdown text right on the command line. If you don't specify anything, it defaults to a paragraph saying

> **Contents are auto-generated, do not edit.**

`--preface-file` and `--postface-file` take a path to a markdown file and will prepend or append the contents to every page.

> :warning: Preface and postface Markdown is parsed separately and added to the body after the main page has been parsed, so it won't influence behaviour tied to the page contents, such as title or front matter detection.

### Page labels

You can specify labels for your page by adding a `labels` entry in your document's front matter, i.e. a YAML block delimited by `---` lines at the top of the file

```yaml
---
labels:
- first label
- second label
---
# Rest of the document here
```

By default, labels will only be added. If you want the final set of labels to _exactly_ match what you listed in the front matter, pass the `--replace-all-labels` option.

### Parent page

If you want to upload the page under **a specific parent**, you can supply the parent's page ID as the `--parent-id` parameter, or its title through the `--parent-title` parameter.

### Update message

You can also optionally specify an **update message** to describe the
change you just made by using the `--message` parameter. Note that if you're using the `--only-changed` option there will also be a hash of the page/attachment contents at the end of the version update message.

### Updating an existing page

Uploading a page with the same title twice will update the existing one.

If you want to update a page by page ID, use the `--page-id` option. This allows you to change the page's title, or to update a page with a title that is annoying to use as a parameter.

With the `--minor-edit` option you can prevent notifications being sent to watcher of the page. This corresponds to the "Notify watchers" checkbox when editing pages manually.

### Avoiding uploading content that hasn't changed

If you want to avoid redundant uploads (and the corresponding update emails) when your content hasn't changed, you can add the `--only-changed` option. Note that this will store a hash of the page/attachment contents at the end of the version update message.

## Linking to other documents (relative links)

Relative link support is disabled by default. You can enable it by passing `--enable-relative-links`. They work similarly to [GitHub relative links](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes#relative-links-and-image-paths-in-readme-files), with the difference that links starting with `/` are **not supported** and will be left as they are.

> :warning: This function requires two uploads for every page containing relative links, since a page needs to be uploaded to Confluence first before we can know which URL it's assigned.
>
> The first upload will replace all internal links with placeholders, and once we know where all the pages ended up, the placeholders will be replaced with the final Confluence link.

By default, relative links that point to a file that doesn't exist (or is not being uploaded in the current batch) will result in an error. If you want to ignore this and keep them as they are, pass `--ignore-relative-link-errors`.

## Directory arguments

### Uploading folders recursively

`md2cf` can upload entire folders for you. This can be useful if you want to mirror some in-repo documentation to Confluence. When uploading entire folders, `md2cf` will recursively traverse all subdirectories and upload any `.md` file it encounters.

By default, `md2cf` will honour your `.gitignore` and skip any files or folders it defines. If you want to avoid this, add the `--no-gitignore` option.

Folders will be represented by empty pages in the final upload, since Confluence can only nest pages under other pages. You can modify this behaviour through three command line parameters.

#### Customizing folder names

Folder names like `interesting-subsection` or `dir1` are not particularly nice. If you pass the `--beautify-folders` option,
all spaces and hyphens in folder names will be replaced with spaces and the first letter will be capitalized, producing
`Interesting subsection` and `Dir1`.

Alternatively, you can create a YAML file called `.pages` with the following format in every folder you wish to rename.
If you pass the `--use-pages-file`, the folder will be given that title.

```yaml
title: "This is a fantastic title!"
```

#### Collapse single pages

You can collapse directories that only contain one document by passing the `--collapse-single-pages` parameter.

<details>
<summary>Example</summary>
This means that a folder layout like this:

```text
document.md
folder1/
  documentA.md
  documentB.md
folder2/
  other-document.md
```

will be uploaded to Confluence like this:

```text
document
folder1/
  documentA
  documentB
other-document
```
</details>

#### Dealing with empty folders

Passing `--skip-empty` will not create pages for empty folders.

<details>
<summary>Example</summary>
```text
document.md
folder1/
  folder2/
    folder3/
      other-document.md
folderA/
  interesting-document.md
    folderB/
      folderC/
        lonely-document.md
```

will be uploaded as:

```text
document
folder3/
  other-document
folderA/
  interesting-document
  folderC/
    lonely-document
```
</details>

Alternatively, you can specify `--collapse-empty` to merge empty folders together.

<details>
<summary>Example</summary>
```text
document.md
folder1/
  folder2/
    folder3/
      other-document.md
folderA/
  interesting-document.md
    folderB/
      folderC/
        lonely-document.md
```

will be uploaded as:

```text
document
folder1/folder2/folder3/
  other-document
folderA/
  interesting-document
  folderB/folderC/
    lonely-document
```
</details>

## Terminal output format

By default, `md2cf` produces rich output with animated progress bars that is meant for human consumption. If the output is piped to a file, no animations will be played and only the final result will be printed. All error messages are printed to standard error.

`md2cf` supports two other output formats.

### JSON output

Passing `--output json` will make `md2cf` print the JSON output for each page as returned by Confluence. Normal progress output will not be printed.

> :warning: JSON entries will only be printed for page creation/updates. They will not be printed for attachment creation/updates and will not be printed for second-pass updates for [relative links](#linking-to-other-documents-relative-links).

### Minimal output

Passing `--output minimal` will make `md2cf` print the final Confluence URL for each page, similarly to versions prior to `2.0.0`. Normal progress output will not be printed.

> :warning: URLs will only be printed for page creation/updates. They will not be printed for attachment creation/updates and will not be printed for second-pass updates for [relative links](#linking-to-other-documents-relative-links).

## Library usage

`md2cf` can of course be used as a Python library. It exposes two useful modules: the renderer and the API wrapper.

### Renderer

Use the `ConfluenceRenderer` class to generate Confluence Storage Format
output from a Markdown document.

```python
import mistune
from md2cf.confluence_renderer import ConfluenceRenderer

markdown_text = "# Page title\n\nInteresting *content* here!"

renderer = ConfluenceRenderer(use_xhtml=True)
confluence_mistune = mistune.Markdown(renderer=renderer)
confluence_body = confluence_mistune(markdown_text)
```

### API

md2cf embeds a teeny-tiny implementation of the Confluence Server REST
API that allows you to create, read, and update pages.

```python
from md2cf.api import MinimalConfluence

confluence = MinimalConfluence(host='https://example.com/rest/api', username='foo', password='bar')

confluence.create_page(space='TEST', title='Test page', body='<p>Nothing</p>', update_message='Created page')

page = confluence.get_page(title='Test page', space_key='TEST')
confluence.update_page(page=page, body='New content', update_message='Changed page contents')
```
