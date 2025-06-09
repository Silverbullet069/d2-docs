
# D2 Oracle

D2 has an API built on top of its AST for **programmatically creating diagrams in Go**.
This package is `d2/d2oracle`.

This API is exercised heavily by Terrastruct to implement bidirectional edits. We have comprehensive test coverage of these functions. If there's any confusion from the docs, there's almost certainly a test that answers your question. (We're also happy to help, just file a GitHub issue!)

For a blog post detailing an example usage (building a SQL table diagram), see https://terrastruct.com/blog/post/generate-diagrams-programmatically/.

No mutations

All functions in `d2oracle` are pure: they do not mutate the original graph, they return a new one. If you are chaining calls, don't forget to use the resulting graph from the previous call.

## Create

Create a shape or connection.

```
func Create(g *d2graph.Graph, boardPath []string, key string) (newG *d2graph.Graph, newKey string, _ error)

```

**Parameters**:

- `g`: D2 graph
- `boardPath`: Path of the board to modify. Only applicable to multi-board diagrams;
  otherwise, pass in `nil`.
- `key`: The ID of the shape or connection being created

**Return**:

- `newG`: The modified D2 graph
- `newKey`: The actual ID of the created shape or connection created

Everything specified in the given `key` will be created. So
for example, if you create a connection between 2 shapes that don't exist, they will be
created in the same call. If you specify a nested object, it will create the parent
containers if they do not exist.

```
// This call creates 6 shapes and 1 connection
d2oracle.Create(g, nil, "a.b.c -> x.y.z")

```

If you call `Create` twice with the same shape ID, you will get an
error. If you call it twice with the same edge ID, you will create another edge.

`newKey` is the ID of the object created. This doesn't always match the input key.

For shapes, there may be an ID collision. `Create` appends a counter in this case.

```
// newKey = "a"
g, newKey, _ = d2oracle.Create(g, nil, "a")
// newKey = "a 2"
_, newKey, _ = d2oracle.Create(g, nil, "a")

```

Connection IDs include the index.

```
// newKey = "(a -> b[0])"
g, newKey, _ = d2oracle.Create(g, nil, "a -> b")
// newKey = "(a -> b[1])"
_, newKey, _ = d2oracle.Create(g, nil, "a -> b")

```

If you have a multi-board diagram like so:

```
x

layers: {
  y: {}
}

```

```
// This creates a shape "a" at the root
g, _ = d2oracle.Create(g, nil, "a")
// This creates a shape "a" at layer "y"
g, _ = d2oracle.Create(g, []string{"y"}, "a")

```

## Set

Set an attribute on a shape or connection.

```
func Set(g *d2graph.Graph, boardPath []string, key string, tag, value *string) (newG *d2graph.Graph, _ error)

```

**Parameters**:

- `g`: D2 graph
- `boardPath`: Path of the board to modify. Only applicable to multi-board diagrams;
  otherwise, pass in `nil`.
- `key`: The identifier for the attribute
- `tag`: The language tag. This is only non-nil when text values are set that can be
  different languages, e.g. a code snippet value.
- `value`: Value being set

**Return**:

- `newG`: The modified D2 graph

The shape or connection that `Set` is modifying must exist.

```
g, _, _ := d2oracle.Create(g, "a")
g, _ = d2oracle.Set(g, "a.style.fill", nil, "red")

```

If the attribute is arleady set, it is overwritten.

```
// D2 graph: "a.style.fill: red"
g, _ = d2oracle.Set(g, "a.style.fill", nil, "red")
// D2 graph: "a.style.fill: blue"
g, _ = d2oracle.Set(g, "a.style.fill", nil, "blue")

```

Connections are targeted with an index.

```
// Set the label of the first connection created
g, _ = d2oracle.Set(g, "(a -> b)[0].style.label", nil, "uno")
// Set the label of the second connection created
g, _ = d2oracle.Set(g, "(a -> b)[1].style.label", nil, "dos")

```

To unset an attribute, just pass `nil`.

```
g, _ = d2oracle.Set(g, "a.style.fill", nil, nil)
g, _ = d2oracle.Set(g, "(a -> b)[0].style.label", nil, nil)

```

If you do not pass an attribute and just give the ID of a shape or connection, it will set
that object's primary value (usually label).

```
// Sets `a`'s label to be Markdown text
g, _ = d2oracle.Set(g, "a", "md", "# I am A")

```

## Delete

Delete a shape or connection.

```
func Delete(g *d2graph.Graph, boardPath []string, key string) (newG *d2graph.Graph, _ error)

```

**Parameters**:

- `g`: D2 graph
- `boardPath`: Path of the board to modify. Only applicable to multi-board diagrams;
  otherwise, pass in `nil`.
- `key`: The identifier for the shape or connection

**Return**:

- `newG`: The modified D2 graph

If you specify a container with children, those children will be deleted too.

```
g, _, _ := d2oracle.Create(g, "a.b")
// `a.b` will also be deleted
g, _ = d2oracle.Delete(g, "a")

```

## Rename

Rename the ID of a shape or connection.

Note that the ID != label. If you want to change the label, use `Set`.

```
func Rename(g *d2graph.Graph, boardPath []string, key, newName string) (newG *d2graph.Graph, err error)

```

**Parameters**:

- `g`: D2 graph
- `boardPath`: Path of the board to modify. Only applicable to multi-board diagrams;
  otherwise, pass in `nil`.
- `key`: The current identifier for the shape or connection
- `newName`: The new identifier for the shape or connection

**Return**:

- `newG`: The modified D2 graph

Rename will rename all references of the given key.

```
g, _, _ := d2oracle.Create(g, "a.b -> a.c")
// New D2: `z.b -> z.c`
g, _ = d2oracle.Rename(g, "a", "z)

```

## Move

Move a given shape or connection to a different container.

```
func Move(g *d2graph.Graph, boardPath []string, key, newKey string) (newG *d2graph.Graph, err error)

```

If you give two keys of the same scope (e.g. "a" to "b"), it's the same as `Rename`.

**Parameters**:

- `g`: D2 graph
- `boardPath`: Path of the board to modify. Only applicable to multi-board diagrams;
  otherwise, pass in `nil`.
- `key`: The current identifier for the shape or connection
- `newKey`: The new identifier for the shape or connection

**Return**:

- `newG`: The modified D2 graph

You can move things out of containers, into container, or across containers

```
g, _, _ := d2oracle.Create(g, "a.b.c -> x.y.z")
// Move c out of b into a
g, _ = d2oracle.Move(g, "a.b.c", "a.c")
// Move c back into b
g, _ = d2oracle.Move(g, "a.c", "a.b.c")
// Move c into x.y.z
g, _ = d2oracle.Move(g, "a.b.c", "x.y.z.c")

```

## ID Deltas

For calls which can affect more than one ID, there is an API for getting a map of every single ID change.
This can be hard to keep track of; for example, if you move a container with many
descendants, the ID of all those descendants have changed, as well as all the connections
that reference anything within the container.

When would you use this? If you are keeping state of D2 objects somewhere else other than
the graph, e.g. in your own storage or writing back to somewhere, these calls should be
used to keep track of all programmatic changes.

Delta methods:

- MoveIDDeltas
- DeleteIDDeltas
- RenameIDDeltas

Each of these have the same input parameters as their counterparts, and return a string to
string map of ID changes.

Given this input D2 script:

```
x
y
x -> z

```

`deltas, err := MoveIDDeltas(g, nil, "x", "y.x")`

`deltas`:

```
{
  "(x -> z)[0]": "(y.x -> z)[0]",
  "x": "y.x"
}

```


# Autoformat

You almost never have to think about style decisions like indentation, newlines,
number of hyphens, or spacing. D2's auto-formatter will format your D2 file for you on
compile, keeping all your declarations consistent and readable, effortlessly.

If your file is

```
aws_s3:    AWS S3 California{
  Monitoring ---------->California
}

```

When you compile, it becomes

```
aws_s3: AWS S3 California {
  Monitoring -> California
}

```

## Running formatter

If you're using the `d2` CLI, you can run the formatter on files with

```
d2 fmt file.d2

```

The formatter is meant to be integrated into plugins and extensions which automatically
call the formatter upon file writing. This functionality is dependent on the plugin.

# Cheat Sheet

Click the preview to download the PDF.

![d2 cheat sheet](https://terrastruct-site-assets.s3.us-west-1.amazonaws.com/documents/d2_cheat_sheet.pdf)

# Classes

Classes let you aggregate attributes and reuse them.

```
direction: right
classes: {
  load balancer: {
    label: load\nbalancer
    width: 100
    height: 200
    style: {
      stroke-width: 0
      fill: "#44C7B1"
      shadow: true
      border-radius: 5
    }
  }
  unhealthy: {
    style: {
      fill: "#FE7070"
      stroke: "#F69E03"
    }
  }
}

web traffic -> web lb
web lb.class: load balancer

web lb -> api1
web lb -> api2
web lb -> api3

api2.class: unhealthy

api1 -> cache lb
api3 -> cache lb

cache lb.class: load balancer

```

## Connection classes

As a reminder of D2 syntax, you can apply classes to connections both at the initial
declaration as well as afterwards.

On initial declaration:

```
a -> b: {class: something}

```

Targeting:

```
a -> b
# ...
(a -> b)[0].class: something

```

## Overriding classes

If your object defines an attribute that the class also has defined, the object's
attribute overrides the class attribute.

```
classes: {
  unhealthy: {
    style.fill: red
  }
}
x.class: unhealthy
x.style.fill: orange

```

## Multiple classes

You may use arrays for the value as well to apply multiple classes.

```
classes: {
  d2: {
    label: ""
    icon: https://play.d2lang.com/assets/icons/d2-logo.svg
  }
  sphere: {
    shape: circle
    style.stroke-width: 0
  }
}

logo.class: [d2; sphere]

```

### Order matters

When multiple classes are given, they are applied left-to-right.

```
classes: {
  uno: {
    label: 1
  }
  dos: {
    label: 2
  }
}

x.class: [uno; dos]
y.class: [dos; uno]

```

## Advanced: Using classes as tags

If you want to post-process D2 diagrams, you can also use classes to arbitrarily tag
objects. Any `class` you apply is written into the SVG element as a `class` attribute. So
for example, you can then apply custom CSS like `.stuff { ... }` (or use Javascript for
onclick handlers and such) on a web page that D2 SVG is embedded in.

# Comments

D2 has line comments and block comments.

## Line comments

Line comments begin with a hash.

They can be added as their own line

```
# Comments start with a hash character and continue until the next newline or EOF.
x -> y

```

Or at the end of a line

```
x -> y # I am at the end

```

## Block comments

Block comments begin and end with three double quotes:

```
x -> y

"""
This is a
block comment
"""

y -> z

```


# Getting help & community

- We have a Discord channel here, https://discord.gg/NF6X8K4eDq. If you have a
  question or need help, you'll find responses quickest through this channel.
- Have a proposal, running into a bug, or anything else you want to discuss with the team
  and community? D2 uses GitHub Issues to track all
  TODOs, and GitHub Discussions for
  feature requests and ideas.
- If you have a private enquiry, please email us: hi@d2lang.com.

# Intro to Composition



This section's documentation is incomplete. We'll be adding more to this section soon.

D2 has built-in mechanisms for you to compose multiple boards into one diagram.

For example, this is a composition of 2 boards, exported as an animated SVG:

The way to define another board in a D2 diagram is to use 1 of 3 keywords. Each of these
declare boards with different inheritance rules.

| Keyword     | Inheritance                                       |
| ----------- | ------------------------------------------------- |
| `layers`    | Boards which do not inherit. They are a new base. |
| `scenarios` | Boards which inherit from the base layer.         |
| `steps`     | Boards which inherit from the previous step.      |

Each one serves different use cases. The example above is achieved by defining a Scenario
(the scenario of when we have to deploy a hotfix).

Thus far, all D2 diagrams we've encountered are single-board diagrams, the root board.

```
# Root board
x -> y

```

Composition in D2 is when you use one of those keywords to declare another board.

```
# Root board
x -> y
layers: {
  # Board named "numbers" that does not inherit anything from root
  numbers: {
    1 -> 2
  }
}

```

So now we have two boards: root and `numbers`. They cannot be visible at the same time of
course, so exports have to accommodate these more dynamic diagrams, such as the animated
SVG you see above.

Composition is one of D2's most powerful features, as you'll see from the use cases in this
section.

# Export formats

Since a diagram composed of multiple boards can't be represented as a single image, the
export options are different.

- Multiple SVGs
  - The default output writes multiple SVG files to your output path.
  - They are nicely organized on the file system, with appropriate directories and file
    names.
  - Internal board links are rewritten to point to those file paths, so they work if you
    open the SVG and just click on it.
- Single animated SVG
  - This is useful for small compositions. It's great for a small number of Steps or
    flipping between some Scenarios, but if the composition has too many boards, the
    viewer may get confused or wait until a loop.
  - On the CLI, passing `--animate-interval=1200` sets the resulting SVG to be an
    animation that stays on each board for 1200ms.
- Single animated GIF
  - Same as above, but useful in different contexts, like where SVG is not rendered.
- PDF
  - PDF can be multiple pages, so this medium is suitable for multi-board diagrams.
    Each board is simply another page, and objects can be linked to those pages.
  - This is the most suitable medium for layers currently.
- PowerPoint
  - Specify the output as `.pptx` to get a PowerPoint presentation.
  - Works on Google Slides, for easy web sharing of multi-board diagrams.
  
  
  
  
  
  Contents

# Atlassian Confluence app

Your browser does not support the video tag.



This app is for D2 Studio and requires an account on D2 Studio.

## Seamless integration between D2 Studio and Confluence

- Configure once for the team
- Every team member can choose from diagrams on D2 Studio
- Permission settings so that only team diagrams are visible
- Every time diagram updates in D2 Studio, it's updated on the Confluence page

Install Confluence app

# Connections

Connections define relationships between shapes.

## Basics

Hyphens/arrows in between shapes define a connection.

```
Write Replica Canada <-> Write Replica Australia

Read Replica <- Master
Write Replica -> Master

Read Replica 1 -- Read Replica 2

```

If you reference an undeclared shape in a connection, it's created (as shown in the [hello world](https://d2lang.com/tour/hello-world) example).



There are 4 valid ways to define a connection:

- `--`
- `->`
- `<-`
- `<->`

### Connection labels

```
Read Replica 1 -- Read Replica 2: Kept in sync

```

### Connections must reference a shape's key, not its label.

```
be: Backend
fe: Frontend

# This would create new shapes
Backend -> Frontend

# This would define a connection over existing labels
be -> fe

```

## Example

```
Write Replica Canada <-> Write Replica Australia

Read Replica <- Master

x -- y

super long shape id here -> super long shape id even longer here

```

## Repeated connections

Repeated connections do not override existing connections. They declare new ones.

```
Database -> S3: backup
Database -> S3
Database -> S3: backup

```

## Connection chaining

For readability, it may look more natural to define multiple connection in a single line.

```
# The label applies to each connection in the chain.
High Mem Instance -> EC2 <- High CPU Instance: Hosted By

```

## Cycles are okay

```
Stage One -> Stage Two -> Stage Three -> Stage Four
Stage Four -> Stage One: repeat

```

## Arrowheads

To override the default arrowhead shape or give a label next to arrowheads, define a special shape on connections named `source-arrowhead` and/or `target-arrowhead`.

```
a: The best way to avoid responsibility is to say, "I've got responsibilities"
b: Whether weary or unweary, O man, do not rest
c: I still maintain the point that designing a monolithic kernel in 1991 is a

a -> b: To err is human, to moo bovine {
  source-arrowhead: 1
  target-arrowhead: * {
    shape: diamond
  }
}

b <-> c: "Reality is just a crutch for people who can't handle science fiction" {
  source-arrowhead.label: 1
  target-arrowhead: * {
    shape: diamond
    style.filled: true
  }
}

d: A black cat crossing your path signifies that the animal is going somewhere

d -> a -> c

```

Arrowhead options

- `triangle` (default)
  - Can be further styled as `style.filled: false`.
- `arrow` (like triangle but pointier)
- `diamond`
  - Can be further styled as `style.filled: true`.
- `circle`
  - Can be further styled as `style.filled: true`.
- `box`
  - Can be further styled as `style.filled: true`.
- `cf-one`, `cf-one-required` (cf stands for crows foot)
- `cf-many`, `cf-many-required`



It's recommended the arrowhead labels be kept short. They do not go through
autolayout for optimal positioning like regular labels do, so long arrowhead labels are
more likely to collide with surrounding objects.

caution

If the connection does not have an endpoint, arrowheads won't do anything.

For example, the following will do nothing, because there is no source arrowhead.

```
x -> y: {
  source-arrowhead.shape: diamond
}

```

## Referencing connections

You can reference a connection by specifying the original ID followed by its index.

```
x -> y: hi
x -> y: hello

(x -> y)[0].style.stroke: red
(x -> y)[1].style.stroke: blue

```


# Containers

```
server
# Declares a shape inside of another shape
server.process

# Can declare the container and child in same line
im a parent.im a child

# Since connections can also declare keys, this works too
apartment.Bedroom.Bathroom -> office.Spare Room.Bathroom: Portal

```

## Nested syntax

You can avoid repeating containers by creating nested maps.

```
clouds: {
  aws: {
    load_balancer -> api
    api -> db
  }
  gcloud: {
    auth -> db
  }

  gcloud -> aws
}

```

## Container labels

There are two ways define container labels.

### 1\. Shorthand container labels

```
gcloud: Google Cloud {
  ...
}

```

### 2\. Reserved keyword `label`

```
gcloud: {
  label: Google Cloud
  ...
}

```

## Example

```
clouds: {
  aws: AWS {
    load_balancer -> api
    api -> db
  }
  gcloud: Google Cloud {
    auth -> db
  }

  gcloud -> aws
}

users -> clouds.aws.load_balancer
users -> clouds.gcloud.auth

ci.deploys -> clouds

```

## Reference parent

Sometimes you want to reference something outside of the container from within. The
underscore ( `_`) refers to parent.

```
christmas: {
  presents
}
birthdays: {
  presents
  _.christmas.presents -> presents: regift
  _.christmas.style.fill: "#ACE1AF"
}

```


# Dagre

Dagre is D2's default layout engine.

## Reference

https://github.com/dagrejs/dagre

## Pros

- Very fast.
- Battle tested (thanks to MermaidJS, which exclusively uses Dagre for its flowcharts).
- Generally good results.
- Theory behind algorithms are the papers that power Graphviz.
- Renders hierarchical layouts well.

## Cons

- Unmaintained. Development stopped in 2018.
- Makes some inexplicable edge routing decisions occasionally
  ( https://github.com/dagrejs/dagre/issues/256).
- Layout algorithm is strictly hierarchical, even if underlying diagram is not
  hierarchical.
- Container child to another container (or another container child) is not natively
  supported by dagre. D2 has added a shim to make it work, but there's some core algorithm
  considerations that are missed due to the shim.
- Multi-segment edge routes are curved, instead of orthogonal. Can result in unaesthetic
  squiggly lines.

## Gallery


# Design decisions

The following are design decisions that guide the development of D2. We've tried our best
to avoid the mistakes of the past and take inspiration from the most successful modern
programming and configuration languages.

Design decisions inherently mean tradeoffs, some of which you may disagree with. But, if
you're a programmer, D2 is built for you, and we believe you'll find the sum of these
decisions to be a language that makes you feels at home.

These will inevitably evolve as the language continues to evolve.

## Readability > prototyping speed

Both readability and prototyping speed are important, but when a decision is one or the
other, D2 usually favors readability.

Most of the time, it's not either/or. Good programming language features usually result in
higher vectors in both directions. D2's syntax is light and designed such that autofmt
always gets it right for you, consistent across projects.

Hopefully you'll find a good balance between ease of use, prototyping speed, and
readability, in that order. What D2 specifically avoids is terse, compact syntax that
inhibits all three.

For example, here's two ways to define a cylinder.

D2:

```
A: Christmas {shape: cylinder}

```

Mermaid:

```
A[(Christmas)]

```

D2's is a little less compact but a lot more readable. It's also more writeable, by which
I mean you don't forget that a cylinder is called a cylinder, but it's easy to forget that
`[(x)]` is a cylinder.

## Warnings > errors

D2 will compile whenever possible. For example, say you apply a class that doesn't exist,
or add a style that has no effect on a particular shape. If the user error is one that D2
can ignore, it will compile successfully and, at most, warn. There's nothing more annoying
than commenting out some code while debugging, and getting a stop-the-world error message
that you have an unused variable.

## Good defaults

Default, zero-customization D2 should produce good diagrams. That requires being
opinionated on what a good default should be. For example, D2 ships with a default theme.
Instead of keeping things open-ended with monochrome shapes, pleasant colors are the
default.

## Optimized for desktops and servers

D2 has a robust CLI with a built-in watch mode, maintained `man` page, and allows reading
from stdin and writing to stdout. Images and fonts are by default embedded into the
diagram so that exported diagrams are standalone -- they'll look the same everywhere. D2
supports a wide variety of formats like PPT and GIF. It allows imports, such that you can
modularize your diagram into multiple files. There's a language API to programmatically
edit and write D2. All of these are antithetical to a web library for browser rendering.
D2 intends to ship and maintain a web library for that purpose, but it'll be trimmed down
from the full feature set and secondary in priority.

## Singular use case: documenting software

D2 is focused on being useful to software engineers for documenting software. We are not a
general-purpose visualization tool. Other languages may support mind maps, Gantt charts,
Sankey, venn diagrams, and have the capability to draw a map of the United States. D2 does
none of those and will not support these.

In D2, these are considered bloat. Stretching a language thin across such a large surface
area of different diagram types essentially splits it into N different mini-DSLs within a
DSL. Syntax can hardly evolve when it has to support N completely different diagram types.
And it's counter to the original purpose of a DSL which is to make a subset task more
convenient.

## Design the system, not the diagram

The purpose of D2 is to describe the system you're documenting. The language should make a
clear separation between design of the system and design of the diagram.

Consider what it takes to customize styles in a Graphviz diagram:

```
digraph "Linux_kernel_diagram" {
    fontname="Helvetica,Arial,sans-serif"
    node [fontname="Helvetica,Arial,sans-serif"]
    edge [fontname="Helvetica,Arial,sans-serif"]
    graph [\
        newrank = true,\
        nodesep = 0.3,\
        ranksep = 0.2,\
        overlap = true,\
        splines = false,\
    ]
    node [\
        fixedsize = false,\
        fontsize = 24,\
        height = 1,\
        shape = box,\
        style = "filled,setlinewidth(5)",\
        width = 2.2\
    ]
    edge [\
        arrowhead = none,\
        arrowsize = 0.5,\
        labelfontname = "Ubuntu",\
        weight = 10,\
        style = "filled,setlinewidth(5)"\
    ]

```

Imagine if you couldn't separate HTML and CSS and it all had to be inlined.

Of course, good aesthetics are essential to good documentation. D2 certianly prioritizes
aesthetics, but it must be decoupled with the content.

D2 is the only language that allows you to define just nodes and edges, and import all the
styles in a separate file, and swap out that file for different aesthetics.

## Whiteboard-fit

While "graph" and "diagram" are often interchangeable terms, for D2's purposes, a diagram
is a simplified representation that can fit on a large whiteboard. After a certain number
of nodes and edges, e.g. 1000 nodes, the representation becomes more like a graph from
graph theory than a software architecture diagram. Their use case is less understanding
each individual shape and connection and more seeing the general patterns. D2 is not
designed for this use case. There are much better tools for that.

!graph example

# Dimensions

You can specify the `width` and `height` of most shapes.



These keywords cannot be set on containers, since containers resize to fit their children.

```
direction: right

small jerry: "" {
  shape: image
  icon: https://static.wikia.nocookie.net/tomandjerry/images/4/46/JerryJumbo3-1-.jpg
  width: 200
  height: 200
}

med jerry: "" {
  shape: image
  icon: https://static.wikia.nocookie.net/tomandjerry/images/4/46/JerryJumbo3-1-.jpg
  width: 300
  height: 300
}

big jerry: "" {
  shape: image
  icon: https://static.wikia.nocookie.net/tomandjerry/images/4/46/JerryJumbo3-1-.jpg
  width: 500
  height: 400
}

big jerry -> med jerry -> small jerry

```


# Discord plugin

!D2 + Discord

### The fastest way to explain what you mean mid-conversation

Keep your projects moving by connecting Terrastruct and Discord. Compile D2 codeblocks by opening the context menu on a D2 codeblock, going to `Apps` and clicking `Compile D2`.

Configure your exports by using `/d2` inside Discord.

Add to Discord Your browser does not support the video tag.

# Editor Support

We have first class language support for both Vim and VS Code.

Automatic indentation and syntax highlighting are fully supported and make working with
the D2 language far more pleasant.

- https://github.com/terrastruct/d2-vim
- https://github.com/terrastruct/d2-vscode

The syntax highlighting will even catch basic errors for you if your color theme
highlights illegal syntax.

# ELK

ELK is a mature, hierarchical layout, actively maintained by an academic research group at
[Christian Albrechts University in Kiel](https://www.rtsys.rmatik.uni-kiel.de/en/team).

## Reference

https://www.eclipse.org/elk/reference.html

## Pros

- Clean, orthogonal routes.
- Highly customizable.
- Fast.
- Good at minimizing crossings.
- Natively supports container to container routing, handling these better than dagre.
- Undergoing active improvements with regular releases.
- Routes SQL tables with exact columns.

## Cons

- Strictly hierarchical, like dagre.
- Some routes have unnecessary bends.
- Minimal consideration for symmetry.

## Gallery


# A diagramming dev tool

D2 is designed towards a single goal: **turn diagramming into a pleasant experience for**
**engineers**. Plenty of tools can claim to do that for simple diagrams, but you stop having
a good time as soon as you get to even slightly complex diagrams — the ones that most need
to exist.

Why is that? Because most diagramming tools today are design tools, not dev tools. They
give you a blank canvas and a drag-and-drop toolbar like you'd see on Figma or Photoshop,
and treat their intended workflow as a design process. Engineers are not visual designers,
and the lack of ability to spatially architect a system should not block the creation of
valuable documentation. Every drag and drop shouldn’t require planning, and updates
shouldn’t be a frustrating exercise in moving things around and resizing to make room for
the new piece. Declarative Diagramming removes that friction.

Before Hashicorp introduced Terraform to let engineers write infrastructure as text, they
were clicking around AWS and Google Cloud consoles to configure their infrastructure.
Nowadays, that's just unprofessional. Where's the review process, the rollback steps, the
history and version control? It's hard to believe that the future of visual documentation
in companies around the world will predominantly be made with drag-and-drop design tools.

# Exports

On the CLI, you may export `.d2` into

- SVG
- PNG
- PDF
- PPTX
- GIF
- Stdout

## SVG

```
d2 in.d2 out.svg

```

SVG is the default export format on the CLI. If you don't specify an output, the export
file will be the input name as an SVG file.

For example, `d2 in.d2` will produce a file named `in.svg`.

The resulting SVG has CSS injected into it. This, along with the use of HTML
`<foreignObject>` s used to make Markdown work, means that the SVG is meant to be viewed in
a web context. For example, opening it up in your browser, embedding it onto a webpage. It
may not look right without a web context, like in Inkscape or Adobe Illustrator.

On the CLI, if you pass in `-`

- for the input, it reads D2 from stdin
- for the output, it writes SVG to stdout

## PNG

```
d2 in.d2 out.png

```

PNG exports work by Playwright spinning up a
headless browser, putting the SVG onto it, and taking a screenshot. The first invocation
of Playwright will download its dependencies, if they don't already exist on the machine.



If you get a message like `err: failed to launch Chromium`, you can try installing
Playwright dependencies outside of D2 on your machine. For example:

```
npm install -g @playwright
npx playwright install --with-deps chromium

```

See #744 for more.

## PDF

```
d2 in.d2 out.pdf

```

PDF exports are the result of taking PNG exports and placing them on PDF pages, along with
headers and fonts. As such, dependencies needed for PNG exports are also needed for PDF
exports.

PDF is _more_ interactive than PNG, but _less_ interactive than SVG.

For example, `animate` keyword won't show up in PDF exports like they would in SVG.

But `link` s can still be clickable in PDFs.

!linked PDF example in D2

## PPTX

```
d2 in.d2 out.pptx

```

Similar to PDF exports. This export format is useful for giving presentations when used
with composition (e.g. diagram with multiple Layers, Scenarios, Steps).

## GIF

```
d2 in.d2 out.gif

```

This export format is useful for giving presentations when used with short compositions.
For example, show two Scenarios, show a couple of steps. Something that the audience can
digest in a loop that lasts a couple of seconds without needing to flip through it
manually.

## Stdout

D2 accepts `-` in place of the input and/or output arguments. SVG is used as the format
for Stdout output.

For example, this writes a D2 script of `x -> y` and outputs it to a file `example.svg`.

```
echo "x -> y" | d2 - - > example.svg

```


# Overview

## Official

Officially developed and maintained by the creators of D2.

- VSCode extension
- Vim plugin
- Obsidian plugin
- D2 app for Slack
- Discord plugin
- Confluence app

## Community

Community developed plugins and extensions. Have one you'd like to share? Please open an
issue and we're happy to include your's!

- Tree-sitter grammar: https://git.pleshevski.ru/pleshevskiy/tree-sitter-d2
- Tree-sitter grammar 2: https://github.com/ravsii/tree-sitter-d2
- Emacs major mode: https://github.com/andorsk/d2-mode
- Goldmark extension: https://github.com/FurqanSoftware/goldmark-d2
- Telegram bot: https://github.com/meinside/telegram-d2-bot
- Postgres importer: https://github.com/zekenie/d2-erd-from-postgres
- Structurizr to D2 exporter: https://github.com/goto1134/structurizr-d2-exporter
- MdBook preprocessor: https://github.com/danieleades/mdbook-d2
- Org-mode support: https://github.com/xcapaldi/ob-d2
- ROS2 D2 Exporter: https://github.com/Greenroom-Robotics/ros-d2
- Python D2 diagram builder: https://github.com/MrBlenny/py-d2
- Clojure D2 transpiler: https://github.com/judepayne/dictim
- JavaScript D2 diagram builder: https://github.com/Kreshnik/d2lang-js
- CIL (C#, Visual Basic, F#, C++ CLR) to D2: https://github.com/HugoVG/AppDiagram
- D2 Snippets (for text editors): https://github.com/Paracelsus-Rose/D2-Language-Code-Snippets
- MongoDB to D2: https://github.com/novuhq/mongo-to-D2
- Pandoc filter: https://github.com/ram02z/d2-filter
- MySQL to D2: https://github.com/JDOsborne1/db\_to\_d2
- Remark D2 code blocks into images: https://github.com/mech-a/remark-d2
- D2 image service: https://github.com/Watt3r/d2-live
- MkDocs plugin: https://github.com/landmaj/mkdocs-d2-plugin
- C# & dotnet SDK: https://github.com/Stephanvs/d2lang-cs
- Zed extension: https://github.com/gabeidx/zed-d2

# Frequently asked questions (FAQ)

- General
  - How does this compare to Mermaid, Graphviz, PlantUML?
  - Is this designed for small diagrams or complex ones?
  - Does the D2 CLI collect telemetry?
  - Does D2 need a browser to run?
  - Can D2 run on a browser?
  - Can I use D2 online?
- Layouts
  - Can an object be part of more than 1 container?
  - Can I specify ports?
- Exports
  - No interactivity in SVG export

## General

### How does this compare to Mermaid, Graphviz, PlantUML?

We've created a website with detailed comparisons:
https://text-to-diagram.com. It is a community effort where
anyone can add examples or request changes or compare features. The maintainers of Mermaid
have contributed to it.

### Is this designed for small diagrams or complex ones?

Both. The syntax is kept minimal and unstructured to make small diagrams with as little
lines as possible. At the same time, the language includes IDE features like an
autoformatter, error messages, and comments to maintain large diagrams.

However, it is not designed for "big data". We do not test D2 on thousands of nodes.

### Does the D2 CLI collect telemetry?

No, D2 does not use an internet connection after installation, except to check for version
updates from GitHub periodically.

### Does D2 need a browser to run?

No, D2 can run entirely server-side.

### Can D2 run on a browser?

Yes, with WebAssembly. D2 runs on https://play.d2lang.com this
way.We are working on including the build with the releases, as well as provide
instructions and examples so you can include it in your browser projects.

### Can I use D2 online?

https://play.d2lang.com

## Layouts

### Can an object be part of more than 1 container?

...e.g., an item in the middle of a venn diagram.

Not currently and not in the near future. See
discussion for more.

### Can I specify ports?

Not currently, but in the near future. See
discussion for more.

## Exports

### No interactivity in SVG export

SVG exports can have some interactive elements when using links and tooltips. However,
interactivity in SVG can be disabled depending on environment. There are several ways to
include SVGs on a web page.

tldr; when it's treated as an image, the interactivity is lost.

| Embedding Method     | Link Clickable |
| -------------------- | -------------- |
| Inline SVG (<svg>)   | Yes            |
| <img> tag            | No             |
| <object> tag         | Yes            |
| <iframe> tag         | Yes            |
| CSS background image | No             |
| <embed> tag          | Yes            |


# Fonts

D2 uses 4 font families:

- Source Sans Pro for the majority of
  text, including labels, Markdown, etc.
- Source Code Pro for code blocks and
  text in Class shape.
- A blend of Architect's Daughter
  and Fuzzy Bubbles for `sketch` mode
  text.

Currently on the CLI, you can customize fonts by replacing Source Sans Pro with your own
TTF files through the following flags:

- `--font-regular`
- `--font-italic`
- `--font-bold`

These should point to a `.ttf` file, for example:

```
d2 --font-regular=./helvetica-regular.ttf input.d2

```

It's advisable to supply either none or all of the fonts, for consistency. If you supply
one but not all of the fonts, it will fall back to Source Sans Pro for the missing styles.
For example, if you give a `--font-regular` and `--font-bold`, then the italic will remain
as Source Sans Pro Italic.



Do you want to customize the fonts for code or sketch mode? Please raise an Issue on
GitHub. We'll support this if there's demand.

# Roadmap

TLDR

https://github.com/terrastruct/d2/issues?q=is%3Aopen+is%3Aissue+label%3Afeature

D2's long-term goal is to significantly reduce the amount of time and effort it takes to
create and maintain high-quality diagrams for every software team. This is a large,
ambitious undertaking that will take many (more) years to get right, and won't ever truly
be "done". It's important to focus on what's most impactful towards increasing its
utility, in more use cases, especially in these early stages. The top priorities today:

. Expand coverage of layouts it can handle.
. Minimize the effort required for aesthetically pleasing diagrams.
. Support integration with popular developer tools.

For each of these, there are survey questions that are used to measure progress.

## Layouts

In some instances, you need a diagram to match the exact image in your head. These
workflows will always require a design tool, with a drag-and-drop GUI. In other cases, you
don't care about exact match, or you can't even picture the end result. The requirements
for these types of diagrams are more relaxed -- it only needs to satisfy a set of
constraints. These are the types of diagrams that an algorithm can get right --
"automatable". The vast majority of software engineering diagrams are automatable.

!freeform diagram example. Source: https://vickiboykis.com/2022/11/18/some-notes-on-the-stable-diffusion-safety-filter/!automatable diagram example

Not automatable. Precise placements, unique shapes.

Automatable. You could show these relationships in 100 different ways that look good.

Currently, D2 can handle a subset of automatable diagrams well, and increasing coverage is
a priority. The primary lever, of course, is layout engines. These are the algorithms that
take shapes, labels, icons, connections and hints as input, and lay it out in such a way
that is "legible".

There are two components to diagram legibility.

. Visual legibility. Minimal noise, clean, aligned, no unnecessary connection crossings,
etc. There is no obvious improvement to be made, like swapping two shapes such that
they're both closer to their respective neighbors.
. Semantic legibility. If your coworker were to draw the same model, would the model be
more understandable to you even if it were visually messier?

Visual legibility is what algorithms are good at, and they should always outperform
humans. The gap increases as diagrams get larger -- it's impossible for me and you to
eyeball 20 swaps that end up reducing overall edge distance by 4 pixels.

Semantic legibility is the bigger challenge. Most layout engines stick to a simple "type"
of diagram, like hierarchical. But how often do you want your software architecture to be
hierarchical? Rarely, and laying it out that way changes the semantics.

### How success is measured

- Visual: Can you spot any obvious improvements to be made?
- Semantic: Turing test -- can you tell the layout was the work of an algorithm vs your
  coworker?

D2 uses Dagre by default. Though it is among the best open-source layout engines (used by
MermaidJS, perhaps the most popular text-to-diagram tool currently), it still leaves much
to be desired. Dagre is unfortunately unmaintained, and only produces hierarchical
layouts. Even if it were to be revived, the research papers it's based off of will not
evolve out of hierarchical layouts. It also doesn't support containers, which is a
non-starter for software architecture diagrams (D2 has shimmed support for it \[2\]).

### Plan

. Integrate with special-purpose layout engines which are "solved". For example, there's
no improvements to be made to a sequence diagram layout engine. Likewise for a git-commit
timeline, or radial layouts, among others.
. We are developing a new general-purpose layout engine from scratch \[3\], testing
exclusively on software architecture diagrams. It can support multiple sub-types in one
diagram, e.g. part of the diagram is a tree, but not all of it has to be. It handles
containers and clusters very well, moves labels and icons to avoid collisions, and much
more.

## Aesthetics

Documentation is only useful if it's consumed, and beautiful diagrams are consumed far
more than ugly ones. For algorithm-generated diagrams to be a suitable replacement to
hand-made, it must be at least as aesthetic. To be consistently preferable to hand-made
ones \[4\], it must be aesthetic out of the box. Customization should then be graduated
levels of how much effort you want to expend. Least effort: choose from pre-made themes
designed by professional designers. More effort: make and edit your own themes. Most
effort: custom-style everything. Most engineers are satisfied at the first and second
levels, but D2 must have good support for granular customization.

### How success is measured

- If you're making a diagram for a large audience (e.g. a blog, company all-hands), would
  you use D2 to make it?

One deciding factor is the time it takes to get to a presentable diagram. Presentable
doesn't mean full of vibrant colors and shadows and design fluff. It does mean consistent
colors that increase clarity, on-brand palette of company colors, distinct shapes and
arrows -- properties that make a professional diagram you'd be proud of presenting.

Another consideration is how customizable the aesthetics are, and how much effort that
customization takes. I wish the bar here could be that customizing via text can get better
than the GUI, but I don't believe this to be a possible outcome. If you want a slightly
different shade of green, are you going to edit the hex code by typing it? Or would you
rather point and click from a gradient. If you want to change the style of 3 objects
you're looking at, you can text-search for the names of each of those 3 objects and change
them individually. But most of us would rather use the superior input device for precision
targeting -- a mouse.

While getting the customization experience to be GUI-level is merely aspirational, D2 can
aim to be at least as good as an existing, battle-tested text-based styling language: CSS.

### Plan

. Build language features to make customization easier. Things like glob targeting (e.g.
x.\*.style.fill: red to make every child of a container red) and classes (define styles
once, reuse as classes). Its catalog of shape and connection types needs to expand.
. Continue investing in themes. Theming is how engineers make beautiful diagrams without
designing. D2 needs to extend what themes can latch onto. Not just colors, but fonts,
background styles (dotted, grids, colors), every aspect of the diagram. They should be
able to target granularly, e.g. for cylinder shapes 2 container-levels deep, use these
styles. And, there should be easy ways to create and modify themes.
. Curate a set of high-quality, consistent icons to bundle with D2.

## Developer Tooling

The better support a dev tool has for existing workflows and familiar tools, the more
useful it is. D2 has official Vim and VSCode extensions. Features within the language
exist solely for bettering editor environments, such as autoformat and being able to parse
broken syntax and output multiple, human-readable error messages.

To further support composability, D2 has a built-in API for manipulating the AST at a
high-level (more intelligent than just CRUD on AST nodes). This lets anyone build up and
edit a diagram programmatically. The goal isn't to enable a specific use case, but rather
unforeseen ones for the infinite workflows out there.

D2's plugin system further makes the language extensible. These allow you to add "hooks"
to stages of D2 compilation. For example, while not core to D2, a hypothetical set of
plugins can add a styled border, add your signature/credits, make everything [look hand-drawn](https://github.com/terrastruct/d2/pull/91), then increase contrast for
accessibility.

### How success is measured

- Does writing and reading D2 in your environment feel "complete"?
- Is it flexible and extendible enough to be used in unforeseen ways?

Completeness: I could not tell you any improvement to Vim's Go plugin if you asked me to
name one \[5\]. It feels complete. However, there are plenty of plugins/extensions I've used
which felt lacking, e.g. the syntax highlighting only identified some basic tokens and
stopped there.

Extensibility: A hallmark of a good developer tool is when its modularity enables
unplanned usage.

### Plan

. D2's plugin system is still in its early stages, and should allow deeper hooks into
parts of compilation.
. Complete editor integrations. Currently, syntax highlighting is supported. But a
feature-complete integration for VSCode for example would allow rendering to its
built-in browser, autoformat, call out to LSP functions to refactor, and more.
. Build out imports/exports. Currently, D2 can take in D2 files and export to SVGs. It
should have a transpiler to import other popular text-to-diagram languages, and output
to other popular image types. It should take in CSVs of schemas to make ERDs. It should
be able to render to ASCII art.
. Build a configurable linter.

## Your feedback

If you have any suggestions or feedback, we'd love to hear from you! Please open an issue
on Github.

## Notes

\[0\] Many other domains are also majority capturable by algorithms, like trees for HR
diagrams. Other domains like biology, the majority of diagrams are custom-drawn.

\[1\] Some diagrams will always require manual labor. For example, if you want to have a
translucent blueprint of an airplane and overlay shapes and lines on specific parts. Good
GUI diagram-makers will always be necessary.

\[2\] MermaidJS seems to have implemented support for containers as well, but it's not
widely released and their live editor still won't allow it.

\[3\] For now, this is closed-source. It's free to download and evaluate. To learn more,
visit https://terrastruct.com/tala.

\[4\] If you still miss the hand-drawn aesthetic, D2 has just the thing:
https://github.com/terrastruct/d2/pull/492

\[5\] Actually, gopls single-handedly brings my machine back to 2005 speeds with how much
CPU it consumes. Not the fault of vim-go though!

# Globs

Etymology

> The glob command, short for global, originates in the earliest versions of Bell Labs' Unix... to expand wildcard characters in unquoted arguments ...

https://en.wikipedia.org/wiki/Glob\_(programming))

Globs are a powerful language feature to make global changes in one line.

```
iphone 10
iphone 11 mini
iphone 11 pro
iphone 12 mini

*.height: 300
*.width: 140
*mini.height: 200
*pro.height: 400

```

## Globs apply backwards and forwards

In the following example, the instructions are as follows:

. Create a shape `a`.
. Apply a glob rule. This immediately applies to existing shapes, i.e., `a`.
. Create a shape `b`. Existing glob rules are re-evaluated, and applied if they meet the
criteria. This does, so it applies to `b`.
. Same with `c`.

```
a

* -> y

b
c

```

## Globs are case insensitive

```
diddy kong
Donkey Kong

*kong.style.fill: brown

```

## Globs can appear multiple times

```
teacher
thriller
thrifter

t*h*r.shape: person

```

## Glob connections

You can use globs to create connections.

```
vars: {
  d2-config: {
    layout-engine: elk
  }
}

Spiderman 1
Spiderman 2
Spiderman 3

* -> *: 👉

```

Notice how self-connections were omitted. While not entirely consistent with what you may
expect from globs, we feel it is more pragmatic for this to be the behavior.

You can also use globs to target modifying existing connections.

```
lady 1
lady 2

barbie

lady 1 -> barbie: hi barbie
lady 2 -> barbie: hi barbie

(lady* -> barbie)[*].style.stroke: pink

```

## Scoped globs

Notice that in the below example, globs only apply to the scope they are specified in.

```
foods: {
  pizzas: {
    cheese
    sausage
    pineapple
    *.shape: circle
  }
  humans: {
    john
    james
    *.shape: person
  }
  humans.* -> pizzas.pineapple: eats
}

```

## Recursive globs

`**` means target recursively.

```
a: {
  b: {
    c
  }
}

**.style.border-radius: 7

```

```
zone-A: {
  machine A
  machine B: {
    submachine A
    submachine B
  }
}

zone-A.** -> load balancer

```

Notice how `machine B` was not captured. Similar to the exception with `* -> *` omitting
self-connections, recursive globs in connections also make an exception for practical
diagramming: it only applies to non-container (AKA leaf) shapes.

## Filters

Use `&` to filter what globs target. You may use any reserved keyword to filter on.

```
bravo team.shape: person
charlie team.shape: person
command center.shape: cloud
hq.shape: rectangle

*: {
  &shape: person
  style.multiple: true
}

```

### Property filters

Aside from reserved keywords, there are special property filters for more specific
targeting.

- `connected: true|false`
- `leaf: true|false`

```
**: {
  &connected: true
  style.fill: yellow
}

**: {
  &leaf: true
  style.stroke: red
}

container: {
  a -> b
}
c

```

### Filters on array values

If the filtered attribute has an array value, the filter will match if it matches any
element of the array.

```
the-little-cannon: {
  class: [server; deployed]
}
dino: {
  class: [internal; deployed]
}
catapult: {
  class: [server]
}

*: {
  &class: server
  style.multiple: true
}

```

### Globs as filter values

Globs can also appear in the value of a filter. `*` by itself as a value for a filter
means the key must be specified.

```
*: {
  &link: *
  style.fill: red
}

x.link: https://google.com
y

```

## Inverse filters

Use `!&` to inverse-filter what globs target.

```
bravo team.shape: person
charlie team.shape: person
command center.shape: cloud
hq.shape: rectangle

*: {
  !&shape: person
  style.multiple: true
}

```

## Nested globs

You can nest globs, combining the features above.

```
conversation 1: {
  shape: sequence_diagram
  alice -> bob: hi
  bob -> alice: hi
}

conversation 2: {
  shape: sequence_diagram
  alice -> bob: hello again
  alice -> bob: hello?
  bob -> alice: hello
}

# Recursively target all shapes...
**: {
  # ... that are sequence diagrams
  &shape: sequence_diagram
  # Then recursively set all shapes in them to person
  **: {shape: person}
}

```

## Global globs

Triple globs apply globally to the whole diagram. The difference between a double glob and
a triple glob is that a triple glob will apply to nested `layers` (see the section on
composition for more on `layers`), as well as persist across imports.

```
***.style.fill: yellow
**.shape: circle
*.style.multiple: true

x: {
  y
}

layers: {
  next: {
    a
  }
}

```

Imports

If you import a file, the globs declared inside it are usually not carried over. Triple
globs are the exception -- since they are global, importing a file with triple glob will
carry that glob as well.

# Globs

Etymology

> The glob command, short for global, originates in the earliest versions of Bell Labs' Unix... to expand wildcard characters in unquoted arguments ...

https://en.wikipedia.org/wiki/Glob\_(programming))

Globs are a powerful language feature to make global changes in one line.

```
iphone 10
iphone 11 mini
iphone 11 pro
iphone 12 mini

*.height: 300
*.width: 140
*mini.height: 200
*pro.height: 400

```

## Globs apply backwards and forwards

In the following example, the instructions are as follows:

. Create a shape `a`.
. Apply a glob rule. This immediately applies to existing shapes, i.e., `a`.
. Create a shape `b`. Existing glob rules are re-evaluated, and applied if they meet the
criteria. This does, so it applies to `b`.
. Same with `c`.

```
a

* -> y

b
c

```

## Globs are case insensitive

```
diddy kong
Donkey Kong

*kong.style.fill: brown

```

## Globs can appear multiple times

```
teacher
thriller
thrifter

t*h*r.shape: person

```

## Glob connections

You can use globs to create connections.

```
vars: {
  d2-config: {
    layout-engine: elk
  }
}

Spiderman 1
Spiderman 2
Spiderman 3

* -> *: 👉

```

Notice how self-connections were omitted. While not entirely consistent with what you may
expect from globs, we feel it is more pragmatic for this to be the behavior.

You can also use globs to target modifying existing connections.

```
lady 1
lady 2

barbie

lady 1 -> barbie: hi barbie
lady 2 -> barbie: hi barbie

(lady* -> barbie)[*].style.stroke: pink

```

## Scoped globs

Notice that in the below example, globs only apply to the scope they are specified in.

```
foods: {
  pizzas: {
    cheese
    sausage
    pineapple
    *.shape: circle
  }
  humans: {
    john
    james
    *.shape: person
  }
  humans.* -> pizzas.pineapple: eats
}

```

## Recursive globs

`**` means target recursively.

```
a: {
  b: {
    c
  }
}

**.style.border-radius: 7

```

```
zone-A: {
  machine A
  machine B: {
    submachine A
    submachine B
  }
}

zone-A.** -> load balancer

```

Notice how `machine B` was not captured. Similar to the exception with `* -> *` omitting
self-connections, recursive globs in connections also make an exception for practical
diagramming: it only applies to non-container (AKA leaf) shapes.

## Filters

Use `&` to filter what globs target. You may use any reserved keyword to filter on.

```
bravo team.shape: person
charlie team.shape: person
command center.shape: cloud
hq.shape: rectangle

*: {
  &shape: person
  style.multiple: true
}

```

### Property filters

Aside from reserved keywords, there are special property filters for more specific
targeting.

- `connected: true|false`
- `leaf: true|false`

```
**: {
  &connected: true
  style.fill: yellow
}

**: {
  &leaf: true
  style.stroke: red
}

container: {
  a -> b
}
c

```

### Filters on array values

If the filtered attribute has an array value, the filter will match if it matches any
element of the array.

```
the-little-cannon: {
  class: [server; deployed]
}
dino: {
  class: [internal; deployed]
}
catapult: {
  class: [server]
}

*: {
  &class: server
  style.multiple: true
}

```

### Globs as filter values

Globs can also appear in the value of a filter. `*` by itself as a value for a filter
means the key must be specified.

```
*: {
  &link: *
  style.fill: red
}

x.link: https://google.com
y

```

## Inverse filters

Use `!&` to inverse-filter what globs target.

```
bravo team.shape: person
charlie team.shape: person
command center.shape: cloud
hq.shape: rectangle

*: {
  !&shape: person
  style.multiple: true
}

```

## Nested globs

You can nest globs, combining the features above.

```
conversation 1: {
  shape: sequence_diagram
  alice -> bob: hi
  bob -> alice: hi
}

conversation 2: {
  shape: sequence_diagram
  alice -> bob: hello again
  alice -> bob: hello?
  bob -> alice: hello
}

# Recursively target all shapes...
**: {
  # ... that are sequence diagrams
  &shape: sequence_diagram
  # Then recursively set all shapes in them to person
  **: {shape: person}
}

```

## Global globs

Triple globs apply globally to the whole diagram. The difference between a double glob and
a triple glob is that a triple glob will apply to nested `layers` (see the section on
composition for more on `layers`), as well as persist across imports.

```
***.style.fill: yellow
**.shape: circle
*.style.multiple: true

x: {
  y
}

layers: {
  next: {
    a
  }
}

```

Imports

If you import a file, the globs declared inside it are usually not carried over. Triple
globs are the exception -- since they are global, importing a file with triple glob will
carry that glob as well.

# Grid Diagrams

Grid diagrams let you display objects in a structured grid.

```
grid-rows: 5
style.fill: black

classes: {
  white square: {
    label: ""
    width: 120
    style: {
      fill: white
      stroke: cornflowerblue
      stroke-width: 10
    }
  }
  block: {
    style: {
      text-transform: uppercase
      font-color: white
      fill: darkcyan
      stroke: black
    }
  }
}

flow1.class: white square
flow2.class: white square
flow3.class: white square
flow4.class: white square
flow5.class: white square
flow6.class: white square
flow7.class: white square
flow8.class: white square
flow9.class: white square

dagger engine: {
  width: 800
  class: block
  style: {
    fill: beige
    stroke: darkcyan
    font-color: blue
    stroke-width: 8
  }
}

any docker compatible runtime: {
  width: 800
  class: block
  style: {
    fill: lightcyan
    stroke: darkcyan
    font-color: black
    stroke-width: 8
  }
  icon: https://icons.terrastruct.com/dev%2Fdocker.svg
}

any ci: {
  class: block
  style: {
    fill: gold
    stroke: maroon
    font-color: maroon
    stroke-width: 8
  }
}
windows.class: block
linux.class: block
macos.class: block
kubernetes.class: block

```

See more

Two keywords do all the magic:

- `grid-rows`
- `grid-columns`

Setting just `grid-rows`:

```
grid-rows: 3
Executive
Legislative
Judicial

```

Setting just `grid-columns`:

```
grid-columns: 3
Executive
Legislative
Judicial

```

Setting both `grid-rows` and `grid-columns`:

```
grid-rows: 2
grid-columns: 2
Executive
Legislative
Judicial

```

## Width and height

To create specific constructions, use `width` and/or `height`.

```
grid-rows: 2
Executive
Legislative
Judicial
The American Government.width: 400

```

Notice how objects are evenly distributed within each row.

## Cells expand to fill

When you define only one of row or column, objects will expand.

```
grid-rows: 3
Executive
Legislative
Judicial
The American Government.width: 400
Voters
Non-voters

```

Notice how `Voters` and `Non-voters` fill the space.

## Dominant direction

When you apply both row and column, the first appearance is the dominant direction. The
dominant direction is the order in which cells are filled.

For example:

```
grid-rows: 4
grid-columns: 2
# bunch of shapes

```

Since `grid-rows` is defined first, objects will fill rows before moving onto columns.

But if it were reversed:

```
grid-columns: 2
grid-rows: 4
# bunch of shapes

```

It would do the opposite.

These animations are also pure D2, so you can animate grid diagrams being built-up. Use
the `animate-interval` flag with this
code.
More on this later, in the composition section.

## Gap size

You can control the gap size of the grid with 3 keywords:

- `vertical-gap`
- `horizontal-gap`
- `grid-gap`

Setting `grid-gap` is equivalent to setting both `vertical-gap` and `horizontal-gap`.

`vertical-gap` and `horizontal-gap` can override `grid-gap`.

### Gap size 0

`grid-gap: 0` in particular can create some interesting constructions:

#### Like this map of Japan

> D2 source

#### Or a table of data

```
# Specified so that objects are written in row-dominant order
grid-rows: 2
grid-columns: 4
grid-gap: 0

classes: {
  header: {
    style.underline: true
  }
}

Element.class: header
Atomic Number.class: header
Atomic Mass.class: header
Melting Point.class: header

Hydrogen

"1.008"
"-259.16"

Carbon

"12.011"


Oxygen

"15.999"
"-218.79"

```

You may find it easier to just use Markdown tables though, especially if there are
duplicate cells.

```
savings: ||md
  | Month    | Savings | Expenses | Balance |
  | -------- | ------- | -------- | ------- |
  | January  | $250    | $150     | $100    |
  | February | $80     | $200     | -$120   |
  | March    | $420    | $180     | $240    |
||

```

| Month    | Savings | Expenses | Balance |
| -------- | ------- | -------- | ------- |
| January  | $250    | $150     | $100    |
| February | $80     | $200     | -$120   |
| March    | $420    | $180     | $240    |

### Gap size 0

## Connections

Connections for grids themselves work normally as you'd expect.

# Identity Native Proxy

> Source code here.

### Connections between grid cells

Connections between shapes inside a grid work a bit differently. Because a grid structure
imposes positioning outside what the layout engine controls, the layout engine is also
unable to make routes. Therefore, these connections are center-center straight segments,
i.e., no path-finding.

> Source code here.

> Source code here.

## Nesting

Currently you can nest grid diagrams within grid diagrams. Nesting other types is coming
soon.

```
grid-gap: 0
grid-columns: 1
header
body: "" {
  grid-gap: 0
  grid-columns: 2
  content
  sidebar
}
footer

```

## Aligning with invisible elements

A common technique to align grid elements to your liking is to pad the grid with invisible
elements.

Consider the following diagram.

```
grid-columns: 1
us-east-1: {
  grid-rows: 1
  a
  b
  c
  d
  e
}

us-west-1: {
  grid-rows: 1
  a
}

us-east-1.c -> us-west-1.a

```

It'd be nicer if it were centered. This can be achieved by adding 2 invisible elements.

```
classes: {
  invisible: {
    style.opacity: 0
    label: a
  }
}

grid-columns: 1
us-east-1: {
  grid-rows: 1
  a
  b
  c
  d
  e
}

us-west-1: {
  grid-rows: 1
  pad1.class: invisible
  pad2.class: invisible
  a
  # Move the label so it doesn't go through the connection
  label.near: bottom-center
}

us-east-1.c -> us-west-1.a

```

## Troubleshooting

### Why is there extra padding in one cell?

Elements in a grid column have the same width and elements in a grid row have the same
height.

So in this example, a small empty space in "Backend Node" is present.

```
classes: {
  kuber: {
    style: {
      fill: "white"
      stroke: "#aeb5bd"
      border-radius: 4
      stroke-dash: 3
    }
  }
  sys: {
    label: ""
    style: {
      fill: "#AFBFDF"
      stroke: "#aeb5bd"
    }
  }
  node: {
    grid-gap: 0
    style: {
      fill: "#ebf3e6"
      border-radius: 8
      stroke: "#aeb5bd"
    }
  }
  clust: {
    style: {
      fill: "#A7CC9E"
      stroke: "#aeb5bd"
    }
  }
  deploy: {
    grid-gap: 0
    style: {
      fill: "#ffe6d5"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
  nextpod: {
    icon: https://www.svgrepo.com/show/378440/nextjs-fill.svg
    style: {
      fill: "#ECECEC"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
  flaskpod: {
    icon: https://www.svgrepo.com/show/508915/flask.svg
    style: {
      fill: "#ECECEC"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
}

classes

Kubernetes: {
  grid-columns: 2
  system: {
    grid-columns: 1
    Backend Node: {
      grid-columns: 2
      ClusterIP\nService 1
      Deployment 1: {
        grid-rows: 3
        NEXT POD 1
        NEXT POD 2
        NEXT POD 3
      }
    }
    Frontend Node: {
      grid-columns: 2
      ClusterIP\nService 2
      Deployment 2: {
        grid-rows: 3
        FLASK POD 1
        FLASK POD 2
        FLASK POD 3
      }
    }
  }
}

kubernetes.class: kuber
kubernetes.system.class: sys

kubernetes.system.backend node.class: node
kubernetes.system.backend node.clusterip\nservice 1.class: clust
kubernetes.system.backend node.deployment 1.class: deploy
kubernetes.system.backend node.deployment 1.next pod*.class: nextpod

kubernetes.system.frontend node.class: node
kubernetes.system.frontend node.clusterip\nservice 2.class: clust
kubernetes.system.frontend node.deployment 2.class: deploy
kubernetes.system.frontend node.deployment 2.flask pod*.class: flaskpod

```

See more

It's due to the label of "Flask Pod" being slightly longer than "Next Pod". So the way we
fix that is to set `width` s to match.

```
classes: {
  kuber: {
    style: {
      fill: "white"
      stroke: "#aeb5bd"
      border-radius: 4
      stroke-dash: 3
    }
  }
  sys: {
    label: ""
    style: {
      fill: "#AFBFDF"
      stroke: "#aeb5bd"
    }
  }
  node: {
    grid-gap: 0
    style: {
      fill: "#ebf3e6"
      border-radius: 8
      stroke: "#aeb5bd"
    }
  }
  clust: {
    style: {
      fill: "#A7CC9E"
      stroke: "#aeb5bd"
    }
  }
  deploy: {
    grid-gap: 0
    style: {
      fill: "#ffe6d5"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
  nextpod: {
    width: 180
    icon: https://www.svgrepo.com/show/378440/nextjs-fill.svg
    style: {
      fill: "#ECECEC"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
  flaskpod: {
    width: 180
    icon: https://www.svgrepo.com/show/508915/flask.svg
    style: {
      fill: "#ECECEC"
      stroke: "#aeb5bd"
      # border-radius: 4
    }
  }
}

classes

Kubernetes: {
  grid-columns: 2
  system: {
    grid-columns: 1
    Backend Node: {
      grid-columns: 2
      ClusterIP\nService 1
      Deployment 1: {
        grid-rows: 3
        NEXT POD 1
        NEXT POD 2
        NEXT POD 3
      }
    }
    Frontend Node: {
      grid-columns: 2
      ClusterIP\nService 2
      Deployment 2: {
        grid-rows: 3
        FLASK POD 1
        FLASK POD 2
        FLASK POD 3
      }
    }
  }
}

kubernetes.class: kuber
kubernetes.system.class: sys

kubernetes.system.backend node.class: node
kubernetes.system.backend node.clusterip\nservice 1.class: clust
kubernetes.system.backend node.deployment 1.class: deploy
kubernetes.system.backend node.deployment 1.next pod*.class: nextpod

kubernetes.system.frontend node.class: node
kubernetes.system.frontend node.clusterip\nservice 2.class: clust
kubernetes.system.frontend node.deployment 2.class: deploy
kubernetes.system.frontend node.deployment 2.flask pod*.class: flaskpod

```

See more

# Hello World

```
x -> y: hello world

```

This declares a connection between two shapes, `x` and `y`, with the label, `hello world`.

# Contributing

Contributions are welcome! Please see the full doc on contributing on D2's Github:
https://github.com/terrastruct/d2/blob/master/docs/CONTRIBUTING.md.

## Help wanted

The above doc is for contributing to D2 core. However, D2's community often requests
things that the core team don't have the bandwidth to do (until D2 reaches 1.0, we're
aggressively prioritizing improvements that make D2 better for the majority of users).
Below is a list of features that could use your help. If you're interested in taking the
initiative, feel free to just start, or you can comment on a GitHub issue, discuss in
Discord, or email us (alex at terrastruct).

- ownCloud Infinite Scale Extension
- IntelliJ plugin
- Chocolatey package
- Docusaurus plugin
- Visual Studio Extension
- PrismJS highlighter

Want to add something to this wishlist? Just make an Issue on D2's GitHub!

# Icons

tip

We host a collection of icons commonly found in software architecture diagrams for free to
help you get started: https://icons.terrastruct.com.

Icons and images are an essential part of production-ready diagrams.

You can use any URL as value.

```
my network: {
  icon: https://icons.terrastruct.com/infra/019-network.svg
}

```

Icons on connections coming soon.

Using the D2 CLI locally? You can specify local images like `icon: ./my_cat.png`.

Icon placement is automatic. Considerations vary depending on layout engine, but things
like coexistence with a label and whether it's a container generally affect where the icon
is placed to not obstruct. Notice how the following diagram has container icons in the
top-left and non-container icons in the center.

```
vpc: VPC 1 10.1.0.0./16 {
  icon: https://icons.terrastruct.com/aws%2F_Group%20Icons%2FVirtual-private-cloud-VPC_light-bg.svg
  style: {
    stroke: green
    font-color: green
    fill: white
  }
  az: Availability Zone A {
    style: {
      stroke: blue
      font-color: blue
      stroke-dash: 3
      fill: white
    }
    firewall: Firewall Subnet A {
      icon: https://icons.terrastruct.com/aws%2FNetworking%20&%20Content%20Delivery%2FAmazon-Route-53_Hosted-Zone_light-bg.svg
      style: {
        stroke: purple
        font-color: purple
        fill: "#e1d5e7"
      }
      ec2: EC2 Instance {
        icon: https://icons.terrastruct.com/aws%2FCompute%2F_Instance%2FAmazon-EC2_C4-Instance_light-bg.svg
      }
    }
  }
}

```

Icons can be positioned with the `near` keyword introduced later.

## Add `shape: image` for standalone icon shapes

```
direction: right
server: {
  shape: image
  icon: https://icons.terrastruct.com/tech/022-server.svg
}

github: {
  shape: image
  icon: https://icons.terrastruct.com/dev/github.svg
}

server -> github

```


# Template

You make diagrams for external consulting clients. In order to appear professional, all
diagrams must be contained within a template that your designer has created that is
on-brand.

- `diagram.d2`

```
template: {
  ...@imports-wrapper-template
  synergy: {
    our firm -> yours: value add
  }
  stakeholders: {
    george.shape: person
    tim.shape: person
    tim.tooltip: is this web scale?
  }
}

```

- `wrapper-template.d2`

```
style: {
  fill: "#E3EDE6"
  fill-pattern: dots
  stroke: "#820758"
  stroke-width: 3
  border-radius: 2
  shadow: true
}
label: ""

```

This use case will be made much more powerful when D2 finishes glob ( `*`) support.

## Render of `diagram.d2`

is this web scale?is this web scale?

# Syntax

There are two ways to import. These two examples both have the same result:

> Result of running both types of imports below

In the next section, we'll see examples of common import use cases.

## Two types of imports

### 1\. Regular import

- `x.d2`

```
x: {
  shape: circle
}

```

- `y.d2`

```
a: @x.d2
a -> b

```

This is the equivalent of giving the entire file of `x` as a map that `a` sets as its
value.

### 2\. Spread import

- `x.d2`

```
x: {
  shape: circle
}

```

- `y.d2`

```
a: {
  ...@x.d2
}
a -> b

```

This tells D2 to take the contents of the file `x` and insert it into the map.

Spread imports only work within maps. Something like `a: ...@x.d2` is an invalid usage.

## Omit the extension

Above, we wrote the full file name for clarity, but the correct usage is to just specify
the file name without the suffix. If you run D2's autoformatter, it'll change

```
x: @x.d2

```

into

```
x: @x

```

D2 will not open files that don't have `.d2` extension, which means an import like
`@x.txt` won't work.

## Partial imports

You don't have to import the full file.

For example, if you have a file that defines all the people in your organization, and you
just want to show some relations between managers, you can import a specific object.

`donut-flowchart.d2`

```
...@people.management
joe -> donuts: loves
jan -> donuts: brings

```

`people.d2`

```
management: {
  joe: {
    shape: person
    label: Joe Donutlover
  }
  jan: {
    shape: person
    label: Jan Donutbaker
  }
}
# Notice how these do not appear in the rendered diagram
employees: {
  toby: {
    shape: person
    label: Toby Simonton
  }
}

```

Since `.` is used for targeting, if you want to import from a file with `.` in its name,
use string quotes.

`@"schema-v0.1.2"`

### Render of donut-flowchart.d2

## Imports are relative to that file

Not to the executing path.

Consider that your working directory is `/Users/You/dev`. Your D2 files:

- `/Users/you/dev/d2-stuff/x.d2`

```
y: @../y.d2

```

The above import will search directory `/Users/you/dev/` for `y.d2`, not `/Users/You`.

Unnecessary relative imports are removed by autoformat.

`@./x` will be autoformatted to `@x`.

# Overview

The following are examples of some popular use cases for imports. Fundamentally, D2
imports behave just like dependencies in other programming languages, so it is flexible
for doing much more than the ones shown here.

## Patterns

- Model-view
- Modular classes
- Nested composition

## Principles

Alongside the patterns that imports can be used for, splitting your diagram into multiple
files can also fulfill useful principles for organizations:

- **Compliance**. Some core piece of architecture needs to follow a spec exactly, and that
  file should not change while others around it can.
- **Domain-driven design**. Diagramming a large architecture? Get different domain experts to
  diagram their parts, and put them together in one diagram.
- **Code reviews**. While text-based diagramming makes reviewing changes possible, we've
  all been in situations where the diff shown makes the review appear more complex than it
  is. When your diagram is more modular, it's easier for everyone to know what's changed.
  Maybe the changes to `style.d2` don't need as much scrutiny as the `security.d2` file
  changes.
- **Reusability**. Your organization can define one `color-classes.d2`, and that can be
  imported and used across all diagrams for a consistent diagram style.
- **Don't repeat yourself**. When multiple files import an object, you only have to make
  changes to that one object to update it everywhere else.

## More Examples

Some more creative, practical examples of using imports that mix and match the above
patterns and principles.

- Version visualization
- Template

# Install

tip

There are more detailed install instructions for Mac, Windows, and Linux, using a variety of
methods, here. This page
is an abridged version.

## Install script

The recommended way to install is to run our install script, which will figure out the
best way to install based on your machine. E.g. if D2 is available through a package
manager installed, it will use that package manager.

```
# With --dry-run the install script will print the commands it will use
# to install without actually installing so you know what it's going to do.
curl -fsSL https://d2lang.com/install.sh | sh -s -- --dry-run
# If things look good, install for real.
curl -fsSL https://d2lang.com/install.sh | sh -s --

```

Follow the instructions, if any. Test your installation was successful by running `d2
version`.

If you want to uninstall:

```
curl -fsSL https://d2lang.com/install.sh | sh -s -- --uninstall

```

## Install from source

Alternatively, you can install from source:

```
go install oss.terrastruct.com/d2@latest

```

You can also download precompiled binaries specific to your OS on the Github releases
page:
https://github.com/terrastruct/d2/releases.

# Try it out

```
echo 'x -> y' > input.d2
d2 -w input.d2 out.svg

```

It should have spun up a local web browser that will automatically refresh when you change
`input.d2`. Modify `input.d2` as you go through this tour to follow along.

# Interactive

## Tooltips

Tooltips are text that appear on hover. They serve two purposes:

. Add secondary context.

- You want to add a description to an object. It's not so important that everyone
  needs it, but those who want extra information can access it.
  . Tidy.
- Your diagram is getting messy. Instead of adding more text, you can tuck some into
  tooltips.

```
x: {tooltip: Total abstinence is easier than perfect moderation}
y: {tooltip: Gee, I feel kind of LIGHT in the head now,\nknowing I can't make my satellite dish PAYMENTS!}
x -> y

```

Try it out, hover over `x` and `y`. Notice that they have an icon indicating that they
have a tooltip.

Total abstinence is easier than perfect moderationGee, I feel kind of LIGHT in the head now,
knowing I can't make my satellite dish PAYMENTS!Total abstinence is easier than perfect moderationGee, I feel kind of LIGHT in the head now,
knowing I can't make my satellite dish PAYMENTS!

When you export to a static format like PNG, D2 will

. Change all the icons to be numbered.
. Add an appendix, where each line corresponds to the number.

!d2 tooltip

caution

Tooltips are implemented with HTML
title tags,
which have basic support for text formatting. Markdown won't be rendered as expected in
tooltips.

## Links

Links are like tooltips, except you click to go to an external link.

When the link contains the `#` character as part of a URI fragment, e.g.,
`https://example.com/page#fragment`, remember that the fragment will be
treated as a comment if unquoted and unescaped.

```
x: I'm a Mac {
  link: https://apple.com
}
y: And I'm a PC {
  link: https://microsoft.com
}
x -> y: gazoontite {
  link: https://google.com
}

```

Try clicking on each.

# D2 Tour

**D2** is a diagram scripting language that turns text to diagrams. It stands for
**Declarative Diagramming**. Declarative, as in, you describe what you want diagrammed, it
generates the image.

For example, download the CLI, create a file named `input.d2`, copy paste the following,
run this command, and you get the image below.

```
d2 --theme=300 --dark-theme=200 -l elk --pad 0 ./input.d2

```

```
vars: {
  d2-config: {
    layout-engine: elk
    # Terminal theme code
    theme-id: 300
  }
}
network: {
  cell tower: {
    satellites: {
      shape: stored_data
      style.multiple: true
    }

    transmitter

    satellites -> transmitter: send
    satellites -> transmitter: send
    satellites -> transmitter: send
  }

  online portal: {
    ui: {shape: hexagon}
  }

  data processor: {
    storage: {
      shape: cylinder
      style.multiple: true
    }
  }

  cell tower.transmitter -> data processor.storage: phone logs
}

user: {
  shape: person
  width: 130
}

user -> network.cell tower: make call
user -> network.online portal.ui: access {
  style.stroke-dash: 3
}

api server -> network.online portal.ui: display
api server -> logs: persist
logs: {shape: page; style.multiple: true}

network.data processor -> api server

```

## Using the CLI watch mode

!D2 CLI

You can finish this tour in about 5-10 minutes, and at the end, there's a cheat sheet you
can download and refer to. If you want just the bare essentials, Getting Started takes
~2 mins.

The source code for D2 is hosted here:
https://github.com/terrastruct/d2.

The source code for these docs are here:
https://github.com/terrastruct/d2-docs.

For each D2 snippet, you can hover over it to open directly in the Playground and tinker.

There's some exceptions like snippets that use imports.

# Layers

A "Layer" represents "a layer of abstraction". Each Layer starts off as a blank
board, since you're representing different objects at every level of abstraction.

Try clicking on the objects.

```
explain: |md
  This is the top layer, highest level of abstraction.
| {
  near: top-center
}

Tik Tok's User Data: {
  link: layers.tiktok
}

layers: {
  tiktok: {
    explain: |md
      One layer deeper:

      Tik Tok's CEO explained that user data is stored in two places currently.
    | {
      near: top-center
    }
    Virginia data center <-> Hong Kong data center
    Virginia data center.link: layers.virginia
    Hong Kong data center.link: layers.hongkong

    layers: {
      virginia: {
        direction: right
        explain: |md
          Getting deeper into details:

          TikTok's CEO explains that Virginia data center has strict security measures.
        | {
          near: top-center
        }
        Oracle Databases: {
          shape: cylinder
          style.multiple: true
        }
        US residents -> Oracle Databases: access
        US residents: {
          shape: person
        }
        Third party auditors -> Oracle Databases: verify
      }
      hongkong: {
        direction: right
        explain: |md
          TikTok's CEO says data is actively being deleted and should be done by the end of the year.
        | {
          near: top-center
        }
        Legacy Databases: {
          shape: cylinder
          style.multiple: true
        }
      }
    }
  }
}

```


# Overview

D2 supports using a variety of different layout engines. The choice of layout engines can
significantly influence your overall diagram. Each layout also has varying degrees of
support for certain keywords. Though we try our best to keep things consistent, ultimately
we have the most control over our custom-built layout engine and are limited by what the
other layout engines support.

## Layout engines

- dagre (default): A fast, directed graph layout engine that produces
  layered/hierarchical layouts. Based on Graphviz's DOT algorithm.
- ELK: Also a directed graph layout engine. More mature than dagre, better
  maintained (part-time academic research team working on it), with recent releases.
- TALA: New layout engine designed specifically for software architecture
  diagrams.

You can choose whichever layout engine you like and works best for the diagram you're
making. Each one has its tradeoffs, visit the individual pages to learn more.

To see available layouts on your machine, you can run `d2 layout`. Each layout engine can
also have specific configurable flags, which you can find by running `d2 layout [engine]`,
e.g. `d2 layout dagre`.

To specify the layout used, you can either set the flag `--layout=dagre` or set it as an
environment variable, `$D2_LAYOUT=dagre`.

### Layout-specific functionality

Some keywords and functionality are only available for certain layout engines. We authored
and maintain TALA, so it is the only one we have full control over. We write shims for
Dagre and ELK, but some things are fundamental to the layout engines, and the only way to
support everything we want to do on those is to fork (which we may eventually do).

These are mentioned in other parts of the doc and aggregated here:

- `near` set to another object. `near` can be set to constants for all layout engines, but
  only TALA can use it to set to objects.
- `width` and `height` on containers. TALA will add this soon, but currently it is only in
  ELK. Note that these keywords work on non-containers in all layout engines.
- `top` and `left` to lock positions only work in TALA.
- Connections from ancestors to descendants (e.g. a container to its child) do not work in
  Dagre.

## Direction

Set `direction` to one of the following to influence an explicit direction your diagram
flows towards.

Options

- `up`
- `down`
- `right`
- `left`

```
direction: right
x -> y -> z: hello

```

```
direction: up
x -> y -> z: hello

```

### Directions per container (TALA only)

Directions can only be set at a global level for all the layout engines except TALA. This
is a limitation of their algorithms, which are hierarchical and only work in one
direction. We are investigating ways to hack it to work.

```
vars: {
  d2-config: {
    layout-engine: tala
  }
}
direction: down
a -> b -> c

b: {
  direction: right
  1 -> 2 -> 3
}

a: {
  direction: left
  foo -> bar
}

```

!directions in TALA

# Linking between boards

We've introduced `link` before as a way to jump to external resources. They can also be
used to create interactivity to jump to other boards. We'll call these "internal links".

Example of internal link:

```
how does the cat go?: {
  link: layers.cat
}

layers: {
  cat: {
    meoowww
  }
}

```

If your board name has a `.`, use quotes to target that board.
For example:

```
a.link: layers."2012.06"

layers: {
  "2012.06": {
    hello
  }
}

```

## Parent reference

The underscore `_` is used to refer to the parent scope, but when used in `link` values,
they refer not to parent containers, but to parent boards.

```
The shire

journey: {
  link: layers.rivendell
}

layers: {
  rivendell: {
    elves: {
      elrond -> frodo: gives advice
    }

    take me home sam.link: _
    go deeper: {
      link: layers.moria
    }

    layers: {
      moria: {
        dwarves

        take me home sam.link: _._
      }
    }
  }
}

```

## Backlinks

Notice how the navigation bar at the top is clickable. You can easily return to the root
or any ancestor page by clicking on the text.

# CLI manual

The following is a copy of the `man` (manual) for the CLI. It is identical to the output
you would get by installing the CLI and running `man d2`.

```
d2(1)               General Commands Manual          d2(1)

NAME
     d2 – compiles and renders d2 diagrams into svgs.

SYNOPSIS
     d2 [--watch false] [--theme 0] file.d2 [file.svg | file.png]
     d2 layout [name]
     d2 fmt file.d2 ...

DESCRIPTION
     d2 compiles and renders file.d2 to file.svg | file.png.

     It defaults to file.svg if no output path is passed.

     Pass - to have d2 read from stdin or write to stdout.

     Never use the presence of the output file to check for success.  Always
     use the exit status of d2.  This is because sometimes when errors occur
     while rendering, d2 still write out a partial render anyway to enable
     iteration on a broken diagram.

     See more docs, the source code and license at
     https://oss.terrastruct.com/d2.

     Hosted icons at https://icons.terrastruct.com.

     Playground runner at https://play.d2lang.com.

OPTIONS
     -w, --watch false
         Watch for changes to input and live reload. Use $PORT and
         $HOST to specify the listening address.

     -h, --host localhost
         Host listening address when used with watch.

     -p, --port 0
         Port listening address when used with watch.

     -t, --theme 0
         Set the diagram theme ID.

     --dark-theme -1
         The theme to use when the viewer's browser is in dark mode.
         When left unset --theme is used for both light and dark mode.
         Be aware that explicit styles set in D2 code will still be
         applied and this may produce unexpected results. We plan on
         resolving this by making style maps in D2 light/dark mode
         specific. See https://github.com/terrastruct/d2/issues/831.

     -s, --sketch false
         Renders the diagram to look like it was sketched by hand.

     --center flag
         Center the SVG in the containing viewbox, such as your
         browser screen.

     --scale -1  Scale the output. E.g., 0.5 to halve the default size.
         Default -1 means that SVG's will fit to screen and all others
         will use their default render size. Setting to 1 turns off
         SVG fitting to screen.

     --font-regular
         Path to .ttf file to use for the regular font. If none
         provided, Source Sans Pro Regular is used.

     --font-italic
         Path to .ttf file to use for the italic font. If none
         provided, Source Sans Pro Regular-Italic is used.

     --font-bold
         Path to .ttf file to use for the bold font. If none provided,
         Source Sans Pro Bold is used.

     --pad 100   Pixels padded around the rendered diagram.

     --animate-interval 0
         If given, multiple boards are packaged as 1 SVG which
         transitions through each board at the interval (in
         milliseconds). Can only be used with SVG exports.

     --browser true
         Browser executable that watch opens. Setting to 0 opens no
         browser.

     -l, --layout dagre
         Set the diagram layout engine to the passed string. For a
         list of available options, run layout.

     -b, --bundle true
         Bundle all assets and layers into the output svg.

     --force-appendix false
         An appendix for tooltips and links is added to PNG exports
         since they are not interactive. Setting this to true adds an
         appendix to SVG exports as well.

     --target    Target board to render. Pass an empty string to target root
         board. If target ends with '*', it will be rendered with all
         of its scenarios, steps, and layers. Otherwise, only the
         target board will be rendered. E.g. --target='' to render
         root board only or --target='layers.x.*' to render layer 'x'
         with all of its children.

     -d, --debug
         Print debug logs.

     --img-cache true
         In watch mode, images used in icons are cached for subsequent
         compilations. This should be disabled if images might change.

     --timeout 120
         The maximum number of seconds that D2 runs for before timing
         out and exiting. When rendering a large diagram, it is
         recommended to increase this value.

     -h, --help  Print usage information and exit.

     -v, --version
         Print version information and exit.

SUBCOMMANDS
     layout  Lists available layout engine options with short help.

     layout [name]
         Display long help for a particular layout engine, including
         its configuration options.

     themes  Lists available themes.

     fmt file.d2 ...
         Format all passed files.

SEE ALSO
     d2plugin-tala(1)

AUTHORS
     Terrastruct Inc.

```


# Model-view

A popular pattern defines your models once, and then reuses it across a number of
different views.

## `models.d2`

```
postgres: {
  shape: cylinder
  icon: https://icons.terrastruct.com/dev%2Fpostgresql.svg
  icon.near: bottom-center
}
it: IT Guy {
  shape: person
  style: {
    fill: maroon
  }
}
vpn: {
  style: {
    shadow: true
  }
  tooltip: IP is 192.2.2.1
}

```

## `access-view.d2`

```
...@models
it -> vpn -> postgres

```

IP is 192.2.2.1IP is 192.2.2.1

## `ssh-view.d2`

```
...@models
it -> postgres: ssh, bypassing VPN

```

IP is 192.2.2.1IP is 192.2.2.1

# Modular classes

This pattern mirrors web development, separating HTML and CSS.

## `classes.d2`

```
classes: {
  base: {
    style: {
      border-radius: 4
      shadow: true
    }
  }
  error: {
    style.fill: pink
    style.stroke: red
  }
  med: {
    width: 200
    height: 200
    style.font-size: 24
  }
  large: {
    width: 300
    height: 300
    style.font-size: 28
  }
  xlarge: {
    width: 400
    height: 400
    style.font-size: 32
  }
  person: {
    shape: person
    style.stroke-dash: 3
  }
}

```

## `main.d2`

```
...@classes
user.class: person
error.class: [base; error]
modal.class: [base; med]

user -> app.signup: click
app.signup -> error: invalid fields
app.signup -> modal: continue registration

```


# Nested composition

Imports make large compositions much more manageable.

Large diagrams like the ones that take you from high-level overview to low-level details
becomes possible to cleanly construct.

Rendering `overview.d2` gives us a nested diagram while each file is kept flat and
readable.

### `overview.d2`

```
serviceA -> serviceB
serviceB.link: layers.serviceB

layers: {
  serviceB: @serviceB
}

```

### `serviceB.d2`

```
aws vault: {
  key
  token
}
stripe: {
  customer id
}
aws vault.key -> data
aws vault.token -> data
stripe.customer id -> data
data.link: layers.data

layers: {
  data: @data
}

```

### `data.d2`

```
users: {
  shape: sql_table
  id: int
  token: string
  customer_id: string
}

# Continue nesting as needed!
# layers: {
#   ...
# }

```

## Render of `overview.d2`


# Obsidian plugin

D2 has an official plugin for Obsidian.

Currently it only supports rendering of D2 diagrams.

Your browser does not support the video tag.

**Github:** https://github.com/terrastruct/d2-obsidian

# Overrides

If you redeclare a shape, the new declaration is merged with the previous declaration.

```
visual studio code text editor
visual studio code text editor: visual_studio_code_text_editor
# Remember that shape keys are case insensitive
visual studio CODE text editor: VisualStudioCodeTextEditor
visual studio code TEXT editor: Visual Studio Code Text Editor
visual STUDIO code text editor

```

The latest explicit setting of the label takes priority.

Here's a more complex example of overrides involving containers:

```
aws_s3: AWS S3 California {
  Monitoring -> California
}
aws_s3: "AWS S3 San Francisco, California" {
  California.San Francisco
}

# Equal to:
# aws_s3: "AWS S3 San Francisco, California" {
#   Monitoring -> California
#   California.San Francisco
# }

```

## Null

You may override with the value `null` to delete the shape/connection/attribute.

```
one
two

one: null

```

When is this useful?

- Import a diagram from a colleague and remove the things you don't want.
- Multi-board compositions where you inherit all the objects from a
  board with some exceptions.
- Use globs to define connections between a batch of objects except one in
  particular you want to leave out.

### Nulling a connection

```
one -> two

(one -> two)[0]: null

```

### Nulling an attribute

```
one: {
  style: {
    fill: pink
    stroke: green
  }
}

one.style.stroke: null

```

### Implicit nulls

If you null a shape with connections, its connections are also nulled (since every
connection in D2 needs an endpoint).

```
one -> two

two: null

```

If you null a shape with descendents, those descendants are also nulled.

```
one: {
  two: {
    three
  }
}

one.two: null

```


# Positions

In general, positioning is controlled entirely by the layout engine. It's one of the
primary benefits of text-to-diagram that you don't have to manually define all the
positions of objects.

However, there are occasions where you want to have some control over positions.
Currently, there are two ways to do that.

## Near

D2 allows you to position items on set points around your diagram.

Possible values

`top-left`, `top-center`, `top-right`,

`center-left`, `center-right`,

`bottom-left`, `bottom-center`, `bottom-right`

Let's explore some use cases:

### Giving your diagram a title

```
title: |md
  # A winning strategy
| {near: top-center}

poll the people -> results
results -> unfavorable -> poll the people
results -> favorable -> will of the people

```

# A winning strategy

### Creating a legend

```
direction: right

x -> y: {
  style.stroke: green
}

y -> z: {
  style.stroke: red
}

legend: {
  near: bottom-center
  color1: foo {
    shape: text
    style.font-color: green
  }

  color2: bar {
    shape: text
    style.font-color: red
  }
}

```

### Longform description or explanation

```
explanation: |md
  # LLMs
  The Large Language Model (LLM) is a powerful AI\
    system that learns from vast amounts of text data.\
  By analyzing patterns and structures in language,\
  it gains an understanding of grammar, facts,\
  and even some reasoning abilities. As users input text,\
  the LLM predicts the most likely next words or phrases\
  to create coherent responses. The model\
  continuously fine-tunes its output, considering both the\
  user's input and its own vast knowledge base.\
  This cutting-edge technology enables LLM to generate human-like text,\
  making it a valuable tool for various applications.
| {
  near: center-left
}

ML Platform -> Pre-trained models
ML Platform -> Model registry
ML Platform -> Compiler
ML Platform -> Validation
ML Platform -> Auditing

Model registry -> Server.Batch Predictor
Server.Online Model Server

```

# LLMs

The Large Language Model (LLM) is a powerful AI

system that learns from vast amounts of text data.

By analyzing patterns and structures in language,

it gains an understanding of grammar, facts,

and even some reasoning abilities. As users input text,

the LLM predicts the most likely next words or phrases

to create coherent responses. The model

continuously fine-tunes its output, considering both the

user's input and its own vast knowledge base.

This cutting-edge technology enables LLM to generate human-like text,

making it a valuable tool for various applications.

## Label and icon positioning

The `near` can be nested to `label` and `icon` to specify their positions.

```
direction: right
x -> y

x: worker {
  label.near: top-center
  icon: https://icons.terrastruct.com/essentials%2F005-programmer.svg
  icon.near: outside-top-right
}

y: profits {
  label.near: bottom-right
  icon: https://icons.terrastruct.com/essentials%2Fprofits.svg
  icon.near: outside-top-left
}

```

Extra values

When positioning labels and icons, in addition to the values that `near` can take
elsewhere, an `outside-` prefix can be added to specify positioning outside the bounding
box of the shape.

`outside-top-left`, `outside-top-center`, `outside-top-right`,

`outside-left-center`, `outside-right-center`,

`outside-bottom-left`, `outside-bottom-center`, `outside-bottom-right`

Note that `outside-left-center` is a different order than `center-left`.

## Near objects

Works in TALA only. We are working on shims to make this possible in other layout engines.

You can also set `near` to the absolute ID of another shape to hint to the layout engine
that they should be in the vicinity of one another.

```
vars: {
  d2-config: {
    layout-engine: tala
  }
}
aws: {
  load_balancer -> api
  api -> db
}
gcloud: {
  auth -> db
}

gcloud -> aws

explanation: |md
  # Why do we use AWS?
  - It has more uptime than GCloud
  - We have free credits
| {
  near: aws
}

```

Notice how the text is positioned near the `aws` node and not the `gcloud` node.

!text near example

## Top and left

On the TALA engine, you can also directly set the `top` and `left` values for objects, and
the layout engine will only move other objects around it.

For more on this, see page 17 of the [TALA user manual](https://github.com/terrastruct/TALA/blob/master/TALA_User_Manual.pdf).

# Scenarios

A "Scenario" represents a different view of the base Layer.

Each Scenario inherits from its base Layer. Any new objects are added onto all objects in
the base Layer, and you can reference any objects from the base Layer to update them.

Notice that in the below Scenario, we simply turn some objects opacity lower, and define a
new connection to show an alternate view of the deployment diagram.

```
direction: right

title: {
  label: Normal deployment
  near: bottom-center
  shape: text
  style.font-size: 40
  style.underline: true
}

local: {
  code: {
    icon: https://icons.terrastruct.com/dev/go.svg
  }
}
local.code -> github.dev: commit

github: {
  icon: https://icons.terrastruct.com/dev/github.svg
  dev
  master: {
    workflows
  }

  dev -> master.workflows: merge trigger
}

github.master.workflows -> aws.builders: upload and run

aws: {
  builders -> s3: upload binaries
  ec2 <- s3: pull binaries

  builders: {
    icon: https://icons.terrastruct.com/aws/Developer%20Tools/AWS-CodeBuild_light-bg.svg
  }
  s3: {
    icon: https://icons.terrastruct.com/aws/Storage/Amazon-S3-Glacier_light-bg.svg
  }
  ec2: {
    icon: https://icons.terrastruct.com/aws/_Group%20Icons/EC2-instance-container_light-bg.svg
  }
}

local.code -> aws.ec2: {
  style.opacity: 0.0
}

scenarios: {
  hotfix: {
    title.label: Hotfix deployment
    (local.code -> github.dev)[0].style: {
      stroke: "#ca052b"
      opacity: 0.1
    }

    github: {
      dev: {
        style.opacity: 0.1
      }
      master: {
        workflows: {
          style.opacity: 0.1
        }
        style.opacity: 0.1
      }

      (dev -> master.workflows)[0].style.opacity: 0.1
      style.opacity: 0.1
      style.fill: "#ca052b"
    }

    (github.master.workflows -> aws.builders)[0].style.opacity: 0.1

    (local.code -> aws.ec2)[0]: {
      style.opacity: 1
      style.stroke-dash: 5
      style.stroke: "#167c3c"
    }
  }
}

```


# Sequence Diagrams

Sequence diagrams are created by setting `shape: sequence_diagram` on an object.

```
shape: sequence_diagram
alice -> bob: What does it mean\nto be well-adjusted?
bob -> alice: The ability to play bridge or\ngolf as if they were games.

```

## Rules

Unlike other tools, there is no special syntax to learn for sequence diagrams. The rules
are also almost exactly the same as everywhere else in D2, with two notable differences.

### Scoping

Children of sequence diagrams share the same scope throughout the sequence diagram.

For example:

```
Office chatter: {
  shape: sequence_diagram
  alice: Alice
  bob: Bobby
  awkward small talk: {
    alice -> bob: uhm, hi
    bob -> alice: oh, hello
    icebreaker attempt: {
      alice -> bob: what did you have for lunch?
    }
    unfortunate outcome: {
      bob -> alice: that's personal
    }
  }
}

```

Outside of a sequence diagram, there would be multiple instances of `alice` and `bob`,
since they have different container scopes. But when nested under `shape:
sequence_diagram`, they refer to the same `alice` and `bob`.

### Ordering

Elsewhere in D2, there is no notion of order. If you define a connection after another,
there is no guarantee is will visually appear after. However, in sequence diagrams, order
matters. The order in which you define everything is the order they will appear.

This includes actors. You don't have to explicitly define actors (except when they first
appear in a group), but if you want to define a specific order, you should.

```
shape: sequence_diagram
# Remember that semicolons allow multiple objects to be defined in one line
# Actors will appear from left-to-right as a, b, c, d...
a; b; c; d
# ... even if the connections are in a different order
c -> d
d -> a
b -> d

```

An actor in D2 is also known elsewhere as "participant".

## Features

### Sequence diagrams are D2 objects

Like every other object in D2, they can be contained, connected, relabeled, re-styled, and
treated like any other object.

```
direction: right
Before and after becoming friends: {
  2007: Office chatter in 2007 {
    shape: sequence_diagram
    alice: Alice
    bob: Bobby
    awkward small talk: {
      alice -> bob: uhm, hi
      bob -> alice: oh, hello
      icebreaker attempt: {
        alice -> bob: what did you have for lunch?
      }
      unfortunate outcome: {
        bob -> alice: that's personal
      }
    }
  }

  2012: Office chatter in 2012 {
    shape: sequence_diagram
    alice: Alice
    bob: Bobby
    alice -> bob: Want to play with ChatGPT?
    bob -> alice: Yes!
    bob -> alice.play: Write a play...
    alice.play -> bob.play: about 2 friends...
    bob.play -> alice.play: who find love...
    alice.play -> bob.play: in a sequence diagram
  }

  2007 -> 2012: Five\nyears\nlater
}

```

### Spans

Spans convey a beginning and end to an interaction within a sequence diagram.

A span in D2 is also known elsewhere as a "lifespan", "activation box", and "activation bar".

You can specify a span by connecting a nested object on an actor.

```
shape: sequence_diagram
alice.t1 -> bob
alice.t2 -> bob.a
alice.t2.a -> bob.a
alice.t2.a <- bob.a
alice.t2 <- bob.a

```

### Groups

Groups help you label a subset of the sequence diagram.

A group in D2 is also known elsewhere as a "fragment", "edge group", and "frame".

We saw an example of this in an earlier example when explaining scoping rules. More
formally, a group is a container within a `sequence_diagram` shape which is not connected
to anything but has connections or objects inside.

```
shape: sequence_diagram
# Predefine actors
alice
bob
shower thoughts: {
  alice -> bob: A physicist is an atom's way of knowing about atoms.
  alice -> bob: Today is the first day of the rest of your life.
}
life advice: {
  bob -> alice: If all else fails, lower your standards.
}

```

caution

Due to the unique scoping rules in sequence diagrams, when you are within a group, the
objects you reference in connections must exist at the top-level. Notice in the above
example that `alice` and `bob` are explicitly declared before group declarations.

### Notes

Notes are declared by defining a nested object on an actor with no connections going to
it.

```
shape: sequence_diagram
alice -> bob
bob."In the eyes of my dog, I'm a man."
# Notes can go into groups, too
important insight: {
  bob."Cold hands, no gloves."
}
bob -> alice: Chocolate chip.

```

### Self-messages

Self-referential messages can be declared from an actor to the themselves.

```
shape: sequence_diagram
son -> father: Can I borrow your car?
friend -> father: Never lend your car to anyone to whom you have given birth.
father -> father: internal debate ensues

```

### Customization

You can style shapes and connections like any other. Here we make some messages dashed and
set the shape on an actor.

```
shape: sequence_diagram
scorer: {shape: person}
scorer.t -> itemResponse.t: getItem()
scorer.t <- itemResponse.t: item {
  style.stroke-dash: 5
}

scorer.t -> item.t1: getRubric()
scorer.t <- item.t1: rubric {
  style.stroke-dash: 5
}

scorer.t -> essayRubric.t: applyTo(essayResp)
itemResponse -> essayRubric.t.c
essayRubric.t.c -> concept.t: match(essayResponse)
scorer <- essayRubric.t: score {
  style.stroke-dash: 5
}

scorer.t -> itemOutcome.t1: new
scorer.t -> item.t2: getNormalMinimum()
scorer.t -> item.t3: getNormalMaximum()

scorer.t -> itemOutcome.t2: setScore(score)
scorer.t -> itemOutcome.t3: setFeedback(missingConcepts)

```

Lifeline edges (those lines going from top-down) inherit the actor's `stroke` and
`stroke-dash` styles.

```
shape: sequence_diagram
alice -> bob: What does it mean\nto be well-adjusted?
bob -> alice: The ability to play bridge or\ngolf as if they were games.

alice.style: {
  stroke: red
  stroke-dash: 0
}

```

## Glossary

!sequence diagram glossary

# Shapes

## Basics

You can declare shapes like so:

```
imAShape
im_a_shape
im a shape
i'm a shape
# notice that one-hyphen is not a connection
# whereas, `a--shape` would be a connection
a-shape

```

You can also use semicolons to define multiple shapes on the same line:

```
SQLite; Cassandra

```

By default, a shape's label is the same as the shape's key. But if you want it to be different, assign a new label like so:

```
pg: PostgreSQL

```

By default, a shape's type is `rectangle`. To specify otherwise, provide the field `shape`:

```
Cloud: my cloud
Cloud.shape: cloud

```

## Example

```
pg: PostgreSQL
Cloud: my cloud
Cloud.shape: cloud
SQLite
Cassandra

```

Keys are case-insensitive, so `postgresql` and `postgreSQL` will reference the same shape.

Shape catalog

There are other values that `shape` can take, but they're special types that are covered
in the next section.

# Sketch (Hand-drawn)

D2 can render diagrams to give the aesthetic of a hand-drawn sketch.

- In Terrastruct, you can enable this with hand-drawn mode.
- If you're using the CLI, this is enabled by the `--sketch` flag.
- https://play.d2lang.com
  supports this as well.

See our blog post on sketch mode

# D2 app for Slack

!D2 + Slack

### The fastest way to explain what you mean mid-conversation

Ever have like 6 rounds of back-and-forth to explain what you mean when it's a pretty
simple model you're explaining? The D2 app for Slack lets you create beautiful diagrams on the fly with
a `/d2` command.

Slack requires configurations that can only be set on a third-party platform. So **you'll**
**need to create a Terrastruct account (paid) to use the D2 app for Slack**. Terrastruct does not
collect any data from Slack.

Add to Slack

## Communicating ideas

Your browser does not support the video tag.

## Put large diagrams in the conversation for discussion

Your browser does not support the video tag.

## Additional information

- Privacy policy
- Security policy

# SQL Tables

## Basics

You can easily diagram entity-relationship diagrams (ERDs) in D2 by using the `sql_table` shape. Here's a minimal example:

```
my_table: {
  shape: sql_table
  # This is defined using the shorthand syntax for labels discussed in the containers section.
  # But here it's for the type of a constraint.
  # The id field becomes a map that looks like {type: int; constraint: primary_key}
  id: int {constraint: primary_key}
  last_updated: timestamp with time zone
}

```

Each key of a SQL Table shape defines a row. The primary value (the thing after the colon)
of each row defines its type.

The constraint value of each row defines its SQL constraint. D2 will recognize and
shorten:

| constraint  | short |
| ----------- | ----- |
| primary_key | PK    |
| foreign_key | FK    |
| unique      | UNQ   |

But you can set any constraint you'd like. It just won't be shortened if unrecognized.

You can also specify multiple constraints with an array.

```
x: int { constraint: [primary_key; unique] }

```

## Foreign Keys

Here's an example of how you'd define a foreign key connection between two tables:

```
objects: {
  shape: sql_table
  id: int {constraint: primary_key}
  disk: int {constraint: foreign_key}

  json: jsonb {constraint: unique}
  last_updated: timestamp with time zone
}

disks: {
  shape: sql_table
  id: int {constraint: primary_key}
}

objects.disk -> disks.id

```

When rendered with the TALA layout engine or ELK layout engine,
connections point to the exact row.

## Example

Like all other shapes, you can nest `sql_tables` into containers and define edges
to them from other shapes. Here's an example:

```
cloud: {
  disks: {
    shape: sql_table
    id: int {constraint: primary_key}
  }
  blocks: {
    shape: sql_table
    id: int {constraint: primary_key}
    disk: int {constraint: foreign_key}
    blob: blob
  }
  blocks.disk -> disks.id

  AWS S3 Vancouver -> disks
}

```


# Steps

A "Step" represents a step in a sequence of events.

Each Step inherits from its the Step before it. The first step inherits from its parent,
whether that's a Scenario or Layer.

Notice how in Step 3 for example, the object "Approach road" exists even though it's not
defined, because it was inherited from Step 2, which inherited it from Step 1.

```
Chicken's plan: {
  style.font-size: 35
  near: top-center
  shape: text
}

steps: {
  1: {
    Approach road
  }
  2: {
    Approach road -> Cross road
  }
  3: {
    Cross road -> Make you wonder why
  }
}

```


# Strings

## Unquoted strings

You may have noticed by now that the examples thus far have not used any quotes. It's our
goal for D2 to be easy to use and quotes tend to get in the way of that.

Quotes add friction in several ways. First, you have to close opened quotes. Second, you
have to remember whether to use single or double quotes. Lastly, they add syntactical
noise.

That means for the most part, don't worry about quotes!

Leading and trailing whitespace is trimmed.

```
   Office Bulb   :     Philips
            Switch   ->   Office Bulb

```

Unquoted strings may not contain certain characters that are used elsewhere in the
language. The syntax highlighting will make it clear if you're using a forbidden symbol.

## Quoted strings

If you need to use such symbols, you can use single or double quoted strings:

```
'$$$' -> "###"

```

You would use double quotes if your text contains single quotes, and vice-versa. If it includes both, use double quotes and simply `\` escape as you would in other languages.

## Autoformat

If your block string's indent is not sufficient, the autoformatter will correct it.

```
parent: {
example_code: |go
  package fs

  type FS interface {
    Open(name string) (File, error)
  }

  type File interface {
    Stat() (FileInfo, error)
    Read([]byte) (int, error)
    Close() error
  }

  var (
    ErrInvalid    = errInvalid()    // "invalid argument"
    ErrPermission = errPermission() // "permission denied"
    ErrExist      = errExist()      // "file already exists"
    ErrNotExist   = errNotExist()   // "file does not exist"
    ErrClosed     = errClosed()     // "file already closed"
  )
|
}

```

Will become:

```
parent: {
  example_code: |go
    package fs

    type FS interface {
      Open(name string) (File, error)
    }

    type File interface {
      Stat() (FileInfo, error)
      Read([]byte) (int, error)
      Close() error
    }

    var (
      ErrInvalid    = errInvalid()    // "invalid argument"
      ErrPermission = errPermission() // "permission denied"
      ErrExist      = errExist()      // "file already exists"
      ErrNotExist   = errNotExist()   // "file does not exist"
      ErrClosed     = errClosed()     // "file already closed"
    )
  |
}

```

Autoformat will check if there are any non empty lines with insufficient indent. If so, all
lines will be prepended with the correct amount of indent (while respecting existing
common indent). This way you don't have to fuss with your editor to get the indent right.
Just paste any code on the first column after starting a block string and autoformat will
correct it for you.

You can use tabs to indent block strings after indenting to the base block string indent
with two spaces. Any tabs in the base block string indent will be automatically converted
to two spaces.

# D2 Studio

D2 Studio is to D2 as IntelliJ is to Java (or XCode is to Swift, or GoLand is to Go, etc).

D2 is a programming language, and you can write and run it anywhere -- the end result is the same. But the medium changes the experience. Writing D2 on a code editor beats writing it on Apple Notes. D2 Studio is the optimized, feature-packed IDE for D2, turning it into a professional, collaborative diagramming tool for the whole team.

D2 Studio is one of the ways the company behind D2 monetizes. It's free to try and
evaluate, and like other IDEs you might try for your code, you can easily onboard by
importing existing D2 code and offboard the same way. No lock-in.

**Try it out**

## Benefits

- **Team assets**: Upload, name, and reuse your company's images and icons across
  diagrams.
- **Brand**: Company color palette, watermarks, team logos on presentations.
- **Bidirectional edits**: Use the GUI where it makes sense, like picking colors or sliding opacity values. Changes are synced to D2 automatically.
- **TALA in the cloud**: TALA runs on our powerful machines, racing multiple iterations to quickly get the best automatic layout on each compile.
- **Diagram management**: Bookmark, set permissions/passwords, and manage all your diagrams, individually and with your team.
- **Present**: Interactive web presentations that let viewers click through, toggle tooltips, and use arrow keys.
- **Multi-layered interface**: Unified interface for all layers, scenarios, and steps.

## Example

Here's an example of a multi-layer D2 diagram presented in D2 studio's web interface:

![D2 Studio screenshot](https://app.terrastruct.com/diagrams/2034296181)

# Styles

If you'd like to customize the style of a shape, the following reserved keywords can be
set under the `style` field.

Below is a catalog of all valid styles, applied individually to this baseline diagram.

note

The following SVGs are rendered with `direction: right`, but omitted from the shown scripts for
brevity.

## Style keywords

- opacity
- stroke
- fill (shape only)
- fill-pattern (shape only)
- stroke-width
- stroke-dash
- border-radius
- shadow (shape only)
- 3D (rectangle/square only)
- multiple (shape only)
- double-border (rectangles and ovals)
- font
- font-size
- font-color
- animated (connection only)
- bold, italic, underline
- text-transform
- root

## Opacity

Float between `0` and `1`.

```
direction: right
x -> y: hi {
  style: {
    opacity: 0.4
  }
}
x.style.opacity: 0
y.style.opacity: 0.7

```

## Stroke

CSS color name, hex code, or a subset of CSS gradient strings.

```
direction: right
x -> y: hi {
  style: {
    # All CSS color names are valid
    stroke: deepskyblue
  }
}
# We need quotes for hex otherwise it gets interpreted as comment
x.style.stroke: "#f4a261"

```

For `sql_table` s and `class` es, `stroke` is applied as `fill` to the body (since `fill` is
already used to control header's `fill`).

## Fill

CSS color name, hex code, or a subset of CSS gradient strings.

```
direction: right
x -> y: hi
y -> z
x.style.fill: "#f4a261"
y.style.fill: honeydew
z.style.fill: "linear-gradient(#f69d3c, #3f87a6)"

```

For `sql_table` s and `class` es, `fill` is applied to the header.

Want transparent?

```
x: {
  y
  y.style.fill: transparent
}
x.style.fill: PapayaWhip

```

## Fill Pattern

Available patterns:

- `dots`
- `lines`
- `grain`
- `none` (for cancelling out ones set by certain themes)

```
direction: right
style.fill-pattern: dots
x -> y: hi
x.style.fill-pattern: lines
y.style.fill-pattern: grain

```

## Stroke Width

Integer between `1` and `15`.

```
direction: right
x -> y: hi {
  style: {
    stroke-width: 8
  }
}
x.style.stroke-width: 1

```

## Stroke Dash

Integer between `0` and `10`.

```
direction: right
x -> y: hi {
  style: {
    stroke-dash: 3
  }
}
x.style.stroke-dash: 5

```

## Border Radius

Integer between `0` and `20`.

```
direction: right
x -> y: hi
x.style.border-radius: 3
y.style.border-radius: 8

```

`border-radius` works on connections too, which controls how rounded the corners are. This
only applies to layout engines that use corners (e.g. ELK), and of course, only has effect
on connections whose routes have corners.

Specifying a very high value creates a "pill" effect.

```
tylenol.style.border-radius: 999

```

## Shadow

`true` or `false`.

```
direction: right
x -> y: hi
x.style.shadow: true

```

## 3D

`true` or `false`.

```
direction: right
x -> y: hi
x.style.3d: true

```

## Multiple

`true` or `false`.

```
direction: right
x -> y: hi
x.style.multiple: true

```

## Double Border

`true` or `false`.

```
direction: right
x -> y: hi
x.style.double-border: true
y.shape: circle
y.style.double-border: true

```

## Font

Currently the only option is to specify `mono`. More coming soon.

```
direction: right
x -> y: hi {
  style: {
    font: mono
  }
}
x.style.font: mono
y.style.font: mono

```

## Font Size

Integer between `8` and `100`.

```
direction: right
x -> y: hi {
  style: {
    font-size: 28
  }
}
x.style.font-size: 8
y.style.font-size: 55

```

## Font Color

CSS color name, hex code, or a subset of CSS gradient strings.

```
direction: right
x -> y: hi {
  style: {
    font-color: red
  }
}
x.style.font-color: "#f4a261"

```

For `sql_table` s and `class` es, `font-color` is applied to the header text only (theme
controls other colors in the body).

## Animated

`true` or `false`.

```
direction: right
x -> y: hi {
  style.animated: true
}
x.style.animated: true

```

## Bold, italic, underline

`true` or `false`.

```
direction: right
x -> y: hi {
  style: {
    bold: true
  }
}
x.style.underline: true
y.style.italic: true
# By default, shape labels are bold. Bold has precedence over italic, so unbold to see
# italic style
y.style.bold: false

```

## Text transform

`text-transform` changes the casing of labels.

- `uppercase`
- `lowercase`
- `title`
- `none` (used for negating caps lock that special themes may apply)

```
direction: right
TOM -> jerry: hi {
  style: {
    text-transform: capitalize
  }
}
TOM.style.text-transform: lowercase
jerry.style.text-transform: uppercase

```

## Root

Some styles are applicable at the root level. For example, to set a diagram background
color, use `style.fill`.

Currently the set of supported keywords are:

- `fill`: diagram background color
- `fill-pattern`: background fill pattern
- `stroke`: frame around the diagram
- `stroke-width`
- `stroke-dash`
- `double-border`: two frames, which is a popular framing method

```
direction: right
x -> y: hi
style: {
  fill: LightBlue
  stroke: FireBrick
  stroke-width: 2
}

```

All diagrams in this documentation are rendered with `pad=0`. If you're using `stroke` to
create a frame for your diagram, you'll likely want to add some padding.

# TALA

Proprietary layout engine developed by Terrastruct, designed specifically for software
architecture diagrams.

TALA is a separate install from D2, to keep a clean cut between 100% free and
open-source D2, and proprietary, closed-source TALA. You can download it here:
https://github.com/terrastruct/tala.

## Reference

https://terrastruct.com/tala/

For the most up-to-date information, please see the [official TALA manual](https://github.com/terrastruct/TALA/blob/master/TALA_User_Manual.pdf).

## Pros

- As a general orthogonal layout engine, TALA is not constrained to one type like
  hierarchies or trees or radial. For fundamentally non-hierarchical layouts, TALA can
  produce diagrams like a human would on a whiteboard.
- `top` and `left` can be used to lock positions.
- Considers and prefers symmetry.
- First-class consideration for containers.
- `direction` can be set per-container.
- `near` can be specified to be to another shape.
- `sql_table` connections point to exact row.
- Dynamic label positioning to avoid obstructions.
- Connections for grid cells use TALA's routing engine instead of always straight lines.

## Cons

- Not free.
- Relatively new. ~2 years in production compared to the 10+ years that alternative layout
  engines have.
- More random than other layout engines. A small change to a label can cascade into an
  entirely different layout.

## Gallery

## Using a different seed

TALA's core algorithms use randomness in its initial placements and iterations. If you are
not satisfied with a layout, you can produce different ones by specifying the seed with
`--tala-seeds` flag. For example, these are the same diagrams as the above, with
`--tala-seeds=2`.

# Text

## Standalone text is Markdown

```
explanation: |md
  # I can do headers
  - lists
  - lists

  And other normal markdown stuff
|

```

# I can do headers

- lists
- lists

And other normal markdown stuff

## Most languages are supported

D2 most likely supports any language you want to use, including non-Latin ones like
Chinese, Japanese, Korean, even emojis!

床前明月光，

疑是地上霜。

举头望明月，

低头思故乡。

## LaTeX

You can use `latex` or `tex` to specify a LaTeX language block.

```
plankton -> formula: will steal
formula: {
  equation: |latex
    \lim_{h \rightarrow 0 } \frac{f(x+h)-f(x)}{h}
  |
}

```

A few things to note about LaTeX blocks:

- LaTeX blocks do not respect `font-size` styling. Instead, you must style these inside
  the Latex script itself with commands:
  - `\tiny{ }`
  - `\small{ }`
  - `\normal{ }`
  - `\large{ }`
  - `\huge{ }`
- Under the hood, this is using MathJax. It is not full LaTeX
  (full LaTeX includes a document layout engine). D2's LaTeX blocks are meant to display
  mathematical notation, but not support the format of existing LaTeX documents. See
  here for a list of all
  supported commands.

caution

D2 runs on the latest version of MathJax, which has a lot of nice things but unfortunately
does not have linebreaks. You can kind of get around this with the `displaylines` command.
See below.

note

Currently cannot be applied to labels, which is why the above example nests an object.
This is coming soon.

## Code

Change `md` to a programming language for code blocks

```
explanation: |go
  awsSession := From(c.Request.Context())
  client := s3.New(awsSession)

  ctx, cancelFn := context.WithTimeout(c.Request.Context(), AWS_TIMEOUT)
  defer cancelFn()
|

```

## Advanced: Non-Markdown text

In some cases, you may want non-Markdown text. Maybe you just don't like Markdown, or the
GitHub-styling of Markdown that D2 uses, or you want to quickly change a shape to text.
Just set `shape: text`.

```
title: A winning strategy {
  shape: text
  near: top-center
  style: {
    font-size: 55
    italic: true
  }
}

poll the people -> results
results -> unfavorable -> poll the people
results -> favorable -> will of the people

```

## Advanced: Block strings

What if you're writing Typescript where the pipe symbol `|` is commonly used? Just add
another pipe, `||`.

```
my_code: ||ts
  declare function getSmallPet(): Fish | Bird;
||

```

Actually, Typescript uses `||` too, so that won't work. Let's keep going.

```
my_code: |||ts
  declare function getSmallPet(): Fish | Bird;
  const works = (a > 1) || (b < 2)
|||

```

There's probably some language or scenario where the triple pipe is used too. D2 actually
allows you to use any special symbols (not alphanumeric, space, or `_`) after the first pipe:

```
# Much cleaner!
my_code: |`ts
  declare function getSmallPet(): Fish | Bird;
  const works = (a > 1) || (b < 2)
`|

```

## Advanced: LaTeX plugins

D2 includes the following LaTeX plugins:

```
grid-columns: 3
grid-gap: 100

*.style.fill: transparent
*.style.stroke: black

amscd plugin: {
  ex: |tex
    \begin{CD} B @>{\text{very long label}}>> C S^{{\mathcal{W}}_\Lambda}\otimes T @>j>> T\\ @VVV V \end{CD}
  |
}

braket plugin: {
  ex: |tex
    \bra{a}\ket{b}
  |
}

cancel plugin: {
  ex: |tex
    \cancel{Culture + 5}
  |
}

color plugin: {
  ex: |tex
    \textcolor{red}{y} = \textcolor{green}{\sin} x
  |
}

gensymb plugin: {
  ex: |tex
    \lambda = 10.6\,\micro\mathrm{m}
  |
}

mhchem plugin: {
  ex: |tex
    \ce{SO4^2- + Ba^2+ -> BaSO4 v}
  |
}

physics plugin: {
  ex: |tex
    \var{F[g(x)]}
    \dd(\cos\theta)
  |
}

multilines: {
  ex: |tex
    \displaylines{x = a + b \\ y = b + c}
    \sum_{k=1}^{n} h_{k} \int_{0}^{1} \bigl(\partial_{k} f(x_{k-1}+t h_{k} e_{k}) -\partial_{k} f(a)\bigr) \,dt
  |
}

asm: {
  ex: |latex
    \min_{ \mathclap{\substack{ x \in \mathbb{R}^n \ x \geq 0 \ Ax \leq b }}} c^T x
  |
}

```


# Themes

D2 comes with many themes that make your diagram look professional and ready to insert
into blogs and wikis.

!D2 theme choices!mixed berry blue theme!vanilla nitro cola theme

### They apply to special shapes like tables too

# Rendered with theme "Grape soda"

# Rendered with theme "Vanilla nitro cola"

## Setting theme on the CLI

To specify the theme used, you can set the flag `-t, --theme`.

```
d2 -t 101 input.d2

```

You can also use an environment variable.

```
D2_THEME=101 d2 input.d2

```

To see which themes are available, run

```
d2 themes

```

## Dark theme

Dark themes are not set by default, so your diagram will look the same regardless of
whether the user's system preferences are light or dark.

All diagrams in these docs have a dark theme. Try toggling your system preference between
light and dark and see how it changes.

If you'd like your diagram to adapt and switch to a dark theme when the user's system
preference is dark, you can do so by specifying the following flag.

```
d2 --dark-theme 200 input.d2

```

Like regular themes, this can also be set with an environment variable.

```
D2_DARK_THEME=200 d2 input.d2

```

The themes are catalogued separately into light and dark, but there's nothing stopping you
from passing a dark theme ID to `theme` for your diagram to always be dark (or vice versa,
to give a surprise to dark mode users).

An example of a dark theme (this one's an image not an SVG, so it won't change according
to your system preference).

!dark theme

## Special themes

Certain, special themes do more than just color.

For example, when you apply the `Terminal` theme, the following attributes are set as
default:

- Caps lock on all labels
- No border radius
- Monospaced font
- `fill-pattern` set to `dots` for all containers
- Most outer container has `double-border` set to `true`

Source code for the above diagram (rendered with ELK) is as follows. Notice that many of
the properties apparent in the diagram do not appear in the source, such as the casing of
the labels, because the special theme uses different defaults.

```
vars: {
  d2-config: {
    layout-engine: elk
    # Terminal theme code
    theme-id: 300
  }
}
network: {
  cell tower: {
    satellites: {
      shape: stored_data
      style.multiple: true
    }

    transmitter

    satellites -> transmitter: send
    satellites -> transmitter: send
    satellites -> transmitter: send
  }

  online portal: {
    ui: {shape: hexagon}
  }

  data processor: {
    storage: {
      shape: cylinder
      style.multiple: true
    }
  }

  cell tower.transmitter -> data processor.storage: phone logs
}

user: {
  shape: person
  width: 130
}

user -> network.cell tower: make call
user -> network.online portal.ui: access {
  style.stroke-dash: 3
}

api server -> network.online portal.ui: display
api server -> logs: persist
logs: {shape: page; style.multiple: true}

network.data processor -> api server

```

## Customizing themes

You can override theme values to customize existing themes or replace them entirely with
your own theme.

This is controlled by two configuration variables:

- `theme-overrides`: replaces color codes for theme
- `dark-theme-overrides`: replaces color codes for dark theme

Adding this snippet to the above code results in the following diagram.

```
vars: {
  d2-config: {
    theme-overrides: {
      B1: "#2E7D32"
      B2: "#66BB6A"
      B3: "#A5D6A7"
      B4: "#C5E1A5"
      B5: "#E6EE9C"
      B6: "#FFF59D"

      AA2: "#0D47A1"
      AA4: "#42A5F5"
      AA5: "#90CAF9"

      AB4: "#F44336"
      AB5: "#FFCDD2"
    }
  }
}

```

### Color codes

!D2 color codes

Not all color codes are currently used right now, but that may change in the future for
new things that come to D2.

# Troubleshooting

- It won't compile a specific label or value
- The text is rendered too wide/long
- Connections look cluttered
- Reserved keywords as regular keys
- My diagram is breaking with HTML in Markdown
- Markdown SVGs won't render in certain SVG viewers
- My SVG isn't interactive when I embed into HTML
- Non-ASCII text breaks stuff

## It won't compile a specific label or value

It probably has some reserved characters, wrap it around `'` or `"`.

```
"x(int y)": "[]int"
'$dollarbills$'

```

## The text is rendered too wide/long

Add newlines.

```
x: When you go out to buy,\ndon't show your silver.

```

## Connections look cluttered

If you have a highly connected diagram with many connections on shapes with short labels,
you will get better results by manually settings widths and heights on them. By default,
D2 adds a minimal amount of padding to shapes past the label dimensions. When a shape has
many connections, increasing dimensions will give it more surface area for the connections
to route aesthetically to.

## Reserved keywords as regular keys

If you'd like to use a reserved keyword as a key, just quote it:

```
x: {
  "width": width
}

```

## My diagram is breaking with HTML in Markdown

Your HTML must be semantic to be parsed in SVG XML correctly, e.g. use `<br/>` instead of `<br>`.

## Markdown SVGs won't render in certain SVG viewers

D2's Markdown support is added via xhtml foreign objects, which means the SVG viewer must
have HTML rendering capabilities. The vast majority of SVG viewing is, but if you intend
to use a pure SVG editor on D2 diagrams that don't have such capabilities (e.g. Adobe
Illustrator), it won't render correctly there.

## My SVG isn't interactive when I embed into HTML

There's a few different ways to embed SVG into HTML, each with tradeoffs. If you use plain
`<img>` tags, interactivity is blocked. Here are two good resources to learn more:

. tldr:
https://docs.asciidoctor.org/asciidoc/latest/macros/image-svg/#options-for-svg-images
. The official W3 spec:
https://www.w3.org/Graphics/SVG/IG/resources/svgprimer.html#SVG\_in\_HTML

## Non-ASCII text breaks stuff

D2 works with any language, but sometimes non-ASCII characters look like reserved characters when they're not.

```
hello世界：مرحبا بالعال

```

The character `：` is not the same as the ASCII `:`, and so it won't register as a label. For foreign language diagrams, please take care to use the ASCII versions of special characters like `:`, `;`, `.`, among others.

# UML Classes

## Basics

D2 fully supports UML Class diagrams. Here's a minimal example:

```
MyClass: {
  shape: class

  field: "[]string"
  method(a uint64): (x, y int)
}

```

Each key of a `class` shape defines either a field or a method.

The value of a field key is its type.

Any key that contains `(` is a method, whose value is the return type.

A method key without a value has a return type of void.

## Visibilities

You can also use UML-style prefixes to indicate field/method visibility.

| visibility prefix | meaning   |
| ----------------- | --------- |
| none              | public    |
| +                 | public    |
| -                 | private   |
| #                 | protected |

See https://www.uml-diagrams.org/visibility.html

Here's an example with differing visibilities and more complex types:

```
D2 Parser: {
  shape: class

  # Default visibility is + so no need to specify.
  +reader: io.RuneReader
  readerPos: d2ast.Position

  # Private field.
  -lookahead: "[]rune"

  # Protected field.
  # We have to escape the # to prevent the line from being parsed as a comment.
  \#lookaheadPos: d2ast.Position

  +peek(): (r rune, eof bool)
  rewind()
  commit()

  \#peekn(n int): (s string, eof bool)
}

"github.com/terrastruct/d2parser.git" -> D2 Parser

```

## Full example

```
DebitCard: Debit card {
  shape: class
  +cardno
  +ownedBy

  +access()
}

Bank: {
  shape: class
  +code
  +address

  +manages()
  +maintains()
}

ATMInfo: ATM info {
  shape: class
  +location
  +manageBy

  +identifies()
  +transactions()
}

Customer: {
  shape: class
  +name
  +address
  +dob

  +owns()
}

Account: {
  shape: class
  +type
  +owner
}

ATMTransaction: ATM Transaction {
  shape: class
  +transactionId
  +date
  +type

  +modifies()
}

CurrentAccount: Current account {
  shape: class
  +accountNo
  +balance

  +debit()
  +credit()
}

SavingAccount: Saving account {
  shape: class
  +accountNo
  +balance

  +debit()
  +credit()
}

WidthdrawlTransaction: Withdrawl transaction {
  shape: class
  +amount

  +Withdrawl()
}

QueryTransaction: Query transaction {
  shape: class
  +query
  +type

  +queryProcessing()
}

TransferTransaction: Transfer transaction {
  shape: class
  +account
  +accountNo
}

PinValidation: Pin validation transaction {
  shape: class
  +oldPin
  +newPin

  +pinChange()
}

DebitCard -- Bank: manages {
  source-arrowhead: 1..*
  target-arrowhead: 1
}

Bank -- ATMInfo: maintains {
  source-arrowhead: 1
  target-arrowhead: 1
}

Bank -- Customer: +has {
  source-arrowhead: 1
  target-arrowhead: 1
}

DebitCard -- Customer: +owns {
  source-arrowhead: 0..*
  target-arrowhead: 1..*
}

DebitCard -- Account: +provides access to {
  source-arrowhead: *
  target-arrowhead: 1..*
}

Customer -- Account: owns {
  source-arrowhead: 1..*
  target-arrowhead: 1..*
}

ATMInfo -- ATMTransaction: +identifies {
  source-arrowhead: 1
  target-arrowhead: *
}

ATMTransaction -> Account: modifies {
  source-arrowhead: *
  target-arrowhead: 1
}

CurrentAccount -> Account: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}

SavingAccount -> Account: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}

WidthdrawlTransaction -> ATMTransaction: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}
QueryTransaction -> ATMTransaction: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}
TransferTransaction -> ATMTransaction: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}
PinValidation -> ATMTransaction: {
  target-arrowhead.shape: triangle
  target-arrowhead.style.filled: false
}

```


# Variables & Substitutions

`vars` is a special keyword that lets you define variables. These variables can be
referenced with the substitution syntax: `${}`.

```
direction: right
vars: {
  server-name: Cat
}

server1: ${server-name}-1
server2: ${server-name}-2

server1 <-> server2

```

## Variables can be nested

Use `.` to refer to nested variables.

```
vars: {
  primaryColors: {
    button: {
      active: "#4baae5"
      border: black
    }
  }
}

button: {
  width: 100
  height: 40
  style: {
    border-radius: 5
    fill: ${primaryColors.button.active}
    stroke: ${primaryColors.button.border}
  }
}

```

## Variables are scoped

They work just like variable scopes in programming. Substitutions can refer to variables
defined in a more outer scope, but not a more inner scope. If a variable appears in two
scopes, the one closer to the substitution is used.

```
vars: {
  region: Global
}

lb: ${region} load balancer

zone1: {
  vars: {
    region: us-east-1
  }
  server: ${region} API
}

```

## Single quotes bypass substitutions

```
direction: right
vars: {
  names: John and Joyce
}
a -> b: 'Send field ${names}'

```

## Spread substitutions

If `x` is a map or an array, `...${x}` will spread the contents of `x` into either a map
or an array.

```
vars: {
  base-constraints: [NOT NULL; UNQ]
  disclaimer: DISCLAIMER {
    I am not a lawyer
    near: top-center
  }
}

data: {
  shape: sql_table
  a: int {constraint: [PK; ...${base-constraints}]}
}

custom-disclaimer: DRAFT DISCLAIMER {
  ...${disclaimer}
}

```

## Configuration variables

Some configurations can be made directly in `vars` instead of using flags or environment
variables.

```
vars: {
  d2-config: {
    theme-id: 4
    dark-theme-id: 200
    pad: 0
    center: true
    sketch: true
    layout-engine: elk
  }
}

direction: right
x -> y

```

This is equivalent to calling the following with no `vars`:

```
d2 --layout=elk --theme=4 --dark-theme=200 --pad=0 --sketch --center input.d2

```

Precedence

Flags and environment variables take precedence.

In other words, if you call `D2_PAD=2 d2 --theme=1 input.d2`, it doesn't matter what
`theme-id` and `pad` are set to in `input.d2`'s `d2-config`; it will use the options from
the command.

`data`

`data` is an anything-goes map of key-value pairs. This is used in contexts where
third-party software read configuration, such as when D2 is used as a library, or D2 is
run with an external plugin.

For example,

```
vars: {
  d2-config: {
    data: {
      power-level: 9000
    }
  }
}

```


# Version visualization

## History

You want to understand how the schema of a system has evolved over time. As long as the
diagram is modularized with imports, such a visualization is easy to whip up.

- `history.d2`

```
direction: right
Users 1: Users Table (v0.1) {
  ...@"users-v0.1"
}

Users 2: Users Table (current) {
  ...@users-current
}

Users 1 -> Users 2

```

- `users.d2` (latest version, 0.2)

```
users: {
  shape: sql_table
  id: int {constraint: primary_key}
  email: int {constraint: foreign_key}
  name: string
  password: text
  created_at: timestamp
  last_updated: timestamp
}

emails: {
  shape: sql_table
  id: int {constraint: [primary_key; unique]}
  local: string
  domain: string
  verified: boolean
}
users.email -> emails.id

```

- `users.d2` (0.1)

```
users: {
  shape: sql_table
  id: int {constraint: primary_key}
  email: string
  name: string
  verified_email: boolean
  password: string
  created_at: timestamp
}

```

Since you want how `users.d2` looked like at `v0.1`, you use `git` to get that version:

```
cp users.d2 users-current.d2
git checkout tags/v0.1 users.d2
mv users.d2 users-v0.1.d2

```

## Compare

You're a manager at Apple who has two teams secretly working on the same product for
multiple years. After years of iterating in their silos, you form a committee to compare
the two results. The evaluation starts by comparing their design decisions in an
overarching diagram projected in a dark room behind a closed door.

- `compare.d2`

```
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

```
gh repo clone apple/team-alpha
gh repo clone apple/team-charlie

cp apple/team-alpha/schema.d2 alpha-schema.d2
cp apple/team-charlie/schema.d2 charlie-schema.d2

cp apple/team-alpha/api.d2 alpha-api.d2
cp apple/team-charlie/api.d2 charlie-api.d2

```

The rendered diagram is left as an exercise to the reader.

# Vim plugin

D2 has an official, creator-maintained plugin for Vim. You can use any plugin manager to
include `terrastruct/d2-vim`. For example:

```
Plug 'terrastruct/d2-vim'

```

Automatic indentation and syntax highlighting are fully supported and make working with
the D2 language far more pleasant. Autoformat on each save is planned.

The syntax highlighting will even catch basic errors for you if your color theme
highlights illegal syntax.

**Github:** https://github.com/terrastruct/d2-vim

# VSCode extension

D2 has an official, creator-maintained extension for VSCode. It's searchable and
downloadable through the VSCode marketplace for free.

Automatic indentation and syntax highlighting are fully supported and make working with
the D2 language far more pleasant. Split-screen diagramming within VSCode is planned.

The syntax highlighting will even catch basic errors for you if your color theme
highlights illegal syntax.

**Github:** https://github.com/terrastruct/d2-vscode


# TALA User Manual

**Version 0.4.1 (compatible with D2 0.6.8 and up)**

*Terrastruct*
*November 7, 2024*

---

## Contents

1.  **[Introduction](#1-introduction)**
2.  **[Command-line usage](#2-command-line-usage)**
    *   [2.1 Installation](#21-installation)
    *   [2.2 Using TALA](#22-using-tala)
    *   [2.3 License key](#23-license-key)
3.  **[Considerations for software architecture](#3-considerations-for-software-architecture)**
    *   [3.1 Symmetry](#31-symmetry)
    *   [3.2 Clusters](#32-clusters)
    *   [3.3 Hierarchy](#33-hierarchy)
    *   [3.4 Balanced connections](#34-balanced-connections)
    *   [3.5 Dynamic label positioning](#35-dynamic-label-positioning)
    *   [3.6 Square aspect ratio](#36-square-aspect-ratio)
    *   [3.7 Direction](#37-direction)
    *   [3.8 Near](#38-near)
    *   [3.9 SQL table column matching](#39-sql-table-column-matching)
    *   [3.10 Sequences](#310-sequences)
    *   [3.11 Self connections](#311-self-connections)
    *   [3.12 Ensure port space for connections](#312-ensure-port-space-for-connections)
    *   [3.13 Prefer horizontal connections for many labels](#313-prefer-horizontal-connections-for-many-labels)
    *   [3.14 Grid edges](#314-grid-edges)
4.  **[Seed](#4-seed)**
5.  **[Positioning](#5-positioning)**
    *   [5.1 Relative to the container](#51-relative-to-the-container)
    *   [5.2 Web app](#52-web-app)
6.  **[Tips](#6-tips)**
    *   [6.1 Dimensions](#61-dimensions)
    *   [6.2 Label collisions](#62-label-collisions)
    *   [6.3 Legends](#63-legends)
7.  **[FAQ](#7-faq)**
    *   [7.1 Does TALA on the CLI require an internet connection?](#71-does-tala-on-the-cli-require-an-internet-connection)
    *   [7.2 Can I use TALA for diagrams which are not software architecture?](#72-can-i-use-tala-for-diagrams-which-are-not-software-architecture)
    *   [7.3 Can I use TALA to layout separately from D2?](#73-can-i-use-tala-to-layout-separately-from-d2)

---

## 1 Introduction

Thank you for purchasing a license for TALA (or if you're using it in the web app, for being a Terrastruct customer!).

TALA (short for Terrastruct's AutoLayout Approach) is a proprietary layout engine designed for software architecture diagrams. The software is written entirely in Go, and was developed by Terrastruct over the span of 2 years. TALA is new, but already its current form can often produce diagrams that are preferred over the layouts generated by any other engine.

Unlike other layout engines, TALA is not one algorithm. It is comprised of many algorithms in multiple stages. It identifies parts of the diagram that are structurally like trees, or hierarchies, clusters, etc, and runs different algorithms for each. We used a combination of ideas from the decades of research done in the field of graph visualization, with modifications and new additions to tailor it to software architecture diagrams. More on this in a following section.

If you find an issue in TALA (or this manual), please file it on [TALA's GitHub](https://github.com/terrastruct/TALA). If it's a D2 issue, please file it on [D2's GitHub](https://github.com/terrastruct/d2). Usually anything that pertains to the layout (sizing, positioning) falls under TALA. You can see how active we are in public and be sure that your issue will quickly be attended to.

This manual is meant to accompany the documentation at [https://d2lang.com](https://d2lang.com). You should familiarize yourself with D2's language first, as this manual does not cover usage of D2.

We'd love to hear from you. Suggestions, ideas for integrating TALA into something, bugs, good things, bad things – all welcome. The above GitHub links are best, so that everyone can see/search and chime in on an open discussion, but we're also around for live chat ~18hrs a day on [Discord](https://discord.gg/aKXhwpCvrY). For private enquiries, email info@terrastruct.com.

## 2 Command-line usage

If you're using TALA through Terrastruct's web app, you can skip this section (as it's installed on our servers).

### 2.1 Installation

The most convenient way to install is through the install script, together with D2.

```sh
# With --dry-run the install script will print the commands it will use
# to install without actually installing so you know what it's going to do.
curl -fsSL https://d2lang.com/install.sh | sh -s -- --tala --dry-run

# If things look good, install for real.
curl -fsSL https://d2lang.com/install.sh | sh -s -- --tala
```
*Listing 1: Installing TALA*

If you want to upgrade versions, just re-run the script.

Alternatively, you can also find binaries [releases](https://github.com/terrastruct/TALA/releases) for Linux and MacOS, for both AMD and ARM, and an installer for Windows.

We also have a Homebrew tap, so you can install via Homebrew if you'd like (the install script uses Homebrew if detected on your machine).

```sh
brew install terrastruct/tap/tala
```

TALA's binary is named `d2plugin-tala`. To check that it was properly installed and in your PATH, run the following.

```sh
d2plugin-tala --version
```

To check that D2 can find TALA, run the following.

```sh
d2 layout tala
```

### 2.2 Using TALA

To use TALA for D2's layout engine, pass the flag `layout` flag:

```sh
d2 --layout tala input.d2
```

Shorthand:

```sh
d2 -l tala input.d2
```

You can also set it to an environment variable to not have to specify it each time:

```sh
export D2_LAYOUT=tala
d2 input.d2
```

### 2.3 License key

The easiest way to reference your license key is to put it in your PATH.

```sh
export TSTRUCT_TOKEN="tstruct_..."
```

You can also write the key to a file. By default, TALA will look for this file in your home directory's `.config/tstruct/auth.json`.

```json
{
  "api_token": "tstruct_xxx"
}
```

You can override that the location of the auth file.

```sh
export TSTRUCT_AUTHFILE="/home/my/path/config.json"
```

## 3 Considerations for software architecture

Instead of relying solely on theoretical cross-minimizing heuristics to dictate layout like other engines might, TALA emulates how software engineers might draw diagrams on a whiteboard. What follows is a non-exhaustive list of considerations that TALA makes towards that goal.

### 3.1 Symmetry

Symmetry is an important part of aesthetics and its weighted in TALA's algorithm. There are instances where a shape could've been closer to its neighbor, but doing so would've broken symmetry, and so it gets placed in the position that gives it more symmetry.


*Figure 1: Example of symmetry*

### 3.2 Clusters

Clusters are multiple shapes connected to the same shape as its sibling, at one or both ends. They are laid out all on the same side as their cluster siblings. For example, notice that "a" and "c" could have minimized distance by going to "x"'s left and right sides, but chose to remain clustered together instead.


*Figure 2: Example of a cluster*

### 3.3 Hierarchy

Hierarchical structures are identified by multiple levels of shapes connected in the same direction and undergoes layout with a special set of rules (e.g. even spacing between levels). Uniquely, TALA handles containers within hierarchies.


*Figure 3: Example of a hierarchy with containers*

You may also force a hierarchical shape even when the algorithm has not detected that it's the most suitable structure by adding `shape: hierarchy`.

### 3.4 Balanced connections

When routing connections, TALA tries to target ports that make them look balanced, given other connections. For example, if it's the only connection on that side, it will try to route to the center – if 3, then at 1/3 intervals, etc.

![Three diagrams demonstrating connection balancing. (a) shows connections balanced on a node. (b) shows connections balanced only if there is enough room. (c) shows connections balanced across different sides of a node.](https://i.imgur.com/vH977y2.png)
*Figure 4: Example of connection balancing*

### 3.5 Dynamic label positioning

Label positions are fixed everywhere. They are positioned to where they avoid obstructions like other shapes and connections.


*Figure 5: Shape label going to top-left to not collide with routes*


*Figure 6: Connection labels avoiding each other*

### 3.6 Square aspect ratio

Non-connected subgraphs are bin-packed to prefer a square aspect ratio.


*Figure 7: Example of bin-packing*

### 3.7 Direction

Unlike other layout directions, TALA is able to specify `direction` keyword per container level. The default direction puts a slight preference on down-right (connections point down or right), though if you specify, the preference is stronger.

This keyword is a suggestion, not a requirement. Some connections may still flow the other direction when it improves other heuristics. But in general, the children of container will prefer the direction specified.


*Figure 8: Top level direction is down, container a's direction is left, container b's direction is up, container c's direction is right*

### 3.8 Near

On top of constants (e.g. `top-center`), `near` can target other shapes in TALA. Provide the absolute ID (e.g. a shape `world` inside of container `hello` is specified as `hello.world`) to the `near` property to tell TALA to place it in proximity of that shape.

![Two diagrams showing the 'near' attribute. In (a), node 'b' is set to be near 'x' and is placed directly below it. In (b), node 'b' is set to be near 'y' and is placed below it, away from 'x'.](https://i.imgur.com/w9C6JvG.png)
*Figure 9: Example of the near attribute*

### 3.9 SQL table column matching

In TALA, connections between two `sql_table` shapes that specify the column will route to that specific column. These diagrams are also known as Entity-Relation Diagrams (ERDs).

![A side-by-side comparison of ERD routing. (a) Dagre shows connections pointing to the center of table shapes. (b) TALA shows connections precisely routed to the specific rows/columns within the table shapes that they are linked to.](https://i.imgur.com/p51gS7K.png)
*Figure 10: Comparison of ERD routing in Dagre vs TALA*

### 3.10 Sequences

If you connect a chain of shapes with their type set to `step`, TALA arranges them tail to head instead of using connections as other shape types would.


*Figure 11: Example of multiple step shapes to form sequence*

### 3.11 Self connections

A self-connection is `x -> x`. In TALA, one corner is reserved for one type of connection (there are 3 types: directed, undirected, bidirected). If there are multiple connections of the same type, the corner is reused.

Notice how the outer loop makes room for the inner loop's label.

![A single shape with three different self-connections. Each connection (labeled 'feedback loop A', 'loop B', 'talk to self', 'neutral') loops back to the shape, originating from and returning to a different side or corner.](https://i.imgur.com/49oM74v.png)
*Figure 12: Example of multiple self-connections of varying relationships*

### 3.12 Ensure port space for connections

TALA ensures sufficient space for connections to aesthetically space themselves along a side. It does this by scaling the dimensions of a node if its default dimensions dictated by the label is too small.

Notice how the nodes scale in the below figure.

![Two diagrams showing node scaling. In (a), two nodes 'x' and 'y' with 2 connections have a certain size. In (b), with 5 connections, the nodes have scaled up vertically to accommodate the additional connections neatly.](https://i.imgur.com/E8b3q2n.png)
*Figure 13: Node scaling*

### 3.13 Prefer horizontal connections for many labels

Imagine you have 3 vertical connections, all with labels. The middle one will find it hard to avoid colliding with the left and right edges.

For that reason, when two nodes have 3 or more connections between them, all with labels, those nodes will prefer a horizontal position with each other. The exception to this is when you've explicitly set a `direction` keyword of "down" or "up".


*Figure 14: Example of preferring horizontal positions for many connection labels*

### 3.14 Grid edges

Since grids are a separate layout structure, they are outside the control of Dagre and ELK. To route connections in-between grid cells, D2 just uses a straight line from center to center. However, TALA has a standalone edge router. This allows edges between grid cells rendered with TALA to be routed better than straight lines.


*Figure 15: Grid connections without TALA*


*Figure 16: Grid connections with TALA*

## 4 Seed

Optimal placements of nodes that minimizes distance and crossings is NP-hard. More so when constraints like hierarchical are removed. TALA searches with heuristics to approximate. This search space has some randomness at each step. Choosing a different seed for this randomness can have significant impact on the overall layout, as it may converge on an entirely different look at the end.

The Terrastruct web app takes 3 seeds and runs them all in parallel, and uses a different set of heuristics to choose the best one. On the CLI, you can specify the seed with a flag, `tala-seeds`.

```sh
d2 --layout tala --tala-seeds 44 input.d2
```
*Listing 2: Specifying the seed on the CLI*

You can also provide more than one seed, which runs multiple executions using the different seeds, in parallel, and uses a heuristic to choose the best outcome.

```sh
d2 --layout tala --tala-seeds 8,3,24 input.d2
```
*Listing 3: Specifying multiple seeds on the CLI*

The intended usage is to "re-roll" for a better layout if TALA did not give an ideal layout with the default. The default is to race the seeds 1, 2, 3.

For example, below, the seed 999 happens to give a better outcome (more symmetrical, straighter lines) than the default. So when it's passed, even alongside others, the resulting diagram is different.

![Two diagrams of the same system layout. (a) shows the layout with default seeds, which is somewhat asymmetrical. (b) shows the layout generated with specific seeds (1,2,3,999), resulting in a more symmetrical and organized diagram.](https://i.imgur.com/M6P7c0f.png)
*Figure 17: Same diagram, different seeds*

If your computer is more powerful, you can run more seeds in parallel all the time. If TALA is taking too much resources by default, you can set it to use one seed all the time.

Not all diagrams will look different with different seeds – multiple random seeds may converge to the same diagram.

## 5 Positioning

Being able to specify fixed positions is an extremely powerful way to customize into your ideal diagram.

There are 2 scenarios where you might want to specify positions:

*   You have a specific configuration in mind that can't be described in D2. (e.g. maybe you want to overlap two shapes in a specific way).
*   You want to override a local layout decision.


*Figure 18: Using custom positions to construct a very specific configuration*

```d2
direction: down
TCP/IP Stack: {
  IP Datagram: {
    top: 0
    left: 0
    IP Header: {
      top: 0
      left: 0
    }
    TCP/UDP Header: {
      top: 0
      left: 100
    }
    Upper Layer Headers: {
      top: 0
      left: 250
    }
    Upper Layer Data: {
      top: 0
      left: 440
    }
  }
  Layer 2 Frame: {
    top: 190
    left: 0
    Layer 2 Header: {
      top: 0
      left: 0
    }
    Rest: ... {
      top: 0
      left: 140
    }
    Upper Layer Data: {
      top: 0
      left: 190
    }
    Layer 2 Footer: {
      top: 0
      left: 350
    }
  }
}
Request -> TCP/IP Stack
```

### 5.1 Relative to the container

Positions are relative to the shape's container. If the shape does not have a container, it's relative to the bounding box of the diagram.

When a position is specified within a container, the top-left is after padding has been applied. Here's an example:

![A diagram demonstrating relative positioning. An 'Input' shape is at absolute position (80, 80). A 'Processing' container contains a 'Renderer' at relative position (0, 100) and a 'Validator'. The coordinates are shown relative to the container's padded area.](https://i.imgur.com/rN5HqG5.png)
*Figure 19: Relative positioning*

```d2
Input: {
  top: 0
  left: 0
}
Processing: {
  top: 80
  left: 80
  Renderer: {
    top: 0
    left: 0
  }
  Validator: {
    top: 100
    left: 0
  }
}
```

`Processing` is specified at `80, 80`, and its bounding box is itself + `Input`. There is no padding. `Renderer` however is specified inside a container, and though it's specified as `0, 0`, it doesn't occupy the same position as `Processing`, because of the container padding.

### 5.2 Web app

Positioning and sizing are precise operations. D2's watch mode lets you easily choose a value, save, see the result, and repeat until you get the desired effect. And while that's a quick feedback loop, this is one area where using a precise tool is best. In the web app, turning on syncing for positions or dimensions lets you drag and drop to your desired position, updating the values automatically.


*Figure 20: Turning on syncing*

## 6 Tips

### 6.1 Dimensions

By default, the dimension of a shape is the dimension of its label, with some padding. TALA also changes dimensions to expand containers enough to fit children, make cluster siblings uniform, and ensure enough room for highly connected objects. Aside from these cases, it won't know if you intended a shape to have a larger presence than others. For example, a central "hub" that many connect to. To achieve layouts that more closely match the ideal, you should supply `width` and `height` to shapes where you imagine it to be larger or smaller in scale than others.


*Figure 21: Default dimensions*

```d2
Software bugs: {
  Compiler error
  Human error
}
```


*Figure 22: Custom dimensions*

```d2
Software bugs: {
  Compiler error: {
    width: 180
    height: 50
  }
  Human error: {
    width: 300
    height: 300
    style.font-size: 40
  }
}
```

Also, connections may look worse bending and twisting to try to reach a well-connected shape situated in the middle of obstructions. Giving custom dimensions can give more space for routes to connect to.

### 6.2 Label collisions

Though labels dynamically avoid collisions, it will not influence the layout to make room for a disproportionately sized label. For a long label, you may find better results splitting them up into multiple lines with `\n`.

### 6.3 Legends

Making a legend is a common task that you can do by combining positions and dimensions.


*Figure 23: Creating a custom legend for your diagram*

```d2
...
Legend: {
  top: 0
  left: 0
  blue: "" {
    width: 20
    height: 20
    left: 0
    top: 0
    style.stroke: black
  }
  blue-explanation: Actions {
    shape: text
    left: 50
    top: 0
  }
  green: "" {
    width: 20
    height: 20
    left: 0
    top: 50
    style.fill: honeydew
    style.stroke: black
  }
  green-explanation: Intermediate artifacts {
    shape: text
    left: 50
    top: 50
  }
}
...
```

## 7 FAQ

### 7.1 Does TALA on the CLI require an internet connection?

No, TALA does not collect telemetry or use the internet in any way except to ping to check the status of a license. This is only done when necessary, e.g. if you purchased a month subscription, TALA will ping at the start of the next month and renew automatically if the subscription is ongoing. If you purchased a year, it won't ping for a year. The only data that's sent in these pings is the API token itself. No diagrams or anything else leaves your computer.

### 7.2 Can I use TALA for diagrams which are not software architecture?

Yes, and it's known to give good results in many cases. But there's two things to note.

1.  There are many categories of diagram types which are "solved". For example, for mind maps, the algorithm that mind-map-specific software has no improvements left to be made. If your diagram type falls under one of these categories, you'll definitely get better results with specific software. Perhaps D2 will support these, but it will be outside of TALA (e.g. D2's sequence diagram layout engine is bundled in D2).
2.  TALA is tested exclusively on software architecture diagrams. It is the focus, and we will not change the algorithm to better handle any other type.

### 7.3 Can I use TALA to layout separately from D2?

Yes, it's possible - they communicate with a serialized, JSON representation of a diagram. There's no reason you couldn't build that JSON from somewhere other than D2. However, it's not an intended use case, so isn't well-documented. If you have a use case in mind, please let us know and we'll help you.
