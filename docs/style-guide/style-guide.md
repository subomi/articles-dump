# Honeybadger Blog Style Guide

Blog posts are written in a simple subset of markdown.

For code style we use the [prettier](https://prettier.io) markdown formatter. They have extensions for VSCode and other editors.

## Things we don't support

Our blog design expects posts to have very simple formatting. We disallow the following:

- List items containing multiple paragraphs or code blocks
- Lists with more than one level of nesting
- Tables
- Images that are inline with a paragraph of text
- Headings like H4, H5 or any HN where N > 3
- Footnotes, or footnote style URLs for links and images.

If you have any of these in your article, we will ask you to fix them. We'll eventually have automated linting.

## What we DO support

### Headings

```markdown
# H1: Only one of these, at the top, with your article title

## H2: This is your main heading type, used throughout the article

### H3: This is a subheading
```

### Bold and Italics

```markdown
_italic text_
**bold text**
```

### Lists

List items shouldn't be more than a few sentences. They should never contain multiple paragraphs, code blocks, images, etc.

Avoid nesting lists more than 1 level deep. We're not writing code. :)

```markdown
- Bulleted
- Lists
  - 1 level indentation max
```

```markdown
1. Numbered
2. Lists

- 1 level indentation max
```

### Links and images

Please include the URL for links and images inline. Don't use the footnote style.

```markdown
[link text](href)
```

We prefer wide images over tall, narrow images. Please give us the highest resolution you have. We will scale them down if needed.

If you source the image from somewhere else, please credit the source and provide a link.

```markdown
![Image alt text](image_path)
_Optional image caption_
```

### Code

Use single back-ticks for code included inline with other text. Don't use them on a separate line by themselves.

```markdown
And then I said `foo=bar` and she said OMG!
```

Use three back-ticks for code blocks. Please try to avoid gigantic code blocks. Code blocks should have an appropriate language specified. For shell commands use `bash`. For plain text omit the language specifier.

````markdown
```ruby
a = "block of code"
```
````
j