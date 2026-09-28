# GitHub Discussions (gh CLI)

`gh` has no porcelain for Discussions — use GraphQL via `gh api graphql`.
Auth: token needs `write:discussion` scope. Check once per session:
`gh auth status` (see `references/auth.md`).

**Permission:** any authenticated user who can *view* a public repo can create a
discussion there (docs: Discussions quickstart). `repository { viewerPermission }`
= `READ` is enough. Managing/deleting needs triage/write.

## Fetch repo + category IDs (do this every time, never hardcode)

```graphql
query {
  repository(owner:"OWNER", name:"REPO") {
    id
    hasDiscussionsEnabled
    discussionCategories(first:20) { nodes { id name slug } }
  }
}
```

```bash
gh api graphql -f query='...'   # jq/cat the nodes; IDs are opaque and typo-prone
```

`hasDiscussionsEnabled` is the correct field — `discussionsEnabled` does not exist
on `Repository` and errors with `undefinedField`.

## Create

```graphql
mutation($repositoryId: ID!, $categoryId: ID!, $title: String!, $body: String!) {
  createDiscussion(input: {repositoryId: $repositoryId, categoryId: $categoryId,
                           title: $title, body: $body}) {
    discussion { url number }
  }
}
```

```bash
# long body: write to a file and pass with @path (avoids shell quoting hell)
gh api graphql -f query="$QUERY" \
  -F repositoryId="$REPO_ID" -F categoryId="$CATEGORY_ID" \
  -f title="$TITLE" -F body=@/tmp/body.md
```

A wrong category ID returns `NOT_FOUND: Could not resolve to a node with the global
id of 'DIC_...'` — re-fetch the ID, don't hand-correct it.

## Verify (never claim published without a fresh read)

```graphql
query { repository(owner:"O", name:"R") { discussion(number:N) { title url category { name } body } } }
```

Check `title`, `category.name`, and the first line of `body` match what you sent.
Common failure: `body` truncated or empty when the `@file` path was wrong.

## Listing / replying

- List: `repository { discussions(first:20, orderBy:{field:UPDATED_AT,direction:DESC})
  { nodes { number title url } } }`
- Reply: `addDiscussionComment(input:{discussionId:..., body:...})` — note the name is
  `addDiscussionComment`, NOT `createDiscussionComment` (the latter errors with
  `undefinedField` on `Mutation`). `discussionId` takes the discussion's global ID
  (`D_kw...`), which comes from `discussion { id }`, not the number.
- Category slug → ID mapping matters: `ideas`, `general`, `q-a`, `announcements`,
  `polls`, `show-and-tell`.
