**PrintPreview --- Spring Boot Development Guide**

*Code-based guide for the PrintPreview CLI that renders Critx documents
to LaTeX and compiles them with XeLaTeX.*

# Getting started (download → build → run)

PrintPreview is a Spring Boot application, but you should think of it as
a batch compiler, not a service. It starts, processes one .critx
document, writes build artifacts, and exits. There is no HTTP layer and
no interactive mode: everything is driven by a strict, positional CLI
contract.

**Step 1 --- Download the repository and check prerequisites**

After you clone/download the repository, you need:

- Java 21 (to run locally)

- Maven (to build the runnable JAR)

- Criterion installed only if you plan to test in "integrated" mode
  (inside Criterion).

**Step 2 --- Build the application (JAR)**

From the project root (the folder that contains pom.xml), build the
runnable Spring Boot JAR with:

mvn -U clean package

After the build, look in target/. You will typically see:

- target/\<artifactId\>-\<version\>.jar (this is the executable "fat
  jar", if Spring Boot repackaging is enabled)

- target/\<artifactId\>-\<version\>.jar.original (sometimes present; the
  pre-repackage thin jar)

Make sure you run the executable fat jar. If repackaging is not active,
Maven may produce only a thin jar (usually small) and running it will
fail with ClassNotFoundException.

**Step 3 --- Choose how to run it**

You have two supported execution modes: integrated (via Criterion) and
standalone (direct CLI).

**A) Integrated mode (run through Criterion)**

Use this mode when you want to test your changes through Criterion's
Print Preview functionality.

Prerequisite: Criterion must already be installed on your machine.

Once you have built a new JAR, copy it into:

\<INSTALLATION-DIRECTORY\>\\resources\\buildResources\\printPreview

Then launch Criterion and test using the Print Preview feature.

**B) Standalone mode (run the CLI directly -- no Criterion required)**

Standalone mode is designed around a "portable folder": everything the
app needs sits next to the JAR. The key detail is that the app resolves
the embedded XeLaTeX toolchain relative to the current working directory
(user.dir). That means you must run the JAR from an "app home" folder
that contains the JAR plus its resources in predictable locations.

1\. Create an APP_HOME folder (portable runtime layout)

Create a folder anywhere you like (this will be your \<APP_HOME\>), then
place the JAR and resources like this:

\<APP_HOME\>/

printpreview-0.0.1-SNAPSHOT.jar

fonts/

TinyTeX-win/... (Windows only)

TinyTeX-lin/... (Linux only)

TinyTeX-mac/... (macOS only)

You only need the TinyTeX-\* folder that matches your OS. The fonts/
folder matters because XeLaTeX uses it when the generated LaTeX contains
"foreign" scripts (Greek, Hebrew, Arabic, Cyrillic, etc.). If fonts are
missing, compilation can fail or you can get broken output (missing
glyphs / wrong substitutions). You can get the fonts/ folder from both
the installed Criterion (under
\<INSTALLATION-DIRECTORY\>\\resources\\buildResources\\printPreview or
from the Criterion repository)

**Important**: do not download TinyTeX from the official website for
this setup. The TinyTeX distribution used here is customized for
Criterion/PrintPreview, and the standard download will not work out of
the box. Instead, copy the TinyTeX folder from a Criterion installation
or obtain it from the project repository used for Criterion.

2\. Understand compileDir (before you run anything)

Every run needs a compileDir. This is not just "where the PDF goes": it
is the working directory for the LaTeX build. The tool writes the
generated .tex there, runs XeLaTeX, and leaves the PDF plus LaTeX
auxiliary/log files there as well.

Pick compileDir as a disposable folder outside \<APP_HOME\>, for
example:

- Windows: C:\\temp\\ppn-build

- macOS/Linux: /tmp/ppn-build

**DANGER --- READ THIS FIRST**

At startup, the application cleans compileDir by deleting every file it
finds there (files only). That is intentional (it prevents stale LaTeX
artifacts from affecting results), but it also means:

- never point compileDir at a folder that contains anything you care
  about

- never point it at your install folder / \<APP_HOME\>

- keep it separate from the application's binaries and resources

3\. Run a minimal end-to-end test

Windows (PowerShell): First cd into \<APP_HOME\> (the folder that
contains the jar, fonts, and TinyTeX-\*), then run:

cd C:\\path\\to\\APP_HOME

java -jar printpreview-0.0.1-SNAPSHOT.jar \"C:\\path\\to\\sample.critx\"
1 1 1 1 \"C:\\temp\\ppn-build\" 4

macOS / Linux (bash/zsh): Again, cd into \<APP_HOME\>, create
compileDir, then run:

cd /path/to/APP_HOME

mkdir -p /tmp/ppn-build

java -jar printpreview-0.0.1-SNAPSHOT.jar \"/path/to/sample.critx\" 1 1
1 1 \"/tmp/ppn-build\" 4

4\. What you should see in compileDir

If your input is sample.critx, the program derives the base name
"sample" and writes:

- sample.tex (always)

- sample.pdf (only if XeLaTeX succeeds)

Step 4 --- The CLI contract (positional arguments)

This application is intentionally strict: there are no named flags and
no interactive prompts. Arguments are parsed by index, so if you shift
one parameter, everything after it becomes wrong. In practice, treat the
invocation like a compiler command and consider wrapping it in a script
or an IDE run configuration to avoid human error. The required arguments
are:

Position 0: critxFilePath

Path to the .critx JSON document.

Position 1--4: section flags (printToc, printIntro, printMain,
printBiblio)

Each flag is interpreted with a very strict rule: only the string \"1\"
means true; any other value means false. This selectively includes or
omits entire document parts from the generated LaTeX.

Position 5: compileDir

Disposable working directory for the build (cleaned at startup). Keep it
outside \<APP_HOME\> and never use a directory with valuable files.

Position 6: compileRuns

Number of XeLaTeX passes. Allowed values are only 1, 2, or 4. Multiple
passes are normal in LaTeX (TOC/cross-references and reledmac-style
constructs often need reruns to stabilize). Use 4 when you want the most
stable output.

Position 7..n: commentAuthors (optional)

Any additional arguments are treated as a list of comment author
identifiers used to filter which comments/annotations are rendered.

Example with comment author filters

Windows:

java -jar printpreview-0.0.1-SNAPSHOT.jar \"C:\\path\\to\\sample.critx\"
1 1 1 1 \"C:\\temp\\ppn-build\" 4 \"mario.rossi\"

macOS/Linux:

java -jar printpreview-0.0.1-SNAPSHOT.jar \"/path/to/sample.critx\" 1 1
1 1 \"/tmp/ppn-build\" 4 \"mario.rossi\"

# Purpose and mental model

PrintPreviewNav is a Spring Boot application that behaves like a batch
compiler rather than a service: it starts, processes one document,
produces artifacts, and exits. That's why it is configured with
WebApplicationType.NONE: there is no HTTP layer, no controllers, no
request/response lifecycle. Spring Boot is used as a lightweight runtime
harness to load configuration, wire a few collaborating components, and
provide consistent logging and error reporting, but the "business logic"
is intentionally kept in plain Java classes so it can run and be tested
outside Spring.

What the application consumes is a Critx JSON document (.critx). In
practice this file carries two kinds of information that must be kept in
sync: the core textual structure (a tree of
nodes/sections/paragraph-like blocks) and the "paratext" that controls
how that structure should be rendered (layout rules, template choices,
formatting preferences), plus the scholarly apparatus material (notes,
variants, references) that must be laid out relative to the base text.
The output is not "a PDF directly"; instead the application first
produces a deterministic LaTeX source, then delegates final rendering to
XeLaTeX. This separation is a design choice: LaTeX is treated as a
stable, explicit intermediate representation---like an assembly language
for typesetting---so you can inspect what was generated, reproduce
builds, and debug layout issues without needing to instrument a GUI.

The most useful mental model is a compiler pipeline with a strict
boundary between front-end and back-end.

On the front-end side, the program reads the Critx JSON and builds an
in-memory representation of the document as a tree. This is where
"shape" is established: which parts are sections, where paragraphs live,
what metadata is attached, and where apparatus anchors occur.
Immediately after parsing, the system normalizes that tree.
Normalization is the step that turns "whatever the authoring tool
happened to emit" into "what this compiler is willing to process":
missing defaults are filled, optional fields are coerced into a
consistent form, whitespace or style declarations are standardized, and
structural invariants are enforced (for example, ensuring nodes appear
where the renderer expects them, and that note references can be
resolved deterministically). If something is invalid, this is the moment
it should fail loudly with an actionable error, rather than producing a
subtly broken PDF.

Next comes the overlay phase for the apparatus. This is where the
program effectively computes an additional layer of content that must be
typeset in relation to the base text. The key idea is that apparatus
notes are not just "extra paragraphs": they are tied to anchors in the
text, they may have ordering rules, they may be split into multiple
apparatus streams (A/B or similar), and they often need formatting and
numbering rules that depend on both local context (the paragraph) and
global settings (document-level apparatus configuration). Treat this as
the semantic analysis phase of a compiler: references are resolved,
numbering and grouping decisions are made, and the system produces a
render-ready model where "what must appear" is fully explicit.

