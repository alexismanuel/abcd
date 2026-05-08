# XML tags over Jinja for template syntax

Contexts use XML-style self-closing tags (`<ref name="summary" />`, `<source name="transcript" />`) instead of Jinja-style `{{ ref('summary') }}`. Input is a self-closing tag; expansion wraps the content in opening/closing tags. This makes input and output syntax consistent — the expanded output is just the input tag with content injected. XML tags are native to markdown, unambiguous to parse, and don't collide with mermaid diagrams or other `{{ }}` usage in markdown.
