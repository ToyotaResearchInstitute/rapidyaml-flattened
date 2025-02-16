Most of the changes are from the giant Parser refactor described below. Before getting to that, a couple of other minor changes first.


### Fixes

- [#PR431](https://github.com/biojppm/rapidyaml/pull/431) - Emitter: prevent stack overflows when emitting malicious trees by providing a max tree depth for the emit visitor. This was done by adding an `EmitOptions` structure as an argument both to the emitter and to the emit functions, which is then forwarded to the emitter. This `EmitOptions` structure has a max tree depth setting with a default value of 64.
- [#PR431](https://github.com/biojppm/rapidyaml/pull/431) - Fix `_RYML_CB_ALLOC()` using `(T)` in parenthesis, making the macro unusable.


### New features

- [#PR431](https://github.com/biojppm/rapidyaml/pull/431) - append-emitting to existing containers in the `emitrs_` functions, suggested in [#345](https://github.com/biojppm/rapidyaml/issues/345). This was achieved by adding a `bool append=false` as the last parameter of these functions.
- [#PR431](https://github.com/biojppm/rapidyaml/pull/431) - add depth query methods:
  ```cpp
  Tree::depth_asc(id_type) const;   // O(log(num_tree_nodes)) get the depth of a node ascending (ie, from root to node)
  Tree::depth_desc(id_type) const;  // O(num_tree_nodes) get the depth of a node descending (ie, from node to deep-most leaf node)
  ConstNodeRef::depth_asc() const;  // likewise
  ConstNodeRef::depth_desc() const;
  NodeRef::depth_asc() const;
  NodeRef::depth_desc() const;
  ```
- [#PR432](https://github.com/biojppm/rapidyaml/pull/432) - Added a function to estimate the required tree capacity, based on yaml markup:
  ```cpp
  size_t estimate_tree_capacity(csubstr); // estimate number of nodes resulting from yaml
  ```


------
All other changes come from [#PR414](https://github.com/biojppm/rapidyaml/pull/414).

### Parser refactor

The parser was completely refactored ([#PR414](https://github.com/biojppm/rapidyaml/pull/414)). This was a large and hard job carried out over several months, but it brings important improvements.

- The new parser is an event-based parser, based on an event dispatcher engine. This engine is templated on event handler, where each event is a function call, which spares branches on the event handler. The parsing code was fully rewritten, and is now much more simple (albeit longer), and much easier to work with and fix.
- YAML standard-conformance was improved significantly. Along with many smaller fixes and additions, (too many to list here), the main changes are the following:
  - The parser engine can now successfully parse container keys, emitting all the events in correctly, **but** as before, the ryml tree cannot accomodate these (and this constraint is no longer enforced by the parser, but instead by `EventHandlerTree`). For an example of a handler which can accomodate key containers, see the one which is used for the test suite at `test/test_suite/test_suite_event_handler.hpp`
  - Anchor keys can now be terminated with colon (eg, `&anchor: key: val`), as dictated by the standard.
- The parser engine can now be used to create native trees in other programming languages, or in cases where the user *must* have container keys.
- Performance of both parsing and emitting improved significantly; see some figures below.


### Strict JSON parser

- A strict JSON parser was added. Use the `parse_json_...()` family of functions to parse json in stricter mode (and faster) than flow-style YAML.


### YAML style preserved while parsing

- The YAML style information is now fully preserved through parsing/emitting round trips. This was made possible because the event model of the new parsing engine now incorporates style varieties. So, for example:
  - a scalar parsed from a plain/single-quoted/double-quoted/block-literal/block-folded scalar will be emitted always using its original style in the YAML source
  - a container parsed in block-style will always be emitted in block-style
  - a container parsed in flow-style will always be emitted in flow-style
  Because of this, the style of YAML emitted by ryml changes from previous releases.
- Scalar filtering was improved and is now done directly in the source being parsed (which may be in place or in the arena), except in the cases where the scalar expands and does not fit its initial range, in which case the scalar is filtered out of place to the tree's arena.
  - Filtering can now be disabled while parsing, to ensure a fully-readonly parse (but this feature is still experimental and somewhat untested, given the scope of the rewrite work).
  - The parser now offers methods to filter scalars in place or out of place.
- Style flags were added to `NodeType_e`:
  ```cpp
    FLOW_SL     ///< mark container with single-line flow style (seqs as '[val1,val2], maps as '{key: val,key2: val2}')
    FLOW_ML     ///< mark container with multi-line flow style (seqs as '[\n  val1,\n  val2\n], maps as '{\n  key: val,\n  key2: val2\n}')
    BLOCK       ///< mark container with block style (seqs as '- val\n', maps as 'key: val')
    KEY_LITERAL ///< mark key scalar as multiline, block literal |
    VAL_LITERAL ///< mark val scalar as multiline, block literal |
    KEY_FOLDED  ///< mark key scalar as multiline, block folded >
    VAL_FOLDED  ///< mark val scalar as multiline, block folded >
    KEY_SQUO    ///< mark key scalar as single quoted '
    VAL_SQUO    ///< mark val scalar as single quoted '
    KEY_DQUO    ///< mark key scalar as double quoted "
    VAL_DQUO    ///< mark val scalar as double quoted "
    KEY_PLAIN   ///< mark key scalar as plain scalar (unquoted, even when multiline)
    VAL_PLAIN   ///< mark val scalar as plain scalar (unquoted, even when multiline)
  ```
- Style predicates were added to `NodeType`, `Tree`, `ConstNodeRef` and `NodeRef`:
  ```cpp
    bool is_container_styled() const;
    bool is_block() const 
    bool is_flow_sl() const;
    bool is_flow_ml() const;
    bool is_flow() const;

    bool is_key_styled() const;
    bool is_val_styled() const;
    bool is_key_literal() const;
    bool is_val_literal() const;
    bool is_key_folded() const;
    bool is_val_folded() const;
    bool is_key_squo() const;
    bool is_val_squo() const;
    bool is_key_dquo() const;
    bool is_val_dquo() const;
    bool is_key_plain() const;
    bool is_val_plain() const;
  ```
- Style modifiers were also added:
  ```cpp
    void set_container_style(NodeType_e style);
    void set_key_style(NodeType_e style);
    void set_val_style(NodeType_e style);
  ```
- Emit helper predicates were added, and are used when an emitted node was built programatically without style flags:
  ```cpp
  /** choose a YAML emitting style based on the scalar's contents */
  NodeType_e scalar_style_choose(csubstr scalar) noexcept;
  /** query whether a scalar can be encoded using single quotes.
   * It may not be possible, notably when there is leading
   * whitespace after a newline. */
  bool scalar_style_query_squo(csubstr s) noexcept;
  /** query whether a scalar can be encoded using plain style (no
   * quotes, not a literal/folded block scalar). */
  bool scalar_style_query_plain(csubstr s) noexcept;
  ```

### Breaking changes

As a result of the refactor, there are some limited changes with impact in client code. Even though this was a large refactor, effort was directed at keeping maximal backwards compatibility, and the changes are not wide. But they still exist:

- The existing `parse_...()` methods in the `Parser` class were all removed. Use the corresponding `parse_...(Parser*, ...)` function from the header [`c4/yml/parse.hpp`](https://github.com/biojppm/rapidyaml/blob/master/src/c4/yml/parse.hpp).
- When instantiated by the user, the parser now needs to receive a `EventHandlerTree` object, which is responsible for building the tree. Although fully functional and tested, the structure of this class is still somewhat experimental and is still likely to change. There is an alternative event handler implementation responsible for producing the events for the YAML test suite in `test/test_suite/test_suite_event_handler.hpp`.
- The declaration and definition of `NodeType` was moved to a separate header file `c4/yml/node_type.hpp` (previously it was in `c4/yml/tree.hpp`).
- Some of the node type flags were removed, and several flags (and combination flags) were added. 
  - Most of the existing flags are kept, as well as their meaning.
  - `KEYQUO` and `VALQUO` are now masks of the several style flags for quoted scalars. In general, however, client code using these flags and `.is_val_quoted()` or `.is_key_quoted()` is not likely to require any changes.


### New type for node IDs

A type `id_type` was added to signify the integer type for the node id, defaulting to the backwards-compatible `size_t` which was previously used in the tree. In the future, this type is likely to change, *and probably to a signed type*, so client code is encouraged to always use `id_type` instead of the `size_t`, and specifically not to rely on the signedness of this type.


### Reference resolver is now exposed

The reference (ie, alias) resolver object is now exposed in
[`c4/yml/reference_resolver.hpp`](https://github.com/biojppm/rapidyaml/blob/master/src/c4/yml/reference_resolver.hpp). Previously this object was temporarily instantiated in `Tree::resolve()`. Exposing it now enables the user to reuse this object through different calls, saving a potential allocation on every call.


### Tag utilities

Tag utilities were moved to the new header [`c4/yml/tag.hpp`](https://github.com/biojppm/rapidyaml/blob/master/src/c4/yml/tag.hpp). The types `Tree::tag_directive_const_iterator` and `Tree::TagDirectiveProxy` were deprecated. Fixed also an unitialization problem with `Tree::m_tag_directives`.


### Performance improvements

To compare performance before and after this changeset, the benchmark runs were run (in the same PC), and the results were collected into these two files:
  - [results before newparser](https://github.com/biojppm/rapidyaml/blob/master/bm/results/results_before_newparser.md)
  - [results after newparser](https://github.com/biojppm/rapidyaml/blob/master/bm/results/results_after_newparser.md)
  - (suggestion: compare these files in a diff viewer)

There are a lot of results in these files, and many insights can be obtained by browsing them; too many to list here. Below we show only some selected results.


#### Parsing
Here are some figures for parsing performance, for `bm_ryml_inplace_reuse` (name before) / `bm_ryml_yaml_inplace_reuse` (name after):

|------|------------|-----------|--------|
| case | B/s before newparser | B/s after newparser | improv % |
|------|------------|-----------|--------|
| [PARSE/appveyor.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/appveyor.yml) | 168.628Mi/s | 232.017Mi/s | ~+40% |
| [PARSE/compile_commands.json](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/compile_commands.yml) | 630.17Mi/s | 609.877Mi/s | ~-3% |
| [PARSE/travis.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/travis.yml) | 193.674Mi/s | 271.598Mi/s | ~+50% |
| [PARSE/scalar_dquot_multiline.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/scalar_dquot_multiline.yml) | 224.796Mi/s | 187.335Mi/s | ~-10% |
| [PARSE/scalar_dquot_singleline.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/scalar_dquot_singleline.yml) | 339.889Mi/s | 388.924Mi/s | ~-16% |

Some conclusions:
- parse performance improved by ~30%-50% for YAML without filtering-heavy parsing.
- parse performance *decreased* by ~10%-15% for YAML with filtering-heavy parsing. There is still some scope for improvement in the parsing code, so this cost may hopefully be minimized in the future.


#### Emitting

Here are some figures emitting performance improvements retrieved from these files, for `bm_ryml_str_reserve` (name before) / `bm_ryml_yaml_str_reserve` (name after):

|------|------------|-----------|
| case | B/s before newparser | B/s after newparser |
|------|------------|-----------|
| [EMIT/appveyor.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/appveyor.yml) | 311.718Mi/s | 1018.44Mi/s |
| [EMIT/compile_commands.json](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/compile_commands.yml) | 434.206Mi/s | 771.682Mi/s |
| [EMIT/travis.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/travis.yml) | 333.322Mi/s | 1.41597Gi/s |
| [EMIT/scalar_dquot_multiline.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/scalar_dquot_multiline.yml) | 868.6Mi/s | 692.564Mi/s |
| [EMIT/scalar_dquot_singleline.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/scalar_dquot_singleline.yml) | 336.98Mi/s | 638.368Mi/s |
| [EMIT/style_seqs_flow_outer1000_inner100.yml](https://github.com/biojppm/rapidyaml/blob/master/bm/cases/style_seqs_flow_outer1000_inner100.yml) | 136.826Mi/s | 279.487Mi/s |

Emit performance improved everywhere by over 1.5x and as much as 3x-4x for YAML without filtering-heavy parsing.
