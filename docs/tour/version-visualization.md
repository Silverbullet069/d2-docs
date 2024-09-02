---
pagination_next: tour/imported-template
---
import CodeBlock from '@theme/CodeBlock';
import ImportsVVHistory from '@site/static/d2/imports-vv-history.d2';
import UsersCurrent from '@site/static/d2/users-current.d2';
import UsersV01 from '@site/static/d2/users-v0.1.d2';

# Version visualization

## History

You want to understand how the schema of a system has evolved over time. As long as the
diagram is modularized with imports, such a visualization is easy to whip up.

- `history.d2`
<CodeBlock className="language-d2-incomplete">
    {ImportsVVHistory}
</CodeBlock>

- `users.d2` (latest version, 0.2)
<CodeBlock className="language-d2-incomplete">
    {UsersCurrent}
</CodeBlock>

- `users.d2` (0.1)
<CodeBlock className="language-d2-incomplete">
    {UsersV01}
</CodeBlock>

Since you want how `users.d2` looked like at `v0.1`, you use `git` to get that version:

```sh
cp users.d2 users-current.d2
git checkout tags/v0.1 users.d2
mv users.d2 users-v0.1.d2
```

<div className="embedSVG" dangerouslySetInnerHTML={{__html: require('@site/static/img/generated/imports-vv-history.svg2')}}></div>

## Compare

You're a manager at Apple who has two teams secretly working on the same product for
multiple years. After years of iterating in their silos, you form a committee to compare
the two results. The evaluation starts by comparing their design decisions in an
overarching diagram projected in a dark room behind a closed door.

- `compare.d2`
```d2-incomplete
Team Alpha: {
  Quick facts: |md
    - 3 L6 engineers
  |
  Schema: {
    ...@alpha-schema
  }
  API: {
    ...@alpha-api
  }
  # etc etc
}

Team Charlie: {
  Quick facts: |md
    - 2 L5
    - 5 L4
    - 15 L3
  |
  Schema: {
    ...@charlie-schema
  }
  API: {
    ...@charlie-api
  }
  # etc etc
}
```

And then you checkout the corresponding diagrams from the different repositories.
```sh
gh repo clone apple/team-alpha
gh repo clone apple/team-charlie

cp apple/team-alpha/schema.d2 alpha-schema.d2
cp apple/team-charlie/schema.d2 charlie-schema.d2

cp apple/team-alpha/api.d2 alpha-api.d2
cp apple/team-charlie/api.d2 charlie-api.d2
```

The rendered diagram is left as an exercise to the reader.
