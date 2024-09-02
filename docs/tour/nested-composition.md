import CodeBlock from '@theme/CodeBlock';
import ImportsNested from '@site/static/bespoke-d2/imports-nested.d2';
import ServiceB from '@site/static/bespoke-d2/serviceB.d2';
import Data from '@site/static/bespoke-d2/data.d2';

# Nested composition

Imports make large compositions much more manageable.

Large diagrams like the ones that take you from high-level overview to low-level details
becomes possible to cleanly construct.

Rendering `overview.d2` gives us a nested diagram while each file is kept flat and
readable.

### `overview.d2`
<CodeBlock className="language-d2-incomplete">
    {ImportsNested}
</CodeBlock>

### `serviceB.d2`
<CodeBlock className="language-d2-incomplete">
    {ServiceB}
</CodeBlock>

### `data.d2`
<CodeBlock className="language-d2-incomplete">
    {Data}
</CodeBlock>

## Render of `overview.d2`

<embed src={require('@site/static/img/generated/imports-nested.pdf').default} width="100%" height="800"
 type="application/pdf" />
