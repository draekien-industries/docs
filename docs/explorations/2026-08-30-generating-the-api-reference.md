---
title: Generating the API reference from XML doc comments
date: 2026-08-30
status: active
supersedes:
---

# Generating the API reference from XML doc comments

Spike for DRA-175. Run against `Waystone.Monads` at `release/v7.0.0`, .NET SDK 10.0.111.

## The question

The reference pages restate what the XML doc comments already say. Should the reference
be generated from the source instead of hand-written, before DRA-169 hand-writes roughly
1,750 lines of it?

## What the source material looks like

Better than expected, and this changes the answer.

- `src/Directory.Build.props` already sets `GenerateDocumentationFile`, so
  `Waystone.Monads.xml` falls out of an ordinary build. It is 11,438 lines.
- It carries **1,129 documented members** — 61 types, 980 methods, 62 properties, 26 fields.
- The comments carry real contracts, not restatements of the signature. `Option<T>.Map`
  documents why a null projection throws instead of collapsing to `None`, and names the
  analyzer rule (`WM2017`) that reports the capturing overload. That is material worth
  publishing.
- `PublicAPI.Shipped.txt` holds 1,156 entries, so there is an exact oracle for whether
  generated output is complete.

The input is good. The question is only whether a tool can turn it into pages.

## What was tested

Both candidates were run against the published output (`dotnet publish`), so dependencies
were resolvable.

### XMLDoc2Markdown 6.0.0 — rejected

Two defects, either of which is disqualifying.

**It cannot see through C# extension blocks.** `OptionExtensions` and `ResultExtensions`
are written with the extension-block syntax:

```csharp
extension<T>(Option<Option<T>> option) where T : notnull
```

The compiler emits a nested type per block, named `<g>$<hash>`. XMLDoc2Markdown reflects
over the assembly, finds those types, and tries to write one file per type:

```
The filename, directory name, or volume label syntax is incorrect. :
  '...\waystone.monads.options.extensions.optionextensions.<g>$02a1f6b98ba397766c5d457032e9b00b-1.md'
```

`<`, `>` and `$` are not legal in a Windows filename. The run reports
`Generation: 30 succeeded, 52 failed`. Roughly two thirds of the surface is silently
lost, and there is no setting that excludes compiler-generated types.

**It drops `<see langword="..."/>` entirely.** The source uses it 115 times. The output
reads:

> Returns  if the option is a [Some&lt;T&gt;](...) value.

The word `true` is simply gone. That is 115 broken sentences, each of which looks like an
editing mistake rather than a tool defect.

Its output is otherwise the more readable of the two, which is worth recording — but not
enough to matter.

### DefaultDocumentation 1.2.5 — works

It reads the XML file rather than reflecting over the assembly, so the extension-block
problem does not arise. Extension methods are documented normally, and `langword` renders
correctly.

Left at its defaults it produces 545 files: one per member, including 107 pages
documenting PolySharp's injected polyfill attributes. Both problems are configuration:

```
defaultdocumentation -a Waystone.Monads.dll -d Waystone.Monads.xml \
  -o <out> -g Namespaces,Types -s public
```

That gives **38 files, one per type, with zero polyfill noise** — the right granularity
for the tree in DRA-161.

## The catch

Complete is not the same as readable.

| | Lines |
| --- | --- |
| Hand-written reference today (three pages) | 1,760 |
| Generated at type granularity | 29,420 |

Sixteen times the volume. `Option<T>` alone generates a 2,288-line page, against 940 for
the current hand-written one that covers *both* types.

The volume is not padding, it is link soup. Every type reference expands to an inline link
carrying a title attribute, so a single parameter renders as:

```
`map` [System\.Func&lt;](https://learn.microsoft.com/...)[T](Waystone.Monads.Options.Option_T_.md#... 'Waystone\.Monads\.Options\.Option\<T\>\.T')[,](https://learn.microsoft.com/...)[TOut](...)[&gt;](...)
```

Every literal dot is also backslash-escaped. It renders acceptably, but nobody will ever
hand-edit it, and reviewing a diff of it is not realistic.

## What this rules in and out

Restating the three options from DRA-175 against what was found:

1. **Generate everything with an off-the-shelf tool.** Available today, costs about a day.
   Complete and never stale. Gives up readability entirely, and gives up the "when to
   reach for this" framing that the guides cannot fully absorb.
2. **Generate the contract, hand-write the guidance.** Not reachable with either tool's
   default output — the generated half is too verbose to sit beside hand-written prose.
   It *is* reachable another way, see below.
3. **Keep hand-writing, add a completeness check.** A test that fails when
   `PublicAPI.Shipped.txt` names a member the reference does not mention. Cheap, and it
   catches the drift that actually matters.

A fourth option appeared during the spike, and it is the interesting one.

4. **Write a small generator over the XML file directly.** The XML is clean, complete, and
   — critically — uses ordinary documentation IDs even for extension-block members:

   ```
   M:Waystone.Monads.Options.Extensions.OptionExtensions.Unzip``2(...)
   ```

   Everything that defeated XMLDoc2Markdown is an artefact of reflecting over the
   assembly. Reading the XML avoids all of it. A generator emitting exactly the entry
   shape DRA-169 specifies — signature, contract, when to reach for it, sample, empty and
   error case — is a few hundred lines, and it can group entries into the categories the
   tree already uses rather than dumping them alphabetically.

## What the evidence favours

Option 4, guarded by option 3.

The reason is that the expensive input already exists and is good. The doc comments carry
the contracts, `PublicAPI.Shipped.txt` says what must be covered, and the XML file joins
them. What is missing is not information but a renderer that emits the shape this space
wants — and neither off-the-shelf tool will emit that shape, because neither was built for
a hand-designed page tree.

Option 1 remains the fallback if option 4 is not funded. It is genuinely available, it is
a day's work, and complete-but-ugly beats absent. It should not be chosen quietly, though:
it makes the reference something no one reads by choice.

Option 3 is worth doing regardless of which of the others is chosen, because it is the
only one that fails a build when the surface and the docs disagree.

## Not settled here

- Where the "when to reach for this" sentence lives if the reference is generated. It is
  not in the XML today. It could be added as a dedicated tag the generator reads, which
  would keep it beside the code it describes.
- Whether GitBook renders DefaultDocumentation's escaping cleanly. The output was judged
  as markdown, not on the live site.
- Whether the same approach holds for the companion packages, which were not built or
  generated during this spike.

## Reproducing it

```
dotnet publish src/Waystone.Monads/Waystone.Monads.csproj -c Release -o <pub>
dotnet tool install --tool-path <tools> DefaultDocumentation.Console
<tools>/defaultdocumentation -a <pub>/Waystone.Monads.dll \
  -d <pub>/Waystone.Monads.xml -o <out> -g Namespaces,Types -s public
```
