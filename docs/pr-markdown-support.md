## Summary

Add Tree-sitter Markdown support to Pommel, enabling semantic search over `.md`, `.markdown`, `.mdown`, and `.mkdn` files. This allows AI coding agents to search documentation alongside code. Brings total supported languages to 14.

## Related Issue

Relates to expanding language support for non-code files.

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [x] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)
- [ ] Documentation update
- [ ] Refactoring (no functional changes)
- [ ] Performance improvement
- [x] Test coverage improvement

## Changes Made

**9 files changed, +573 lines**

- Add markdown grammar import and registry entry in `internal/chunker/treesitter.go`
- Add `LangMarkdown` constant and parser initialization
- Add `DetectLanguage` case for `.md`, `.markdown`, `.mdown`, `.mkdn` extensions
- Create `languages/markdown.yaml` with chunk mappings (headings→class, code blocks/lists→method)
- Add `extractMarkdownName()` in `internal/chunker/generic.go` for markdown-specific name extraction (markdown nodes lack a "name" field unlike code constructs)
- Add `scripts/test-markdown.sh` for testing on markdown-heavy codebases

### Integration Tests Added

- `TestIntegration_MarkdownFileChunking` - Full chunking pipeline
- `TestIntegration_MarkdownWithCodeBlocks` - Code block extraction
- `TestIntegration_MarkdownHeadings` - Heading hierarchy
- `TestIntegration_MarkdownEmptyFile` - Edge case handling
- `TestIntegration_MarkdownDeterministicIDs` - ID stability

## Testing

### How has this been tested?

- [x] Unit tests added/updated
- [x] Integration tests added/updated
- [x] Manual testing performed

### Test commands run:

```bash
go test ./internal/chunker/... -v -run "TestIntegration_Markdown"
go test ./internal/chunker/...
go build -tags "fts5" ./...
```

### Manual testing steps:

1. Add `'**/*.md'` to `include_patterns` in `.pommel/config.yaml`
2. Run `pm reindex`
3. Run `pm search "contributing guidelines"`
4. Expected: Markdown files appear in search results with proper chunking

## Checklist

- [x] My code follows the project's code style
- [x] I have run `gofmt` and `go vet`
- [x] I have added tests that prove my fix/feature works
- [x] All new and existing tests pass
- [x] I have updated documentation (if applicable)
- [x] My changes don't introduce new warnings
- [x] This PR targets the `dev` branch (not `main`)

## Additional Notes

**Chunk mapping rationale:**

| Node Type | Chunk Level | Why |
|-----------|-------------|-----|
| `atx_heading` | class | Section headers define document structure |
| `setext_heading` | class | Alternative heading syntax |
| `fenced_code_block` | method | Code snippets are highly searchable |
| `indented_code_block` | method | Legacy code block syntax |
| `list` / `list_item` | method | Lists as searchable units |

**Recommended config for markdown-heavy projects:** Use `batch_size: 8` in embedding config to avoid Ollama 500 errors during batch indexing.

**Reference:** Uses [smacker/go-tree-sitter/markdown](https://github.com/smacker/go-tree-sitter/tree/master/markdown) grammar.
