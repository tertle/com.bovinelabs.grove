# Changelog

## [2.0.0-exp] - 2026-04-19

### Breaking
* Grove editor authoring now uses Unity GraphToolkit instead of GraphView
* GraphView-era graphs, editor extensions, and compatibility paths have been removed; no backwards compatibility
* Requires Unity 6.5+

### Added
* First-pass GraphToolkit authoring flow 
* Support for GraphToolkit user, context, block, variable, constant nodes
* Native GraphToolkit subgraph support

### Changed
* Grove graphs are now authored as GraphToolkit files and imported into `GroveAuthGraph` assets as the main authoring path

## [0.9.0] - 2026-01-01

### Added
* 14 Analyzers for the source generator
* ProcessNodeAttribute for new `void Method(ref Data data, in EntityContext entityContext, ref Context context, ref Value state)` signature

### Fixed
* Avoid multiple Animation.curves calls
* Stop source generators running on Grove assemblies

## [0.8.2] - 2025-12-06

### Added
* Unity 6.4 support

## [0.8.1] - 2025-10-03

### Changed
* Sped up baking even more (3s-> 0.5s on production project).
* Added more control over naming

## [0.8.0] - 2025-07-13

### Changed
* [Breaking] Reworked instance value serialization. You can now use instance values on nearly all unity serializable types including UnityEngine.Object, array/list and large structs. It also now works with custom editors instead of just property drawers

## [0.7.3] - 2025-06-07

### Added
* StateSelectNode can now link if no sub graph specified for simple state usage
* DataEmptyNode
* Variants now work on StateSelect node
* AlwaysAddInstanceAttribute to always add a field to the instance list
* HideInstanceAttribute to hide a field from appearing in the instance menu
* Default name filtering to remove things like Select, Qualifier etc

### Removed
* Logging package references

### Changed
* Improved the inspector detecting changes in nested PropertyFields
* Source generators moved to asmdef for improved performance
* Shrunk Graph Phass Through Node to be more useful
* StateIf and StateSet are now generic on an asset to allow graph filtering

### Fixed
* Source generators spamming a warning if using a constant node
* Can no longer copy/paste Input nodes
* StateIf Inverted wasn't working
* Variant data not being removed when Variant removed from graph

## [0.7.2] - 2025-05-18

### Added
* Cut/Copy/Paste/Duplicate and hotkeys ctrl+c/x/v/d
* Undo/Redo support
* StateIf

### Removed
* DefaultState from StateNode

### Changed
* A lot of BaseNodeElement logic moved to INodeElement and extension methods in preparation of StackNode support

### Fixed
* Wrong GraphLinkNode being used if you had multiple Context

## [0.7.1] - 2025-04-26

### Added
* State nodes
* Ability to debug node data at runtime on selected entities
* Optional Unity.Localization support to nodes
* Support to use a custom GraphInstances when baking to allow assignment of extra data (useful for frame slicing etc)
* Exposed OnWindowLoaded and added AddStyleSheet to GraphEditorWindow to easily customize styling

### Fixed
* Node inspector breaking if you select a node, enter playmode and leave playmode

### Documentation
* Debugging Nodes
* Random documentation on source code

## [0.7.0] - 2025-03-22

### Added
* Initial public release.