Only after the tree is stable and the apparatus overlay has been
computed does PrintPreviewNav move into rendering. Rendering is
intentionally deterministic: given the same Critx input and the same
configuration, it should generate the same LaTeX output byte-for-byte
(or close enough that differences are meaningful). This is crucial
because LaTeX compilation can be a noisy black box; determinism lets you
debug by comparing generated sources, not by guessing what changed.
Rendering typically means selecting the right LaTeX templates/macros,
emitting preamble and package configuration based on settings, then
emitting document content in a predictable order: sections, paragraphs,
inline markers, apparatus blocks, and any additional paratext (headers,
footers, navigational constructs, etc.). The renderer's job is not to
"be clever"; it is to translate a fully-decided internal model into
explicit LaTeX.

On the back-end side, the system compiles LaTeX to PDF using XeLaTeX.
The "embedded XeLaTeX binary" aspect matters because it makes the tool
reproducible across machines: instead of relying on whatever TeX
distribution happens to be installed (or not installed) on the host,
PrintPreviewNav ships with a TinyTeX-based toolchain and selects the
correct executable for the current OS. That OS selection is a
portability concern, not a feature: the goal is that a Windows machine,
a Linux CI runner, and a macOS laptop all produce the same PDF from the
same source without manual environment setup. Compilation also implies
controlled working directories, predictable output paths, and capture of
compiler logs. In a pipeline mindset, the LaTeX log is your "backend
diagnostics": when a build fails, you want the exact LaTeX file, the
exact command line (or equivalent), and a clean error summary that
points to the relevant part of the generated output.

Finally, the architecture encourages testing at the right seam. Most of
the meaningful correctness is in the front-end phases---parsing,
normalization, overlay computation, and rendering---so those pieces are
built as plain Java helpers that can be unit-tested with a small Critx
snippet and asserted against stable outputs (for example, a rendered
LaTeX fragment). Spring Boot should not be required to test "how
paragraphs are processed" or "how apparatus numbering is computed"; it
should only be necessary when you want to test wiring, configuration
resolution, or end-to-end runs. That division keeps the project
maintainable: Spring is the host; the compiler pipeline is the product.

## Inputs and outputs

Input: a path to a .critx file. Internally the Critx file is treated as
JSON with a root node that includes sections, a template subtree, and an
\'apparatuses\' array.

Output: a .tex file written to the chosen compilation directory, and
(after one or more XeLaTeX runs) a .pdf file in the same directory.

# Repository layout and package map

The code mirrors a standard Maven or Gradle source layout, but without
build files:

\- java/... corresponds to src/main/java\
- resources/... corresponds to src/main/resources

All application code lives under the root package
\'com.example.printpreviewnav\'. The packages are organized by feature
(loader, navigation, apparatus, paratextual, latex, language,
utilities). That structure matches how you reason about the pipeline and
keeps responsibilities crisp.

## Package guide (what goes where)

The repository snapshot follows the familiar Maven/Gradle "main sources"
convention. Practically, that means you can read it as if it were a
standard Spring Boot project. In this codebase, resources/ currently
contains application.properties, which is consistent with the project's
role as a non-web Spring Boot application: configuration is still loaded
the usual Boot way, but the actual "work" is almost entirely plain Java.

All application code lives under the root package
com.example.printpreviewnav. The important design choice is that
packages are organized by feature and pipeline phase rather than by
technical layer. That keeps responsibilities crisp: parsing lives
together, traversal lives together, LaTeX emission lives together, and
so on. When you're debugging a PDF issue, you can usually jump straight
to the phase that "owns" the symptom rather than hunting through generic
"service/repository/controller" folders that don't exist (and aren't
useful here anyway). A simple dependency rule helps you keep the
structure clean: loader/model/navigation/apparatus/paratextual are the
front-end pipeline (parse, normalize, compute), latex/language are the
rendering front-end + output encoding, and utilities is strictly support
code. If a class feels like it belongs to two packages, that's a smell:
it usually means the class is doing two jobs and should be split.

**Package guide (what goes where, and why)**

- com.example.printpreviewnav: This is intentionally tiny: it contains
  the entry point (PrintPreviewNavApplication) and should stay that way.
  The point of keeping the root package simple is to make it obvious
  where real logic should not go. Any parsing, rendering, or domain
  rules placed here will quickly become untestable glue.

- com.example.printpreviewnav.loader: This is the document front door.
  It is responsible for turning a .critx JSON payload into the internal
  "node tree" representation that the rest of the pipeline understands.
  In your code this includes the Node model and its supporting enums
  (NodeType, InlineMarkType, SectionType, etc.), plus collectors that
  extract text runs and paragraph material in a structured way
  (TextCollector, ParagraphCollector, HeadingTextCollector,
  ListParagraphTextNodes).

- com.example.printpreviewnav.model.corestructure: This is the core
  domain model: the "stable internal representation" that should not
  care about JSON shapes or LaTeX syntax. In your code this includes
  DocumentModel and Section, plus typed attribute objects such as
  HeadingAttributes, ParagraphAttributes, and TextStyleAttributes, and
  CLI-ish options like ArgsOptions. The value of this package is
  stability: changes in Critx input format should mostly be absorbed in
  loader; changes in LaTeX layout should mostly be absorbed in
  latex/paratextual. This core model should change slowly, because many
  parts of the pipeline depend on it.

