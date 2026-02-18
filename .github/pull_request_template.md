## Summary

<!-- What does this PR change and why? -->

## Type of change

- [ ] Bug fix (existing workflow behaved incorrectly)
- [ ] New feature (new workflow, action, or input)
- [ ] Breaking change (removes or renames an input, changes behaviour)
- [ ] Documentation only

## Breaking change checklist

If this is a breaking change, complete all items before requesting review:

- [ ] Major version increment planned (`v1` → `v2`)
- [ ] Migration guide written (add to `docs/` or PR description)
- [ ] All service repos using `@v1` identified and listed below
- [ ] Communication plan agreed with team

**Service repos affected:**
- `bb-claw/` (list affected repos)

## Testing

- [ ] Tested against at least one service repo on a feature branch
- [ ] Actions run URL: <!-- paste the URL here -->
- [ ] All jobs passed end-to-end (including deploy-prod and smoke-test-prod)

## Backwards compatibility

- [ ] All existing `inputs:` still accepted (no renames, no removals)
- [ ] New inputs have sensible defaults so existing callers don't need changes
- [ ] Workflow outputs are unchanged (or additions only)

## Documentation

- [ ] `CHANGELOG.md` updated under `[Unreleased]`
- [ ] Relevant `docs/*.md` files updated if behaviour changed
- [ ] Inline YAML comments updated if step logic changed