- com.example.printpreviewnav.navigation: This package answers one
  question: "given a document, how do we traverse it predictably?" It
  provides cursor-like, deterministic navigation across the structure
  (DocumentNavigator, SectionNavigator). Keeping traversal separate is a
  quiet win: it prevents every rendering or extraction routine from
  reinventing its own traversal rules, which is how you end up with
  mismatched ordering bugs ("the TOC sees sections in one order, the
  renderer in another").

- com.example.printpreviewnav.apparatus and
  com.example.printpreviewnav.apparatus.model: This is the apparatus
  subsystem: parsing apparatus definitions, indexing entries, computing
  overlaps/spans, and building the data that will later be rendered into
  LaTeX apparatus constructs. Your classes here (ApparatusParser,
  ApparatusDataExtractor, ApparatusOverlapFinder, ApparatusOverlaySpan,
  ApparatusNavigator, ApparatusStore and related model types such as
  ApparatusEntry, ApparatusElement, StyledSpan, TextFragment, etc.)
  clearly separate three responsibilities:

  - read/parse apparatus material into typed objects,

  - compute relationships (ordering, overlap, spans, sequencing),

  - expose a store/navigation API that rendering can consume without
    re-deriving rules.

> If you need to change how notes are grouped, sequenced, or merged,
> this is where you want to be. If you need to change how a note looks
> on the page, that belongs in latex.

- com.example.printpreviewnav.comments (plus subpackages): This package
  is easy to miss, but it matters because it models a second "overlay"
  stream alongside apparatus: annotations/comments. In your code
  (Annotations, AnnotationsLoader, Comment, and the AnnotationTarget
  enum) it reads and represents comment-like metadata that can be
  attached to parts of the document. The practical guideline: apparatus
  is scholarly-critical content with layout rules; comments are
  annotation content with a different targeting model. Keeping them
  separate prevents conceptual blur and avoids a single overloaded
  "note" concept that becomes impossible to evolve.

- com.example.printpreviewnav.paratextual and
  com.example.printpreviewnav.paratextual.model: This is where
  "rendering policy" is parsed and stored: templates, page setup,
  document metadata, references format, sorting rules, and other
  page-level or book-level settings that control LaTeX generation
  without being part of the base text. In your code this shows up as
  parsers (TemplateParser, PageSetupParser, MetadataParser,
  ReferecensFormatParser, SortParser) and the corresponding model
  holders (Template, plus whatever structured data those parsers
  populate). A useful boundary: if the information describes "how the
  edition should look" rather than "what the edition says", it belongs
  here.

- com.example.printpreviewnav.latex: This is the LaTeX emission layer:
  it takes the normalized internal model plus computed overlays
  (apparatus/comments) plus paratextual settings and produces LaTeX
  content units. Your code makes this explicit with LatexContent
  implementations (ChapterLatexContent, HeaderFooterLatexContent,
  PageSetupLatexContent, FontsLatexContent, StylesLatexContent,
  TocLatexContent, ApparatusLatexContent, AnnotationsLatexContent, etc.)
  and the node-level renderers that do the heavy lifting for text blocks
  (ProcessParagraph, ProcessList), producing structured results
  (ParagraphResult, ListResult). The discipline here is important: this
  package should not "figure out" semantics. It should assume that
  parsing/normalization/overlay computation already happened, then focus
  on deterministic LaTeX assembly.

- com.example.printpreviewnav.language (and its subpackages): This is
  the encoding and language-awareness layer that sits right at the
  boundary between your internal Unicode-rich model and LaTeX's
  expectations. In your code it covers detection/segmentation
  (DetectLanguage, UnicodeLanguageClassifier, LanguageRunSegmenter),
  mapping to Polyglossia (PolyglossiaMapper), font
  selection/configuration (FontFamilyConfig), and safe output
  (LatexEscaper, SymbolTable). If you ever see LaTeX break because of
  special characters, mixed scripts, or language switches, the fix
  should land here rather than being patched ad-hoc inside
  ProcessParagraph.

- com.example.printpreviewnav.utilities (and utilities.print): This is
  cross-cutting support code that should not own business rules. In your
  project it includes small mappers and finders (ApparatusSeriesMapper,
  CritxColorFinder, CritxFontFinder), shared text helpers
  (TextUtilities, TextStyleUtilities), OS/process support
  (SystemUtilities), and a set of debugging/diagnostic printers under
  utilities.print (TemplatePrinter, MetadataPrinter,
  ReferencesFormatPrinter, RawOverlapPrinter, etc.). The "utilities"
  trap is real: it becomes a junk drawer. Our current split between core
  helpers and explicit print/debug tooling is a good guardrail. A good
  rule is: if a utility starts depending on multiple feat ure packages
  (loader + apparatus + latex), consider moving it into the feature that
  truly owns the behavior, or split it into smaller feature-local
  helpers.

# Execution flow in one pass

PrintPreviewNavApplication.main is the pipeline entrypoint and the only
place where the end-to-end build is sequenced. It constructs the runtime
context, resolves input/output locations, and then executes the build in
a single linear pass. No phase runs "in the background", and no phase
mutates global state outside the build directory created for the current
run. The implementation is intentionally imperative: each phase produces
explicit artifacts that become inputs to the next phase.

Phase A --- Load (Critx → DocumentModel)

The first phase reads the .critx JSON file and turns it into the
in-memory document tree used throughout the build. This step establishes
structural correctness early: the parser maps JSON nodes into domain
objects (document, sections, paragraphs, inline spans, note anchors) and
attaches document-level configuration blocks (layout, rendering flags,
apparatus configuration). Load does not perform rendering or LaTeX
formatting; it only builds a complete DocumentModel with the minimum
normalization required for downstream logic to operate on stable
structures (for example, ensuring collections are non-null, assigning
IDs if missing, and resolving basic type variants into one
representation). Outputs:

- DocumentModel (root object + node tree)

- Document metadata (input path, build id, document id/version if
  present)

Phase B --- Precompute (DocumentModel → render-ready model)

Precompute builds the derived state required to render
deterministically. The central product is the ApparatusStore: an indexed
structure that allows the renderer to answer "given this anchor in the
base text, what apparatus entry/entries exist, in what order, with what
numbering, and on which apparatus stream?". This phase also computes
overlay relationships such as overlaps/collisions between anchors and
notes (for example, multiple anchors mapping to the same entry, or
entries spanning ranges depending on the input model). Precompute is
also where rendering configuration is frozen: template selection, LaTeX
feature flags, style mappings, and any required resource resolution
(fonts, LaTeX class/style files, template fragments). The key constraint
is that after Phase B the renderer does not need to "decide"; it only
needs to emit. Outputs:

- ApparatusStore (indices, ordering, numbering, grouping)

- Resolved rendering settings (template id/path, style config, LaTeX
  flags)

- Resolved resource set (fonts/templates/assets copied or referenced
  into build directory)

- Validation failures surfaced as hard errors before rendering

Phase C --- Render (render-ready model → LaTeX source)

Render generates a complete, standalone LaTeX document. "Complete" means
the output contains the preamble (packages/macros), document body, and
all stream outputs (main text plus apparatus streams) in a single .tex
file. Rendering is deterministic: for the same DocumentModel and
settings it generates stable output ordering and stable formatting. The
renderer consumes the DocumentModel for structure and the ApparatusStore
for apparatus emission; it does not recompute apparatus content from
scratch, and it does not emit apparatus bodies inline inside paragraph
rendering. Paragraph processing converts text and inline elements into
LaTeX-safe output (escaping, macro usage, style application), emitting
only anchors where apparatus references occur. Outputs:

- \<buildDir\>/document.tex (and optionally included .tex fragments)

Phase D --- Compile (LaTeX → PDF)

Compile runs XeLaTeX as an external process in the build directory. The
application executes XeLaTeX multiple times to reach a stable layout
state, because LaTeX resolves cross-references, page breaks, footnote
placement, and other layout-dependent constructs across passes. The
build uses a fixed pass policy: 1, 2, or 4 runs depending on what
features are enabled (for example, if the template activates constructs
that require additional stabilization passes). Each run is executed with
an explicit output directory and flags that stop on errors and produce
consistent logs (for example, -halt-on-error and a non-interactive
mode). The compile wrapper captures exit codes and persists
stdout/stderr and the .log file. Outputs:

- \<buildDir\>/document.pdf

- \<buildDir\>/document.log and auxiliary files (.aux, .toc, etc.)

- Structured compile result (success/failure, exit code, key error
  excerpt)

Phase E --- Report (verify artifacts + diagnostics)

The final phase verifies that the expected PDF exists and is non-empty,
then reports success with absolute output paths. On failure, it reports
a single error summary that includes the resolved XeLaTeX path, the
LaTeX source path, the log path, and the first meaningful LaTeX error
block. This phase does not attempt recovery. The output directory is the
primary diagnostic artifact: it contains the generated .tex and the
compiler log needed to reproduce the failure outside the application.

This orchestration level matches the architecture: one entrypoint
defines the build pipeline and delegates each phase to focused
components (parser/model builder, apparatus precomputation, LaTeX
renderer, XeLaTeX runner). The separation is strict: parsing does not
render, rendering does not compute apparatus, compilation does not
interpret the Critx model.

# Core model and traversal: Node, DocumentModel, SectionNavigator

The rendering pipeline relies on a strict invariant: before any
traversal or LaTeX emission happens, the Critx JSON is mapped into a
normalized object graph with a small, fixed vocabulary. That invariant
removes "JSON-shape awareness" from the rest of the codebase. Renderers
and navigators do not branch on arbitrary JSON fields; they branch on
NodeType and on typed attributes that exist only when the NodeType
requires them. This is the reason the code stays testable: unit tests
build a small Node tree and run rendering logic without any JSON
parsing, and integration tests validate the loader separately.

## Node: the normalized building block

loader.Node is the canonical representation of a document element after
loading. The loader translates raw JSON "type" strings into a NodeType
enum and keeps the original type string (typeName) for diagnostics and
forward compatibility. The Node contains a generic attrs object
(ObjectNode) for raw attributes and typed attribute views for the node
types that have stable, renderer-relevant semantics. The fields
renderers actually consume are the normalized kind, children, and
content payload:

- typeName preserves the original JSON type token. It exists for
  debugging and for "unknown type" handling; it is not used for
  rendering decisions.

- nodeType is the only discriminator used in traversal/render logic.
  NodeType collapses many JSON variants into a stable set the code
  understands.

- attrs is the raw attribute bag. It is allowed to contain nulls and
  unrecognized keys. The loader does not force every attribute into a
  typed model; it types only the attributes that affect rendering and
  navigation rules.

- content is the ordered list of child nodes. Ordering is part of
  semantics: DFS traversal and emitted LaTeX depend on it.

- text is present for text nodes and is the payload emitted into LaTeX
  (after escaping/normalization in the text utilities).

- headingAttrs and paragraphAttrs are pre-parsed typed views. They exist
  only for HEADING and PARAGRAPH nodes; renderers read these instead of
  repeatedly inspecting raw attrs keys.

- marks is the list of inline marks for TEXT nodes (bold, italic,
  smallcaps, etc.). Marks are normalized and ordered so that the inline
  renderer can apply them deterministically.

  ----------------------------------------------------------------------
  private final String typeName; // original \"type\"\
  private final NodeType nodeType; // normalized kind\
  private final ObjectNode attrs; // raw attrs (can contain nulls)\
  private final List\<Node\> content; // children\
  private final String text; // for text nodes\
  \
  private final HeadingAttributes headingAttrs; // only for HEADING\
  private final ParagraphAttributes paragraphAttrs; // only for
  PARAGRAPH\
  private final List\<InlineMark\> marks; // only for TEXT (can be
  empty)
  ----------------------------------------------------------------------

  ----------------------------------------------------------------------

A concrete rule enforced by this model: renderers do not read raw attrs
directly unless the attribute is truly "format-specific" or not worth
stabilizing. For the common path (paragraph/heading/text), renderers
rely on typed attributes and NodeType. That concentrates schema
interpretation in the loader and avoids duplicating JSON parsing logic
across LaTeX emitters.

## DocumentModel and Section

model.corestructure.DocumentModel partitions the document into four
logical sections: TOC, INTRODUCTION, MAIN, and BIBLIOGRAPHY. This is not
a presentation detail; it is a structural constraint used by rendering
rules. LaTeX emission differs by section: the TOC has its own structure
and macros, bibliographic content has different environments, and the
main body is the only section that carries most apparatus integration.
model.corestructure.Section is a thin wrapper around:

- a root Node (the subtree to render)

- section metadata (section type enum plus any section-level settings
  used by navigation/render decisions)

DocumentNavigator is the "selection adapter": given a section enum, it
returns the corresponding Section (or an empty/absent result if the
section is missing). This keeps section choice out of renderers: a
renderer receives a Section (or a Section root Node) and proceeds
identically regardless of how the Section was picked.

## SectionNavigator: cursor semantics you can rely on

navigation.SectionNavigator converts a Section's Node tree into a
deterministic depth-first sequence of NodeHandle objects. NodeHandle
exists because traversal often needs more than "the node": it needs
context such as parent linkage, depth, index within parent, and possibly
computed properties used by render rules. Flattening once into dfs\[\]
has two measurable effects:

- traversal logic becomes constant-time cursor operations
  (first/next/previous/current)

- renderer code becomes structural and predictable: it reads a stream of
  handles and emits output without re-implementing tree walking

The cursor semantics are explicit and stable. idx is allowed to take
sentinel values:

idx == -1 means "before first"

idx in \[0..size-1\] means "valid current element"

idx == size means "after last"

The methods enforce those states:

- first() sets idx = 0 and returns dfs\[0\] (or empty for an empty
  section).

- last() sets idx = size-1 and returns dfs\[size-1\] (or empty for an
  empty section).

- current() returns dfs\[idx\] only when idx is in range.

- next() increments idx when possible; otherwise sets idx = size
  (after-last) and returns empty.

- previous() decrements idx when possible; otherwise sets idx = -1
  (before-first) and returns empty.

This matters because renderers rely on "current is empty" to mean
"cursor is outside the stream" rather than "there is a node but it has
no content". That distinction avoids off-by-one bugs when renderers do
lookahead or when they need to stop precisely at section boundaries.

  ----------------------------------------------------------------------
  public Optional\<NodeHandle\> first() {\
  if (dfs.isEmpty()) { idx = -1; return Optional.empty(); }\
  idx = 0;\
  \
  if (log.isTraceEnabled()) log.trace(\"Move FIRST -\> idx={}\", idx);\
  \
  return Optional.of(dfs.get(idx));\
  }\
  \
  public Optional\<NodeHandle\> last() {\
  if (dfs.isEmpty()) { idx = -1; return Optional.empty(); }\
  idx = dfs.size() - 1;\
  \
  if (log.isTraceEnabled()) log.trace(\"Move LAST -\> idx={}\", idx);\
  \
  return Optional.of(dfs.get(idx));\
  }\
  \
  public Optional\<NodeHandle\> current() {\
  if (idx \< 0 \|\| idx \>= dfs.size()) return Optional.empty();\
  return Optional.of(dfs.get(idx));\
  }\
  \
  public Optional\<NodeHandle\> next() {\
  if (idx + 1 \< dfs.size()) {\
  idx++;\
  if (log.isTraceEnabled()) log.trace(\"Move NEXT -\> idx={}\", idx);\
  return Optional.of(dfs.get(idx));\
  }\
  idx = dfs.size(); // move \"after last\"\
  if (log.isTraceEnabled()) log.trace(\"Move NEXT -\> after-last
  (idx={})\", idx);\
  return Optional.empty();\
  }\
  \
  public Optional\<NodeHandle\> previous() {\
  if (idx - 1 \>= 0) {\
  idx\--;\
  if (log.isTraceEnabled()) log.trace(\"Move PREVIOUS -\> idx={}\",
  idx);\
  return Optional.of(dfs.get(idx));\
  }\
  idx = -1; // move \"before first\"\
  if (log.isTraceEnabled()) log.trace(\"Move PREVIOUS -\> before-first
  (idx={})\", idx);\
  return Optional.empty();\
  }\
  \
  // \-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\--\
  // Typed / predicate seeks\
  // \-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\--\
  \
  public Optional\<NodeHandle\> nextOfType(NodeType t) {\
  int i = idx + 1;\
  for (; i \< dfs.size(); i++) {\
  if (dfs.get(i).node().nodeType() == t) {\
  idx = i;\
  if (log.isTraceEnabled()) log.trace(\"Seek nextOfType({}) -\>
  idx={}\", t, idx);\
  return Optional.of(dfs.get(i));\
  }\
  }\
  idx = dfs.size();\
  if (log.isTraceEnabled()) log.trace(\"Seek nextOfType({}) -\> not
  found; idx=after-last\", t);\
  return Optional.empty();\
  }\
  \
  public Optional\<NodeHandle\> previousOfType(NodeType t) {\
  int i = idx - 1;\
  return Optional.empty();\
  }
  ----------------------------------------------------------------------

  ----------------------------------------------------------------------

After Node normalization and DFS flattening, the LaTeX side of the
codebase treats the document as an ordered stream of stable node kinds.
Paragraph rendering code does not parse JSON. Apparatus emission code
does not navigate arbitrary trees. Everything operates on (NodeType +
typed attrs + ordered handles). That separation is the reason the "core
model" acts as an API boundary: loader correctness is validated once,
and traversal/render correctness is validated on deterministic inputs.

# Apparatus subsystem: parse, index, extract, and overlay

The apparatus subsystem is the point where PrintPreviewNav stops being
"a LaTeX renderer" and becomes "a critical edition renderer". It has two
explicit outputs that share the same data source: (1) LaTeX emission
(critical apparatus streams and inline notes) and (2)
preview/diagnostics features (span annotations and overlap detection).
Every downstream step reads from that store; no downstream step
re-parses JSON or walks raw JsonNode trees.

## ApparatusParser: build the store

ApparatusParser reads root.apparatuses and builds an ApparatusStore. The
store has two primary indices:

- apparatusById: Map\<String, Apparatus\>: Keyed by apparatus id. This
  is the top-level object used by stream-level rendering ("emit
  apparatus A section", "emit apparatus B section"). It is also used to
  access apparatus-level configuration such as type, ordering,
  formatting, and grouping behavior.

- entryById: Map\<String, ApparatusEntry\>: Keyed by entry id. This is
  the hot path for rendering and for preview overlays: when the
  paragraph pipeline encounters an anchor marker or reference token, it
  resolves that token into an ApparatusEntry in O(1) and then renders or
  annotates it without scanning any list.

The parse method is intentionally small and mechanical because it
defines the boundary between "untyped JSON" and "typed domain objects".
It performs three tasks only: locate the apparatuses array, iterate it,
and delegate parsing of each apparatus while populating the entry index
as a side effect.

Key properties of the implementation:

- Single pass, eager indexing: The code walks the JSON array once and
  constructs both indices while parsing. Index construction happens
  during parsing, not after. That keeps the "store built" state
  internally consistent: if an Apparatus exists in apparatusById, all of
  its entries that have ids are already registered in entryById.

- Stable runtime characteristics: Rendering and preview logic rely on
  constant-time lookups. This design removes the need for repeated "find
  entry by id" scans across apparatus collections, which becomes
  expensive as documents grow.

The core loop is the indexing step. parseApparatus(appNode, entryById)
parses the apparatus object and registers every parsed ApparatusEntry
into entryById as it goes. The parse method then registers the returned
Apparatus into apparatusById keyed by apparatus.getId().

  ----------------------------------------------------------------------
  public ApparatusStore parse(JsonNode root) {\
  Map\<String, Apparatus\> apparatusById = new HashMap\<\>();\
  Map\<String, ApparatusEntry\> entryById = new HashMap\<\>();\
  \
  JsonNode arr = root.path(\"apparatuses\");\
  \
  if (log.isDebugEnabled()) {\
  log.debug(\"parse: root.apparatuses isArray={} sizeHint={}\",\
  arr.isArray(), arr.isArray() ? arr.size() : -1);\
  }\
  \
  if (arr.isArray()) {\
  for (JsonNode appNode : arr) {\
  Apparatus apparatus = parseApparatus(appNode, entryById);\
  if (apparatus != null) {\
  apparatusById.put(apparatus.getId(), apparatus);\
  }\
  }\
  }\
  \
  if (log.isInfoEnabled()) {\
  log.info(\"parse: built store apparatusCount={} entryCount={}\",\
  apparatusById.size(), entryById.size());\
  }\
  \
  return new ApparatusStore(apparatusById, entryById);\
  }
  ----------------------------------------------------------------------

  ----------------------------------------------------------------------

Index correctness is the enabling constraint for the rest of the
subsystem. Once ApparatusStore exists, "extract and overlay" becomes a
pure read operation: paragraph processing resolves anchors into
ApparatusEntry instances from entryById; apparatus stream rendering
iterates Apparatus instances from apparatusById; preview tooling uses
the same entries plus the anchor positions computed during paragraph
processing to build annotated spans and detect overlapping ranges. The
subsystem stays deterministic because the parse step produces a
canonical store and every later phase reads from that canonical
structure.

## ApparatusDataExtractor: deterministic segments

ApparatusDataExtractor converts a raw apparatus entry into a linear,
ordered list of "segments" that the renderer can emit without additional
decisions. A segment is the smallest renderable unit in the apparatus
stream: plain text, a formatted span, a witness list, a reference token,
a separator, or an inline command marker. The output of
ApparatusDataExtractor is deterministic: given the same apparatus entry
and the same configuration, it produces the same segment sequence in the
same order. That determinism is enforced here so ProcessParagraph
remains a paragraph renderer, not a second apparatus formatter.

The extractor performs three operations in a fixed order.

- First, it selects the relevant fields from the apparatus entry and
  resolves all references needed for rendering. Resolution includes
  mapping symbolic fields into concrete render values: normalized
  witness identifiers, resolved bibliography/authority references,
  normalized punctuation tokens, and configuration-driven labels (for
  example apparatus stream name, note prefix/suffix, or numbering
  style). Resolution produces a fully-expanded intermediate structure
  with no deferred lookups.

<!-- -->

- Second, it linearizes that intermediate structure into segments using
  a stable ordering rule set. Ordering rules are explicit and local to
  this class. Examples of stable rules:

  - segments are emitted in a fixed block order: opening label → lemma →
    separator → readings/variants → witnesses → closing punctuation
    (exact ordering depends on the apparatus type, but it is encoded as
    a single recipe here)

  - collections are sorted using a stable key (for example normalized
    witness siglum, then original order as a tiebreaker)

  - optional fields emit either a specific segment sequence or nothing,
    never a partially emitted structure

<!-- -->

- Third, it applies formatting at the segment level. Formatting is
  encoded as segment attributes, not as string concatenation. For
  example, italicization, small caps, or LaTeX macro wrapping is
  represented as flags or wrapper tokens on the segment, not by
  injecting raw LaTeX into the content. The LaTeX renderer becomes a
  direct mapping: each segment type maps to a LaTeX emission strategy.
  That keeps LaTeX syntax decisions in the renderer layer, while
  ApparatusDataExtractor owns content ordering and segmentation.

This class exists to remove two failure modes from ProcessParagraph.

- The first is duplicated logic. When ProcessParagraph formats apparatus
  inline, the same ordering rules get reimplemented in multiple places:
  one code path for paragraph notes, another for footnote notes, another
  for apparatus block notes. ApparatusDataExtractor centralizes the
  recipe and returns the exact same segment stream to any caller.

- The second is nondeterministic output. If ProcessParagraph iterates
  over maps/sets, merges lists opportunistically, or formats
  conditionally based on incidental state, the LaTeX output changes
  between runs. ApparatusDataExtractor eliminates incidental ordering by
  sorting collections explicitly, emitting fixed separators, and
  representing the result as a segment list whose order is the canonical
  order for that apparatus entry.

The contract is simple: ProcessParagraph passes a normalized apparatus
entry and receives a List\<Segment\>. ProcessParagraph does not reorder
segments, does not insert punctuation, and does not "help" by formatting
witness lists. It only decides placement (where the apparatus anchor
appears in the paragraph) and delegates content construction to
ApparatusDataExtractor. The renderer consumes the segments and emits
LaTeX with a pure mapping from Segment type to LaTeX tokens.

## ApparatusSeriesMapper: stable mapping for reledmac

reledmac models multiple apparatus streams as "series", identified by
letters: A, B, C... The LaTeX side does not care about your internal
apparatus IDs; it only sees series letters, and it assumes those letters
remain consistent within a document build. ApparatusSeriesMapper is the
adapter between the application's apparatus registry (ApparatusStore)
and reledmac's series-letter convention.

The constructor implements a deterministic mapping rule: it extracts
only apparatus entries whose type is CRITICAL, preserves their insertion
order as returned by store.getAllApparatus(), and assigns letters
sequentially starting from \'A\'. The stability property comes from two
constraints: (1) the store enumeration order is stable for a given input
document, and (2) the mapping uses only that order plus the fixed \'A\'
offset. There is no hashing, sorting, or iteration over a non-ordered
collection, so the same Critx input produces the same series-letter
assignment every time.

The mapper enforces the reledmac series limit explicitly. Letters run
from \'A\' to \'Z\', so the maximum supported CRITICAL apparatus count
is 26. The constructor logs a document-level summary (INFO) indicating
how many CRITICAL apparatuses are being mapped and the maximum supported
size. If the count exceeds 26, the code fails fast with an
IllegalStateException. This prevents silent truncation or undefined
LaTeX behavior later in the pipeline, where errors become harder to
trace back to their real cause.

A subtle failure mode in this area is duplicate apparatus IDs. The map
apparatusToLetter uses the apparatus id as its key. If two apparatus
objects share the same id, the later one overwrites the earlier entry,
changing the effective series assignment while leaving the input list
length unchanged. The constructor does not change the behavior (it still
overwrites), but it surfaces the problem via a WARN log that includes
the id, the previously assigned letter, and the new letter. That log
line is operationally important: it pinpoints a data integrity issue
that otherwise manifests as "wrong apparatus stream" in the final PDF.

The loop itself is intentionally straightforward:

- critical.get(i) selects the i-th CRITICAL apparatus in insertion
  order.

- letter = (char)(\'A\' + i) assigns A to the first, B to the second,
  etc.

- apparatusToLetter.put(app.getId(), letter) stores the mapping.

- DEBUG logging provides per-apparatus traceability without polluting
  normal runs.

The public API is a resolver: getLetterForApparatus(String apparatusId)
returns an Optional\<Character\>. The Optional is a deliberate signal
that not every apparatus id maps to a series letter: only CRITICAL
apparatuses participate in the A..Z assignment. Non-critical apparatuses
(or unknown ids) produce Optional.empty(), which the renderer uses to
decide whether to emit reledmac series-specific macros or skip series
handling for that apparatus.

Technically, the core contract of ApparatusSeriesMapper is: "Given an
ApparatusStore with a stable iteration order, produce a stable, bounded,
observable mapping from CRITICAL apparatus IDs to reledmac series
letters." This contract isolates reledmac's letter-based requirement to
one class and keeps the rest of the rendering code working with domain
identifiers rather than LaTeX conventions.

  ----------------------------------------------------------------------
  public ApparatusSeriesMapper(ApparatusStore store) {\
  Objects.requireNonNull(store, \"store\");\
  \
  // Collect CRITICAL apparatuses in insertion order\
  List\<Apparatus\> critical = new ArrayList\<\>();\
  for (Apparatus a : store.getAllApparatus()) {\
  if (a != null && \"CRITICAL\".equalsIgnoreCase(a.getType())) {\
  critical.add(a);\
  }\
  }\
  \
  int n = critical.size();\
  int max = (\'Z\' - \'A\' + 1); // 26\
  \
  // INFO: document-level summary for business observability.\
  log.info(\"Assigning series letters to {} CRITICAL apparatus(es) (max
  supported={})\", n, max);\
  \
  if (n \> max) {\
  throw new IllegalStateException(\"Too many CRITICAL apparatus (max
  \" + max + \"), got \" + n);\
  }\
  \
  for (int i = 0; i \< n; i++) {\
  Apparatus app = critical.get(i);\
  char letter = (char) (\'A\' + i);\
  \
  // Because: duplicate ids would silently overwrite; surface it without
  changing behavior.\
  if (apparatusToLetter.containsKey(app.getId())) {\
  log.warn(\"Duplicate apparatus id \'{}\' encountered; previous series
  \'{}\' will be overwritten with \'{}\'\",\
  app.getId(), apparatusToLetter.get(app.getId()), letter);\
  }\
  \
  apparatusToLetter.put(app.getId(), letter);\
  if (log.isDebugEnabled()) {\
  log.debug(\"Mapped apparatus id=\'{}\' to series \'{}\'\",
  app.getId(), letter);\
  }\
  }\
  }\
  \
  /\*\*\
  \* Resolve the LaTeX series letter for a given apparatus id.\
  \*\
  \* \@param apparatusId id of the apparatus (may be {@code null})\
  \* \@return optional character letter ({@code \'A\'..\'Z\'}) if the
  apparatus was {@code CRITICAL} and mapped\
  \*/\
  public Optional\<Character\> getLetterForApparatus(String apparatusId)
  {
  ----------------------------------------------------------------------

  ----------------------------------------------------------------------

## Overlap analysis and preview overlays

Overlap analysis converts "a set of apparatus anchors in a paragraph"
into explicit regions where apparatus coverage intersects. The input is
a paragraph plus a set of apparatus ranges expressed in the paragraph's
coordinate space (typically character offsets or token indices after
paragraph normalization). ApparatusOverlapFinder performs a sweep over
these ranges and emits a sequence of non-overlapping segments where the
active apparatus set is constant. Each segment contains start/end
offsets plus the set of active notes (or note IDs) that cover that
segment. The output is deterministic: the same paragraph text and the
same normalized ranges produce the same segment list.

The algorithm behaves like an interval sweep. It builds boundary events
for every apparatus range (start, end), sorts boundaries by position,
then walks left-to-right maintaining an "active set" of notes. Between
boundary positions, the active set is stable; those spans are the
overlap regions. This avoids pairwise note comparisons and scales
linearly with the number of ranges once boundaries are sorted. It also
provides exact semantics for edge cases: adjacent ranges (end at N,
start at N) generate no overlap span, nested ranges produce increasing
and decreasing active-set sizes, and identical ranges collapse into a
span with multiple active notes covering the same coordinates.

The overlap regions then drive the preview overlay model.
ApparatusNavigator maps overlap segments to UI-level semantics: for each
span it resolves which ApparatusType(s) are active (A, B, etc.), applies
precedence and grouping rules, and builds a typed overlay structure
aligned with the paragraph text. This is the point where the system
stops being "note math" and becomes "preview output": the model no
longer talks about raw offsets only, it talks about annotated runs that
can be rendered as styled text.

AnnotatedRunsBuilder turns those typed spans into a compact sequence of
styled runs suitable for the preview layer. It emits "runs" that include
the text slice plus style attributes derived from apparatus types and
note metadata. Runs are coalesced aggressively: adjacent runs with
identical style signatures merge into one run. Coalescing happens per
ApparatusType so the preview stays stable: if two adjacent spans are
both covered by apparatus A with the same computed style, they become
one run; if A changes to B or to A+B, a new run starts. The result is a
minimal, render-friendly run list with no redundant fragmentation.

This overlap/overlay pipeline serves two outputs simultaneously.

For UX, it provides deterministic highlighting: apparatus coverage
becomes visible, multi-coverage becomes visually distinct, and
boundaries align exactly to normalized text positions. The preview can
highlight "Apparatus A only", "Apparatus B only", and "A+B overlap" as
separate styles because the overlap finder already computed the
segmentation.

For QA, it produces a measurable signal. The system can count overlap
spans, compute total characters under overlap, and detect pathological
patterns such as deep nesting or excessive density within a paragraph.
Those metrics are derived from the same overlap segments used for
rendering, so a validation step can assert invariants such as "no span
has more than N simultaneous notes" or "overlaps for apparatus A and B
exist only where allowed by configuration". Logging the overlap summary
for each paragraph makes apparatus integrity issues visible without
needing to inspect the final PDF.

# Paratextual subsystem: template, layout, metadata, and references

The paratextual subsystem defines the rendering contract between the
Critx document and the LaTeX generator. The Critx input embeds a
template subtree that encodes page geometry, layout blocks, style rules,
document metadata, reference formatting conventions, and ordering rules.
PrintPreviewNav treats that subtree as configuration, not content: it is
parsed once, validated once, and then used as immutable input to every
rendering decision.

This subsystem is implemented under
com.example.printpreviewnav.paratextual. Its core responsibility is
translation: convert JSON structures into a typed Template model that
the rest of the application consumes. The LaTeX writers never traverse
raw JSON. They take a Template instance and operate on strongly typed
fields (page setup, styles, layout definitions, metadata, and reference
formatting). This removes JSON-shape coupling from the renderers and
forces all schema assumptions into a single boundary.

Template is the aggregate root for paratext. It contains the resolved
page setup, style catalog, layout definitions for page regions/blocks,
metadata, reference formatting rules, and any sorting directives that
affect deterministic rendering (TOC ordering, apparatus entry ordering,
bibliography/reference ordering). Template is created once during
parsing and then passed through the pipeline as read-only configuration.

Key parsers and responsibilities

- TemplateParser parses root.template and builds the Template aggregate.
  It resolves the top-level template structure and delegates specialized
  parts to other parsers. It also performs structural validation:
  required sections exist, unknown keys are either rejected or mapped to
  an "extensions" bucket, and defaults are applied in a single place so
  downstream components never re-default fields.

- PageSetupParser parses the page geometry and layout constraints that
  directly affect LaTeX preamble generation and environments. This
  includes page size, margins, column configuration, header/footer
  structure, and any global switches that change the page model. Its
  output is a PageSetup object inside Template, consumed by LaTeX
  writers to emit geometry packages, column environments, and
  header/footer definitions. This parser is also the normalization point
  for units: the internal representation uses one unit system (for
  example, millimeters or TeX lengths), and the renderer converts that
  representation into LaTeX tokens without reinterpreting the raw JSON.

- MetadataParser parses document-level metadata that must be
  materialized into LaTeX once and then referenced by other generators.
  Examples include title, author/editor labels, edition identifiers,
  language, and any metadata used by PDF properties or the title page.
  Metadata is stored in Template as a typed structure (not a generic
  map) so writers can address fields directly and can treat absent
  values as explicit null/empty, not as missing JSON keys.

- ReferencesFormatParser (spelled ReferecensFormatParser in the current
  text) parses the formatting contract for references: numbering style,
  separators, prefix/suffix conventions, and any locale-dependent
  punctuation rules that must remain consistent across the whole
  document. The key design point is that reference formatting is not
  recomputed ad hoc. Writers ask the Template for a ReferencesFormat
  object and render references using its rules, so the apparatus
  subsystem and any bibliography/reference emitter stay consistent.

- SortParser parses ordering directives used to make rendering
  deterministic. Sorting rules can apply to references, apparatus
  entries, and paratextual blocks such as TOC. The parser produces a
  typed representation of sort keys and directions (for example, by
  appearance order, numeric order, custom label order). The renderers
  execute these rules when producing LaTeX so that the same input yields
  the same output ordering.

- ParatextualParser parses page elements and layout blocks that are not
  part of the main text flow but still appear in the rendered document:
  TOC structures, running headers, footer content, page-region blocks,
  and apparatus layout scaffolding. The result is a set of typed "layout
  block" objects stored in Template and consumed by LaTeX writers that
  emit the corresponding LaTeX environments and macros. This parser is
  the single point that understands how Critx encodes those blocks;
  writers operate only on the typed blocks.

The overall effect is a strict dependency direction: Critx JSON →
paratextual parsers → Template model → LaTeX writers. No other package
reads raw JSON to make layout decisions, and no renderer defines its own
interpretation of styles, page setup, metadata, reference formatting, or
sorting.

# Language subsystem: Unicode detection + Polyglossia

The language subsystem turns plain Java strings into renderable LaTeX
text with explicit language boundaries. The output target is
Polyglossia, so the renderer emits language switches as LaTeX macros
instead of assuming a single document language. The subsystem has four
responsibilities: segment text into language runs, map each run to
Polyglossia language commands, escape LaTeX-sensitive characters, then
apply styling on top of the escaped, language-tagged content.

UnicodeLanguageClassifier performs the segmentation. Its output is a
sequence of LanguageRun objects. A LanguageRun holds two things: the
literal substring and a language tag selected from a small internal
language set (for example LATIN, GREEK, ARABIC, HEBREW, CYRILLIC, or a
fallback like UNKNOWN). Segmentation is deterministic and based on
Unicode script ranges, not heuristics like dictionaries. The classifier
scans code points, assigns each code point to a script bucket, then
groups adjacent code points that resolve to the same language/script
into a single run. Whitespace and punctuation are either merged into the
surrounding run using a stable rule (attach to previous run except at
the beginning) or emitted as their own neutral run; the rule is fixed so
that two passes over the same text produce identical run boundaries.
This run list becomes the authoritative representation for "what
language is this fragment".

PolyglossiaMapper translates the run language tag into the LaTeX wrapper
used to switch language for that fragment. The mapping is one-to-one:
each internal language value maps to a Polyglossia language name (for
example latin → latin, greek → greek, arabic → arabic). The mapper emits
a wrapper pattern such as:\`\\text\<lang\>{\...}\`

The mapper does not attempt runtime inference beyond the classifier
output. If a run is UNKNOWN, it maps to a configured default language
wrapper or emits the content without a wrapper, depending on the
project's conventions. The selection is controlled by configuration so
the same input and configuration always yield the same LaTeX.

LatexEscaper ensures the run content is safe to embed inside LaTeX
macros. Escaping is applied to the raw substring before style or
language wrappers are applied, and it is applied exactly once. The
escaper replaces LaTeX special characters (such as \`\\\`, \`{\`, \`}\`,
\`%\`, \`\$\`, \`#\`, \`&\`, \`\_\`, \`\^\`, \`\~\`) with their
LaTeX-safe forms. It also normalizes line breaks into the project's
LaTeX linebreak strategy (either explicit \`\\\\\`, \`\\par\`, or
whitespace normalization), so the renderer never emits raw newlines
accidentally. XeLaTeX handles Unicode, so the escaper does not
transliterate; it only prevents LaTeX parsing errors.

TextStyleUtilities integrates these steps to produce the final LaTeX
fragment used by paragraph rendering. The method takes a string plus a
style context (font selection, italic/bold/smallcaps, color, etc.),
then:

1\. calls UnicodeLanguageClassifier to obtain LanguageRun segments

2\. escapes each run's text via LatexEscaper

3\. wraps each escaped run via PolyglossiaMapper

4\. applies the style wrapper(s) around the already language-wrapped
content (or in the exact order defined by the template contract)

The ordering matters because Polyglossia wrappers and style wrappers are
both macros. The subsystem fixes a single nesting order across the
codebase to avoid subtle formatting drift. The output is a single LaTeX
string that is simultaneously Unicode-safe, LaTeX-safe, and
language-explicit, and paragraph-level renderers consume it as an atomic
fragment instead of re-implementing escaping, language switching, or
styling locally.

#  LaTeX subsystem: document assembly and node-level rendering

The latex package implements two distinct responsibilities that stay
separate in code: document assembly and node-level rendering. Assembly
builds the LaTeX "frame" (preamble, macros, layout configuration,
structural wrappers). Node renderers generate LaTeX for the actual
document content (paragraphs, lists) and inject apparatus material at
the exact anchor points.

Assembly blocks

- The assembly layer is a set of small generators that take Template
  plus its related configuration models and return self-contained LaTeX
  fragments. Each block owns one concern and emits LaTeX that is valid
  in isolation. The naming reflects the output responsibility:

- FontsLatexContent emits font configuration and font-related macros. It
  translates the Template's font selections into XeLaTeX-compatible
  declarations and ensures font setup happens before any content is
  typeset.

- ColorsLatexContent emits color definitions and related macros. It
  defines the color palette used by the rest of the document and avoids
  inline color literals scattered through renderers.

- StylesLatexContent emits style macros and mappings from logical styles
  (for example "body", "heading", "apparatus") to concrete LaTeX
  commands. This is the central place where typographic intent becomes
  LaTeX vocabulary.

- HeaderFooterLatexContent emits page header/footer configuration. It
  materializes Template settings into package configuration and the
  macros that define running heads and footers.

- PageSetupLatexContent emits page geometry and layout primitives
  (margins, paper size, line spacing primitives that are global). It
  establishes the physical page constraints that every node renderer
  must respect implicitly.

- LineNumberLatexContent emits the LaTeX configuration for line
  numbering (when enabled). It defines the packages/macros used to
  number lines and aligns numbering configuration with the apparatus
  conventions used elsewhere.

- PageNotesLatexContent and SectionNotesLatexContent emit the apparatus
  infrastructure. They define the LaTeX environments, counters, and
  formatting rules used to place and render notes at the page level or
  section level, depending on the Template configuration. These blocks
  do not render note content; they declare how note streams behave and
  how note text is formatted once injected by node renderers.

- TocLatexContent emits table-of-contents support and TOC formatting
  rules. It binds Template TOC settings to LaTeX commands and ensures
  TOC generation is consistent with headings emitted by the
  orchestrator.

- ChapterLatexContent emits chapter/section structural wrappers. It
  defines the LaTeX macros/environments that represent higher-level
  structure so that headings and navigational constructs remain
  consistent across the document.

- StaticLatexContent emits fixed LaTeX that the project requires
  regardless of Template choices (common macros, package imports that
  are always needed, compatibility shims). Keeping this content
  centralized prevents duplication across blocks.

The assembly blocks operate as pure functions from models to strings (or
string-like fragments). They avoid traversal of document nodes. They
emit LaTeX for configuration and structure, not for content.

## LatexContent: orchestrate traversal and ordering

LatexContent is the rendering-side orchestrator. It owns traversal,
ordering, and stitching. It iterates the document structure via
SectionNavigator, emitting headings and block nodes in document order.
It then assembles the final LaTeX document by concatenating assembly
fragments and body fragments according to Template-defined ordering.
Template ordering is the contract for where each assembly block's output
appears relative to the document body (for example, fonts and static
macros in the preamble, header/footer setup before \\begin{document},
TOC at the start of the body, apparatus infrastructure before first
content, etc.).

This separation is enforced in code by responsibility boundaries, not by
convention: LatexContent controls "when" and "in what order";
ProcessParagraph and ProcessList control "how a node becomes LaTeX".
Assembly blocks control "which LaTeX configuration exists" for fonts,
layout, styles, and apparatus infrastructure. The result is a rendering
pipeline where traversal logic does not mix with low-level LaTeX
emission, and node renderers do not pull in global document assembly
concerns.

## ProcessParagraph: the core algorithm

ProcessParagraph.processParagraph renders a single paragraph subtree
into LaTeX. It performs two jobs at the same time: it emits the visible
text in order, and it injects apparatus markup at exact structural
boundaries. The main correctness constraint is LaTeX wrapper integrity.
For reledmac-style critical notes, wrappers such as \\edtext{\...}{\...}
must be properly nested at all times. Once an \\edtext opens, everything
it wraps must appear as a contiguous LaTeX fragment; the wrapper cannot
be broken by forced page breaks or by emitting an ending wrapper while
an inner wrapper remains active.

The method enforces this by treating "critical notes" as a structured
interval problem. Each child node boundary defines a set of critical ids
that apply to the span that follows. The algorithm walks the paragraph's
child nodes left-to-right and maintains a stack (critStack) of active
critical ids that have been opened in LaTeX but not yet closed. A second
set (crits) tracks ids that have been fully emitted as complete LaTeX
note constructs to prevent duplicate close rendering. Invariant:
critStack represents the current LaTeX nesting order (outer to inner).
At any point, the stack content is exactly the set of critical ids that
are currently "open" in the emitted LaTeX stream.

Opening and closing are encoded as two local functions, openCrit and
closeCrit.

openCrit performs a visibility gate. It loads apparatus metadata for a
crit id, resolves which apparatus stream it belongs to via
ApparatusDataExtractor.extractSequenced(\...), and checks the template
configuration (layoutTemplate.critical.findApparatusById(\...)) to
decide if this apparatus is visible. If visible and not already closed
(not in crits), it emits the LaTeX opening fragment and pushes the id
onto critStack:

sb.append(\"\\edtext{\");

critStack.addLast(critId);

This means "from now on, emitted text belongs to this \\edtext until
closeCrit is called for the same id".

closeCrit implements two different close modes depending on whether the
id was already "finalized" earlier. If crits contains the id, the method
only updates internal state (removes from critStack) because the LaTeX
close fragment has already been emitted. If crits does not contain the
id, it emits the closing fragment that completes the \\edtext by
providing lemma + footnote payload:

}{\\lemma{\<lemma\>}\<letter\>footnote{\<body\>}}

Then it removes the id from the stack and adds it to crits.
Operationally: an id transitions from "open" (in critStack) to "closed"
(in crits) exactly once with LaTeX output; subsequent structural unwinds
only manipulate the stack.

The boundary logic is the core of correctness. For each node boundary,
the method derives three sets: starts, ends, continuing. The inputs are
the critical ids applicable "here" (currCrit) and the critical ids
applicable "previously" (prevCrit). These sets are computed with pure
set operations:

starts = currCrit − prevCrit

ends = prevCrit − currCrit

continuing = prevCrit ∩ currCrit

These represent transitions at the boundary between (i-1) and i. The
algorithm then performs one additional pass to handle the only case that
breaks simple close-then-open ordering: split crossings.

Split crossings occur when an outer critical id ends at the boundary
while an inner critical id continues across the same boundary. In a
well-nested interval model this cannot happen, but the input model can
express it because ids are attached to arbitrary nodes rather than
derived from a single properly-nested annotation tree. In LaTeX terms,
it produces an illegal structure: you cannot close the outer \\edtext
while leaving an inner \\edtext still open, because the closing fragment
completes the brace structure of the outer wrapper. The code fixes this
by splitting the continuing inner across the boundary: close and
immediately reopen the continuing inner before closing the outer.

The implementation inspects the current stack order (critStack) from
inner to outer. For each continuing id s, it checks whether any ending
id e appears below s in the stack (meaning e wraps s). If so, s
"crosses" e at this boundary. The repair is:

closeCrit(s);

openCrit(s);

This forces a legal LaTeX structure by turning one continuous interval
into two adjacent intervals in the emitted stream. After split repair,
the algorithm performs deterministic wrapper transitions at the
boundary:

1\. Close ids that end at this boundary, inner-first: The fast path
closes only while the stack top is in ends. This enforces correct brace
nesting: the most recently opened wrapper must be the first one closed.
The fallback path handles the rare case where an id that must end is not
currently on top (the comment calls it "rare after splitting"). In that
case the code unwinds to the target id. The existence of this fallback
is pragmatic: it prevents hard failure on imperfectly structured inputs,
but it also defines a test target because it represents "non-ideal but
supported" input.

2\. Open ids that start at this boundary, outer-to-inner.: (That
ordering is implicit: by opening after closing, the stack represents the
new active set for the upcoming text fragment. The exact ordering among
multiple starts must be stable to keep output deterministic; the
implementation uses stack operations and iteration order to enforce
this.)

3\. Emit the text for the node: The renderer writes the node's textual
content once the wrapper state matches the ids that apply to that text
span. This is the point where forced breaks matter. A forced break
cannot be emitted while an \\edtext wrapper is open, because it splits
the wrapped fragment into multiple LaTeX segments. The class therefore
treats breaks as boundaries that trigger closure and re-open decisions
using the same transition mechanism. The constraint "must not be split
by forced page breaks" is enforced by making "break nodes" participate
in boundary computations: before a break, active wrappers are closed to
a legal state; after the break, wrappers that continue are reopened if
their ids still apply.

4\. Inject inline notes at the correct boundary: Inline notes are
collected separately from critical notes (inlineHere =
filterIdsByTypeNot(\...)). They are emitted at structural boundaries
rather than managed via critStack because they do not impose the same
nesting rules as \\edtext. The separation between "critical" and
"inline" is important: only CRITICAL ids participate in stack invariants
and split-crossing repair.

**Technical invariants that the algorithm enforces**

The emitted LaTeX stream maintains well-formed brace nesting for
\\edtext wrappers: every \\edtext{ has a matching }{\...}} emitted
exactly once per id. At any moment during traversal, critStack equals
the ordered nesting of currently-open critical ids in the emitted
output. A structural boundary never closes an outer wrapper while
keeping an inner wrapper open; when the input model demands it, the
algorithm splits continuing inners to restore a legal nesting order.

Testing best practice for this class

Tests target structure, not typography. Use minimal synthetic Critx
paragraph trees with known critical-id placements and produce LaTeX
output. Assertions operate on the generated LaTeX string.

1\. Overlap chains: Create overlaps that force split-crossing repair.
Example: A wraps B at start, then A ends while B continues. The expected
output contains two separate B \\edtext segments separated at the
boundary where A ends, and A closes without leaving B open.

2\. Nested overlaps: Create A overlaps B and B overlaps C with boundary
transitions in the middle of the chain. Assert that closings occur
inner-first and that the stack behavior produces stable ordering.

3\. Forced break inside a critical span: Place a hard break node inside
an active critical id. Assert that the output does not contain the break
inside an \\edtext{\...} segment. Concretely: parse the LaTeX and ensure
no break macro appears between an \\edtext{ and its corresponding close
fragment for the same id.

4\. Non-duplication of note bodies: For each crit id, assert that the
note body snippet appears once in the LaTeX output (in the close
fragment) and never appears inline in the wrapped text emission.

These tests lock down the class's real contract: well-formed LaTeX
structure under arbitrary node boundary transitions.

#  Frequently used classes (where most changes land)

Most changes in this codebase land in a small set of classes that sit on
the "hot path" from Critx input to rendered LaTeX. The fastest way to
become productive is to treat these classes as the stable seams of the
pipeline: text in, structure traversed, apparatus overlaid, LaTeX
emitted, PDF compiled. Each class below owns a specific seam. When a bug
report arrives, map the symptom to the seam and start there.

- ProcessParagraph: ProcessParagraph sits on the hottest path in the
  renderer: it takes a paragraph-like node plus its local configuration
  (Template/Style, language/style runs, and apparatus anchors) and turns
  it into the LaTeX fragment that represents "the readable text the user
  expects to see". In practice it is the seam where three concerns meet
  and conflicts show up:

<!-- -->

- Text correctness: it calls into TextUtilities to normalize and escape
  raw content before any LaTeX is emitted. If a paragraph contains lemma
  anchors, ProcessParagraph is the place where offsets/ranges are
  applied (directly or indirectly) and where "what the lemma refers to"
  can drift if normalization changes the string length or character
  positions.

- Styling scope: it coordinates TextStyleUtilities to wrap spans with
  the correct LaTeX commands and to ensure wrapper boundaries align with
  run boundaries. When styling "leaks" across words or the wrong
  font/language applies to part of a paragraph, the defect usually
  appears as incorrect wrapper nesting or missing closures produced
  during paragraph processing.

- Apparatus anchoring: it emits the inline anchor markers that connect
  the base text to apparatus entries stored in ApparatusStore. It does
  not emit apparatus content; it emits the stable anchor that the
  apparatus stream later resolves. When notes attach to the wrong place,
  duplicate anchors appear, or anchors go missing, ProcessParagraph is
  the first place to verify that the correct ids are being emitted at
  the correct point in the LaTeX output.

> When debugging paragraph-level issues, the fastest method is to
> capture the exact LaTeX fragment generated by ProcessParagraph for one
> problematic paragraph and inspect three things in order: the
> normalized text (TextUtilities effect), the wrapper nesting
> (TextStyleUtilities effect), and the presence/position of anchor
> markers (apparatus linkage).

- TextUtilities: TextUtilities is the low-level normalization layer used
  by renderers before they emit LaTeX. It contains operations that turn
  "raw editor text" into "render-safe text", and it is where many
  visually small but structurally important transformations happen.
  Typical responsibilities include whitespace normalization, escaping or
  sanitizing characters that break LaTeX, and the slicing logic used to
  extract lemma fragments from a paragraph (the part of the text that an
  apparatus entry refers to). When the PDF output shows wrong
  characters, broken punctuation, missing spaces, or a lemma points to
  the wrong substring, the defect is usually not in the renderer that
  prints LaTeX, but in TextUtilities producing the wrong normalized text
  or offsets.

- TextStyleUtilities: TextStyleUtilities is the translation layer from
  paratextual styling rules to concrete LaTeX wrappers. It consumes
  Template/Style definitions (and any run-level language/font markers
  coming from the document model) and emits the LaTeX "envelopes" that
  implement them: font switches, emphasis wrappers, language selection
  commands, and any macro-level decoration needed for consistent
  typography. When users report "the font changed unexpectedly",
  "italics bleed into the next word", "language-specific hyphenation
  looks wrong", or "a style toggle works in one section but not
  another", the error is typically in how TextStyleUtilities composes
  wrappers (missing closing braces, incorrect nesting, or a mismatch
  between style scope and run scope).

- Template and Style (paratextual.model): Template and Style are the
  typed contract between the editor-facing configuration and the
  renderer. They represent layout and formatting choices in a form that
  can be validated and applied deterministically. These classes matter
  because they sit at the boundary between "user configuration" and
  "render logic": changes in visual settings are expressed as changes to
  Template/Style instances, and the renderer reads those values to
  decide which LaTeX macros to emit and which layout branches to take.
  When a new formatting option is introduced, the work usually starts by
  extending Template/Style (new field, enum, or nested config object),
  then wiring it through TextStyleUtilities and the relevant renderers.
  When an existing option appears to be ignored, inspect how
  Template/Style is constructed from the Critx JSON and where the field
  is read during rendering.

- ApparatusStore, ApparatusEntry, ApparatusAnnotation: These classes
  form the apparatus lookup and rendering model. ApparatusEntry
  represents the content to be rendered (note text, lemma, references,
  metadata), ApparatusAnnotation represents the attachment of that entry
  to a specific point in the base text (anchor id, offset/range, scope),
  and ApparatusStore is the index that makes lookup deterministic and
  fast during rendering. When apparatus notes go missing, render under
  the wrong anchor, render with wrong numbering, or appear in the wrong
  apparatus stream, the defect usually resolves to one of two failures:
  store construction produced an incomplete or inconsistent index, or
  lookups use the wrong key (for example, mixing node ids vs annotation
  ids). Debugging is mechanical: verify that the Critx parsing phase
  produces the expected ApparatusEntry objects, then verify that
  ApparatusStore indexes them under the exact ids used later by
  paragraph processing and LaTeX emission.

- SectionNavigator: SectionNavigator defines traversal semantics over
  the document tree: what "next section" means, how nested sections are
  entered and exited, and which nodes count as renderable content versus
  structural containers. This class is the usual suspect when output is
  structurally wrong: skipped sections, duplicated paragraphs, wrong
  ordering, or apparatus attached to the wrong rendered location because
  traversal visited nodes in an unexpected order. The key mental model
  is that traversal is policy, not data: the tree might be correct, but
  traversal rules can still generate incorrect ordering. When
  investigating issues here, trace the traversal sequence for a minimal
  input and compare it to the expected logical reading order. If
  ordering changes based on configuration (for example, preview modes),
  ensure that mode-specific traversal logic remains centralized in
  SectionNavigator rather than duplicated across renderers.

- LatexContent: LatexContent orchestrates section-level rendering and
  stitches together the LaTeX document in the correct order. It is not a
  "renderer for one node"; it is the component that decides which major
  LaTeX blocks exist (preamble hooks, body structure, TOC placement,
  main text placement, apparatus placement) and in which sequence they
  appear. When entire sections are missing, the TOC appears in the wrong
  place, main text is emitted before required macro definitions, or
  apparatus streams appear duplicated at the document level,
  LatexContent is the entry point. Most fixes here are about
  orchestration: ensuring the renderer is invoked once per stream,
  ensuring section boundaries are respected, and ensuring
  mode/configuration switches affect orchestration in one place.

- PrintPreviewNavApplication: PrintPreviewNavApplication is the pipeline
  coordinator and the integration boundary with the external compiler.
  It wires together parsing, normalization, traversal, LaTeX generation,
  and XeLaTeX execution, and it owns the file-system contract: where
  inputs are read from, where intermediate artifacts are written, and
  where outputs land. Failures that mention missing files, wrong paths,
  OS-specific behavior, permissions, or inconsistent build directories
  are rooted here. It is also the correct place to enforce pipeline
  invariants at runtime: validate inputs early, fail fast with explicit
  diagnostics, log resolved toolchain paths, and persist the generated
  .tex and .log artifacts whenever compilation fails.

Taken together, these classes map directly to the pipeline seams:
TextUtilities and TextStyleUtilities are "text correctness and styling",
Template/Style is "configuration contract", apparatus classes are "note
overlay model", SectionNavigator is "tree traversal policy",
LatexContent is "document stitching", and PrintPreviewNavApplication is
"end-to-end execution + toolchain boundary".
