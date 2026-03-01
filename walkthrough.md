# Amplifier Foundation — Code Walkthrough

*2026-03-01T20:30:20Z by Showboat 0.6.1*
<!-- showboat-id: 0c5e0f40-3fd0-4247-840d-0f1b9cad69b9 -->

## What is Amplifier Foundation?

Amplifier Foundation is an ultra-thin **mechanism layer** for bundle composition that sits between `amplifier-core` (the kernel) and applications. Its core philosophy is captured in three words: **mechanism, not policy**.

Foundation provides the *how* — loading bundles from URIs, composing them together, activating modules, resolving `@mentions` — without ever dictating the *what*. The foundation bundle content co-located in this repository is just content: it's discovered and loaded exactly the same way any other bundle would be.

The central abstraction is the **Bundle**: a composable unit containing mount-plan configuration and resources. You load bundles from any source (git, file, http, zip), compose them together, prepare them (downloading and installing module dependencies), and then create a live `AmplifierSession` ready to execute instructions.

Here is the high-level flow:

```
load_bundle(uri) → Bundle → compose() → prepare() → PreparedBundle → create_session() → AmplifierSession → execute()
```

Let's walk through every piece of this system, starting with the project layout.

## 1. Project Structure

Let's see the full layout of the `amplifier_foundation` package — the Python library at the heart of this repo:

```bash
find amplifier_foundation -type f -name '*.py' | sort
```

```output
amplifier_foundation/__init__.py
amplifier_foundation/bundle.py
amplifier_foundation/cache/__init__.py
amplifier_foundation/cache/disk.py
amplifier_foundation/cache/protocol.py
amplifier_foundation/cache/simple.py
amplifier_foundation/dicts/__init__.py
amplifier_foundation/dicts/merge.py
amplifier_foundation/dicts/navigation.py
amplifier_foundation/discovery/__init__.py
amplifier_foundation/exceptions.py
amplifier_foundation/io/__init__.py
amplifier_foundation/io/files.py
amplifier_foundation/io/frontmatter.py
amplifier_foundation/io/yaml.py
amplifier_foundation/mentions/__init__.py
amplifier_foundation/mentions/deduplicator.py
amplifier_foundation/mentions/loader.py
amplifier_foundation/mentions/models.py
amplifier_foundation/mentions/parser.py
amplifier_foundation/mentions/protocol.py
amplifier_foundation/mentions/resolver.py
amplifier_foundation/mentions/utils.py
amplifier_foundation/modules/__init__.py
amplifier_foundation/modules/activator.py
amplifier_foundation/modules/install_state.py
amplifier_foundation/paths/__init__.py
amplifier_foundation/paths/construction.py
amplifier_foundation/paths/discovery.py
amplifier_foundation/paths/resolution.py
amplifier_foundation/registry.py
amplifier_foundation/serialization.py
amplifier_foundation/session/__init__.py
amplifier_foundation/session/events.py
amplifier_foundation/session/fork.py
amplifier_foundation/session/slice.py
amplifier_foundation/sources/__init__.py
amplifier_foundation/sources/file.py
amplifier_foundation/sources/git.py
amplifier_foundation/sources/http.py
amplifier_foundation/sources/protocol.py
amplifier_foundation/sources/resolver.py
amplifier_foundation/sources/zip.py
amplifier_foundation/tracing.py
amplifier_foundation/updates/__init__.py
amplifier_foundation/validator.py
```

The package is organized into focused sub-packages:

| Sub-package | Purpose |
|---|---|
| `bundle.py` | Core `Bundle` dataclass plus `PreparedBundle` and `BundleModuleResolver` |
| `registry.py` | `BundleRegistry` — central loading, caching, include resolution |
| `validator.py` | Structural validation of bundles |
| `exceptions.py` | Clean exception hierarchy |
| `serialization.py` | JSON sanitization for LLM responses |
| `tracing.py` | W3C Trace Context ID generation for sub-sessions |
| `sources/` | URI → local-path resolution (git, file, http, zip handlers) |
| `modules/` | Module activation — download, install dependencies, add to sys.path |
| `mentions/` | `@mention` parsing, resolution, recursive loading, deduplication |
| `dicts/` | Deep merge and nested dict navigation |
| `paths/` | URI parsing, path construction, file discovery |
| `io/` | YAML reading, frontmatter parsing, cloud-sync-safe file I/O |
| `cache/` | In-memory and disk cache providers |
| `session/` | Session fork, slice-to-turn, event slicing |

Alongside the library, the repo ships *content* — the foundation bundle itself (in `bundle.md`), reference providers, agents, behaviors, context files, and examples. That content is loaded by the library the same way any external bundle would be.

## 2. The Bundle Dataclass — The Core Composable Unit

Everything starts with `Bundle`. It is a Python dataclass that holds mount-plan configuration and resources. Let's look at its definition:

```bash
sed -n '26,75p' amplifier_foundation/bundle.py
```

```output
@dataclass
class Bundle:
    """Composable unit containing mount plan config and resources.

    Bundles are the core composable unit in amplifier-foundation. They contain
    mount plan configuration and resources, producing mount plans for AmplifierSession.

    Attributes:
        name: Bundle name (namespace for @mentions).
        version: Bundle version string.
        description: Optional description.
        includes: List of bundle URIs to include.
        session: Session config (orchestrator, context).
        providers: List of provider configs.
        tools: List of tool configs.
        hooks: List of hook configs.
        agents: Dict mapping agent name to definition.
        context: Dict mapping context name to file path.
        instruction: System instruction from markdown body.
        base_path: Path to bundle root directory.
        source_base_paths: Dict mapping namespace to base_path for @mention resolution.
            Tracks original base_path for each bundle during composition, enabling
            @namespace:path references to resolve correctly to source files.
    """

    # Metadata
    name: str
    version: str = "1.0.0"
    description: str = ""
    includes: list[str] = field(default_factory=list)

    # Mount plan sections
    session: dict[str, Any] = field(default_factory=dict)
    providers: list[dict[str, Any]] = field(default_factory=list)
    tools: list[dict[str, Any]] = field(default_factory=list)
    hooks: list[dict[str, Any]] = field(default_factory=list)

    # Resources
    agents: dict[str, dict[str, Any]] = field(default_factory=dict)
    context: dict[str, Path] = field(default_factory=dict)
    instruction: str | None = None

    # Internal
    base_path: Path | None = None
    source_base_paths: dict[str, Path] = field(
        default_factory=dict
    )  # Track base_path for each source namespace
    _pending_context: dict[str, str] = field(
        default_factory=dict
    )  # Context refs needing namespace resolution
```

A Bundle groups three kinds of data:

**Metadata** — `name`, `version`, `description`, `includes`. The name serves as the namespace for `@mention` resolution (e.g. `@foundation:context/AGENTS.md`). The `includes` list references other bundles to compose in.

**Mount plan sections** — `session` (orchestrator + context module config), `providers` (LLM provider modules), `tools` (tool modules), `hooks` (hook modules). These map directly to sections in an AmplifierSession mount plan.

**Resources** — `agents` (agent definitions), `context` (context files), and `instruction` (system instruction from markdown body).

**Internal tracking** — `base_path` (where the bundle lives on disk), `source_base_paths` (maps each included bundle's namespace to its root path, enabling cross-bundle `@mention` resolution), and `_pending_context` (context refs that need deferred resolution after composition).

The Bundle is created from parsed YAML/frontmatter via the `from_dict` class method:

```bash
sed -n '431,463p' amplifier_foundation/bundle.py
```

```output
    @classmethod
    def from_dict(cls, data: dict[str, Any], base_path: Path | None = None) -> Bundle:
        """Create Bundle from parsed dict (from YAML/frontmatter).

        Args:
            data: Dict with bundle configuration.
            base_path: Path to bundle root directory.

        Returns:
            Bundle instance.
        """
        bundle_meta = data.get("bundle", {})

        # Parse context - returns (resolved, pending) tuple
        resolved_context, pending_context = _parse_context(
            data.get("context", {}), base_path
        )

        return cls(
            name=bundle_meta.get("name", ""),
            version=bundle_meta.get("version", "1.0.0"),
            description=bundle_meta.get("description", ""),
            includes=data.get("includes", []),
            session=data.get("session", {}),
            providers=data.get("providers", []),
            tools=data.get("tools", []),
            hooks=data.get("hooks", []),
            agents=_parse_agents(data.get("agents", {}), base_path),
            context=resolved_context,
            _pending_context=pending_context,
            instruction=None,  # Set separately from markdown body
            base_path=base_path,
        )
```

Notice that `instruction` is set to `None` here — it gets filled in separately from the markdown body when loading a `.md` bundle file. Context parsing splits into *resolved* (local paths) and *pending* (namespaced refs like `foundation:context/file.md` that need deferred resolution after composition populates `source_base_paths`).

## 3. Bundle File Formats

Bundles are declared in one of two formats:

### Markdown with Frontmatter (`bundle.md`)

This is the primary format. YAML configuration lives between `---` delimiters at the top; the markdown body below becomes the system instruction. Here is the beginning of the real foundation bundle:

```bash
sed -n '1,70p' bundle.md
```

```output
---
bundle:
  name: foundation
  version: 1.0.0
  description: Foundation bundle - provider-agnostic base configuration
  # Sub-bundles available within this bundle's namespace
  # These are discoverable via `amplifier bundle list` when foundation is loaded
  sub_bundles:
    - name: amplifier-dev
      path: bundles/amplifier-dev.yaml
      description: Amplifier ecosystem development - multi-repo workflows, shadow environments
    - name: minimal
      path: bundles/minimal.yaml
      description: Minimal tools only - filesystem, bash, web
    - name: with-anthropic
      path: bundles/with-anthropic.yaml
      description: Foundation with Anthropic Claude provider
    - name: with-openai
      path: bundles/with-openai.yaml
      description: Foundation with OpenAI provider
    - name: exp-delegation
      path: experiments/delegation-only
      description: Experimental delegation-only bundle

includes:
  # Ecosystem expert behaviors (provides @amplifier: and @core: namespaces)
  - bundle: git+https://github.com/microsoft/amplifier@main#subdirectory=behaviors/amplifier-expert.yaml
  - bundle: git+https://github.com/microsoft/amplifier-core@main#subdirectory=behaviors/core-expert.yaml
  # Foundation behaviors
  - bundle: foundation:behaviors/sessions
  - bundle: foundation:behaviors/status-context
  - bundle: foundation:behaviors/redaction
  - bundle: foundation:behaviors/todo-reminder
  - bundle: foundation:behaviors/streaming-ui
  # External bundles
  - bundle: git+https://github.com/microsoft/amplifier-bundle-recipes@main#subdirectory=behaviors/recipes.yaml
  - bundle: git+https://github.com/microsoft/amplifier-bundle-design-intelligence@main#subdirectory=behaviors/design-intelligence.yaml
  - bundle: git+https://github.com/microsoft/amplifier-bundle-python-dev@main
  - bundle: git+https://github.com/microsoft/amplifier-bundle-shadow@main
  - bundle: git+https://github.com/microsoft/amplifier-module-tool-skills@main#subdirectory=behaviors/skills.yaml
  - bundle: git+https://github.com/microsoft/amplifier-module-hook-shell@main#subdirectory=behaviors/hook-shell.yaml


session:
  orchestrator:
    module: loop-streaming
    source: git+https://github.com/microsoft/amplifier-module-loop-streaming@main
    config:
      extended_thinking: true
  context:
    module: context-simple
    source: git+https://github.com/microsoft/amplifier-module-context-simple@main
    config:
      max_tokens: 300000
      compact_threshold: 0.8
      auto_compact: true

tools:
  - module: tool-filesystem
    source: git+https://github.com/microsoft/amplifier-module-tool-filesystem@main
  - module: tool-bash
    source: git+https://github.com/microsoft/amplifier-module-tool-bash@main
  - module: tool-web
    source: git+https://github.com/microsoft/amplifier-module-tool-web@main
  - module: tool-search
    source: git+https://github.com/microsoft/amplifier-module-tool-search@main
  - module: tool-task
    source: git+https://github.com/microsoft/amplifier-module-tool-task@main
    config:
      exclude_tools: [tool-task]  # Spawned agents do the work, they don't delegate
```

Key elements visible here:

- **`bundle:`** — metadata (name, version, description, sub-bundles)
- **`includes:`** — other bundles to compose in. Three styles:
  - Full git URI: `git+https://github.com/...@main`
  - Git URI with subdirectory fragment: `git+https://...@main#subdirectory=behaviors/x.yaml`
  - Namespace-relative: `foundation:behaviors/sessions` (resolved via the registry)
- **`session:`** — orchestrator and context modules with their source URIs and configs
- **`tools:`** — list of tool modules, each with a `module` identifier and `source` URI
- **`agents:`** — references to agent definition files (shown further below in the same file)

The markdown body (everything after the closing `---`) becomes the `instruction` field — the system prompt. It references context files using `@mention` syntax like `@foundation:context/shared/common-system-base.md`.

### Pure YAML (`bundle.yaml`)

Provider bundles and behaviors typically use pure YAML (no markdown body = no system instruction). Here is what a provider YAML bundle looks like:

```bash
cat providers/anthropic-sonnet.yaml
```

```output
bundle:
  name: provider-anthropic-sonnet
  version: 1.0.0
  description: Anthropic Claude Sonnet provider

providers:
  - module: provider-anthropic
    source: git+https://github.com/microsoft/amplifier-module-provider-anthropic@main
    config:
      default_model: claude-sonnet-4-5
      debug: true
      raw_debug: true
```

A provider bundle contributes just one section — `providers:`. When composed with the foundation bundle, its provider list merges in via `merge_module_lists()`. This is the power of composition: small, focused bundles combine into a complete configuration.

The frontmatter parser that splits markdown into YAML + body is straightforward:

```bash
cat amplifier_foundation/io/frontmatter.py
```

```output
"""Frontmatter parsing for markdown files with YAML headers."""

from __future__ import annotations

import re
from typing import Any

import yaml


def parse_frontmatter(text: str) -> tuple[dict[str, Any], str]:
    """Parse YAML frontmatter from markdown text.

    Extracts YAML between --- delimiters at the start of the text.

    Args:
        text: Markdown text with optional YAML frontmatter.

    Returns:
        Tuple of (frontmatter_dict, body_text).
        If no frontmatter, returns ({}, original_text).

    Raises:
        yaml.YAMLError: If frontmatter contains invalid YAML.
    """
    # Match --- at start, then content, then ---
    pattern = r"^---\s*\n(.*?)\n---\s*\n?"
    match = re.match(pattern, text, re.DOTALL)

    if not match:
        return {}, text

    frontmatter_str = match.group(1)
    body = text[match.end() :]

    frontmatter = yaml.safe_load(frontmatter_str) or {}

    return frontmatter, body
```

Simple regex to find `---` delimiters, `yaml.safe_load()` on the frontmatter, and everything after the closing `---` becomes the body (system instruction).

## 4. URI Parsing — How Source Addresses Are Understood

Before loading anything, URIs must be parsed. The `parse_uri()` function in `paths/resolution.py` handles every supported scheme:

```bash
sed -n '31,65p' amplifier_foundation/paths/resolution.py
```

```output
@dataclass
class ParsedURI:
    """Parsed URI components."""

    scheme: str  # git, file, http, https, zip, or empty for package names
    host: str  # github.com, etc.
    path: str  # /org/repo or local path
    ref: str  # @main, @v1.0.0, etc. (empty if not specified)
    subpath: str  # path inside container (from #subdirectory= fragment)

    @property
    def is_git(self) -> bool:
        """True if this is a git URI."""
        return self.scheme == "git" or self.scheme.startswith("git+")

    @property
    def is_file(self) -> bool:
        """True if this is a file URI or local path."""
        return self.scheme == "file" or (self.scheme == "" and "/" in self.path)

    @property
    def is_http(self) -> bool:
        """True if this is an HTTP/HTTPS URI."""
        return self.scheme in ("http", "https")

    @property
    def is_zip(self) -> bool:
        """True if this is a zip URI (zip+https://, zip+file://)."""
        return self.scheme.startswith("zip+")

    @property
    def is_package(self) -> bool:
        """True if this looks like a package/bundle name."""
        return self.scheme == "" and "/" not in self.path

```

`ParsedURI` decomposes a URI into scheme, host, path, ref, and subpath. The boolean properties let source handlers quickly determine which URIs they can handle.

The `parse_uri()` function dispatches based on prefix:

```bash
sed -n '93,157p' amplifier_foundation/paths/resolution.py
```

```output
def parse_uri(uri: str) -> ParsedURI:
    """Parse a URI into components.

    Supports pip/uv standard syntax with #subdirectory= fragment:
    - git+https://github.com/org/repo@ref#subdirectory=path/inside
    - zip+https://example.com/bundle.zip#subdirectory=path/inside
    - zip+file:///local/archive.zip#subdirectory=path/inside
    - file:///path/to/file
    - /absolute/path
    - ./relative/path
    - package-name
    - package/subpath

    Args:
        uri: URI string to parse.

    Returns:
        ParsedURI with extracted components.
    """
    # Handle git+ prefix (pip/uv standard)
    if uri.startswith("git+"):
        return _parse_vcs_uri(uri, prefix="git+")

    # Handle zip+ prefix (extended pattern for archives)
    if uri.startswith("zip+"):
        return _parse_vcs_uri(uri, prefix="zip+")

    # Handle explicit file:// scheme
    if uri.startswith("file://"):
        path, subpath = _extract_fragment_subpath(uri[7:])
        return ParsedURI(scheme="file", host="", path=path, ref="", subpath=subpath)

    # Handle absolute paths
    if uri.startswith("/"):
        return ParsedURI(scheme="file", host="", path=uri, ref="", subpath="")

    # Handle relative paths
    if uri.startswith("./") or uri.startswith("../"):
        return ParsedURI(scheme="file", host="", path=uri, ref="", subpath="")

    # Handle http/https URLs
    if uri.startswith("http://") or uri.startswith("https://"):
        parsed = urlparse(uri)
        subpath = _extract_subdirectory_from_fragment(parsed.fragment)
        return ParsedURI(
            scheme=parsed.scheme,
            host=parsed.netloc,
            path=parsed.path,
            ref="",
            subpath=subpath,
        )

    # Assume package name or package/subpath
    if "/" in uri:
        # Could be package/subpath like "foundation/providers/anthropic"
        parts = uri.split("/", 1)
        return ParsedURI(
            scheme="",
            host="",
            path=parts[0],
            ref="",
            subpath=parts[1] if len(parts) > 1 else "",
        )

    return ParsedURI(scheme="", host="", path=uri, ref="", subpath="")
```

The function follows pip/uv standard URI syntax. The `#subdirectory=` fragment (critical for referencing a path *inside* a git repo) is extracted by a small helper. The `@ref` syntax (like `@main` or `@v1.0.0`) is parsed from git URIs to specify which branch or tag to clone.

Also important is `ResolvedSource`, which tracks both the requested path and the source root:

```bash
sed -n '67,91p' amplifier_foundation/paths/resolution.py
```

```output
@dataclass
class ResolvedSource:
    """Result of resolving a source URI to local paths.

    Tracks both the requested path (which may be a subdirectory) and the
    source root (full clone/extract root), enabling @-mention resolution
    to access files outside the immediate subdirectory when needed.

    When loading from a subdirectory (e.g., git+https://...#subdirectory=behaviors/x),
    the registry can walk back from active_path to source_root to find the nearest
    bundle.md/bundle.yaml and register it for @-mention access.

    Attributes:
        active_path: The requested path (subdirectory or root).
        source_root: The full clone/extract root (always the container root).
    """

    active_path: Path  # The requested path (subdirectory or root)
    source_root: Path  # The full clone/extract root (always the container root)

    @property
    def is_subdirectory(self) -> bool:
        """True if active_path is a subdirectory of source_root."""
        return self.active_path != self.source_root

```

This dual-path design is key: when you load `git+https://...@main#subdirectory=behaviors/logging`, the `active_path` is the logging behavior directory, but `source_root` is the full repo clone. This lets the registry walk up from the subdirectory to find the root `bundle.md` and register it for namespace resolution.

## 5. Source Resolution — Downloading and Caching

`SimpleSourceResolver` is the orchestrator that turns a URI string into a local path on disk. It delegates to a chain of source handlers:

```bash
cat amplifier_foundation/sources/resolver.py
```

```output
"""Simple source resolver implementation."""

from __future__ import annotations

from pathlib import Path

from amplifier_foundation.exceptions import BundleNotFoundError
from amplifier_foundation.paths.resolution import ResolvedSource
from amplifier_foundation.paths.resolution import get_amplifier_home
from amplifier_foundation.paths.resolution import parse_uri

from .file import FileSourceHandler
from .git import GitSourceHandler
from .http import HttpSourceHandler
from .protocol import SourceHandlerProtocol
from .zip import ZipSourceHandler


class SimpleSourceResolver:
    """Simple implementation of SourceResolverProtocol.

    Supports:
    - file:// and local paths via FileSourceHandler
    - git+https:// via GitSourceHandler
    - https:// and http:// via HttpSourceHandler
    - zip+https:// and zip+file:// via ZipSourceHandler

    Apps can extend by adding custom handlers.
    """

    def __init__(
        self,
        cache_dir: Path | None = None,
        base_path: Path | None = None,
    ) -> None:
        """Initialize resolver.

        Args:
            cache_dir: Directory for caching remote content.
            base_path: Base path for resolving relative paths.
        """
        self.cache_dir = cache_dir or get_amplifier_home() / "cache" / "bundles"
        self.base_path = base_path or Path.cwd()

        # Default handlers - order matters for URI matching
        self._handlers: list[SourceHandlerProtocol] = [
            FileSourceHandler(base_path=self.base_path),
            GitSourceHandler(),
            ZipSourceHandler(),  # Must be before HttpSourceHandler (zip+https matches before https)
            HttpSourceHandler(),
        ]

    def add_handler(self, handler: SourceHandlerProtocol) -> None:
        """Add a custom source handler.

        Handlers are tried in order, first match wins.

        Args:
            handler: Handler to add.
        """
        self._handlers.insert(0, handler)  # Custom handlers take priority

    async def resolve(self, uri: str) -> ResolvedSource:
        """Resolve a URI to local paths.

        Args:
            uri: URI string.

        Returns:
            ResolvedSource with active_path and source_root.

        Raises:
            BundleNotFoundError: If no handler can resolve the URI.
        """
        parsed = parse_uri(uri)

        for handler in self._handlers:
            if handler.can_handle(parsed):
                return await handler.resolve(parsed, self.cache_dir)

        raise BundleNotFoundError(f"No handler for URI: {uri}")
```

The design is a classic chain-of-responsibility: parse the URI, iterate handlers, first one that says `can_handle(parsed)` gets to resolve it. Custom handlers added via `add_handler()` are inserted at position 0, so they take priority.

Handler order matters — `ZipSourceHandler` comes before `HttpSourceHandler` because `zip+https://` would otherwise match the HTTP handler's `https://` check.

### GitSourceHandler — The Workhorse

Most bundles and modules come from git. The `GitSourceHandler` performs shallow clones to a cache directory:

```bash
sed -n '144,229p' amplifier_foundation/sources/git.py
```

```output
    async def resolve(self, parsed: ParsedURI, cache_dir: Path) -> ResolvedSource:
        """Resolve git URI to local cached path.

        Args:
            parsed: Parsed URI components.
            cache_dir: Directory for caching cloned repos.

        Returns:
            ResolvedSource with active_path and source_root.

        Raises:
            BundleNotFoundError: If clone fails or ref not found.
        """
        git_url = self._build_git_url(parsed)
        ref = parsed.ref or "HEAD"
        cache_path = self._get_cache_path(parsed, cache_dir)

        # Check if already cached and valid
        if cache_path.exists():
            # Verify cache integrity before using
            if not self._verify_clone_integrity(cache_path):
                logger.warning(f"Cached clone is invalid, removing: {cache_path}")
                shutil.rmtree(cache_path, ignore_errors=True)
            else:
                result_path = cache_path
                if parsed.subpath:
                    result_path = cache_path / parsed.subpath
                if result_path.exists():
                    return ResolvedSource(active_path=result_path, source_root=cache_path)

        # Clone repository
        cache_path.parent.mkdir(parents=True, exist_ok=True)

        # Remove partial clone if exists
        if cache_path.exists():
            shutil.rmtree(cache_path)

        try:
            # Shallow clone with specific ref
            # Note: "HEAD" is not a valid --branch argument; it's a symbolic reference.
            # When ref is HEAD (or not specified), let git clone use the repo's default branch.
            clone_args = ["git", "clone", "--depth", "1"]
            if parsed.ref and parsed.ref != "HEAD":
                clone_args.extend(["--branch", parsed.ref])
            clone_args.extend([git_url, str(cache_path)])

            subprocess.run(
                clone_args,
                check=True,
                capture_output=True,
                text=True,
            )

            # Verify clone completed with expected structure
            if not self._verify_clone_integrity(cache_path):
                # Clone succeeded but result is invalid - remove and raise error
                shutil.rmtree(cache_path, ignore_errors=True)
                raise BundleNotFoundError(
                    f"Clone of {git_url}@{ref} completed but result is invalid "
                    "(missing pyproject.toml/setup.py/bundle.md). "
                    "This may indicate a network issue or cloud sync interference."
                )

            # Save metadata after successful clone
            commit = self._get_local_commit(cache_path)
            self._save_cache_metadata(
                cache_path,
                {
                    "cached_at": datetime.now().isoformat(),
                    "ref": ref,
                    "commit": commit,
                    "git_url": git_url,
                },
            )
        except subprocess.CalledProcessError as e:
            raise BundleNotFoundError(f"Failed to clone {git_url}@{ref}: {e.stderr}") from e

        # Return path with subpath if specified
        result_path = cache_path
        if parsed.subpath:
            result_path = cache_path / parsed.subpath

        if not result_path.exists():
            raise BundleNotFoundError(f"Subpath not found after clone: {parsed.subpath}")

        return ResolvedSource(active_path=result_path, source_root=cache_path)
```

The flow:

1. **Cache check**: If the cache directory exists and passes integrity verification (`.git` present, plus `pyproject.toml` or `bundle.md`), return it immediately.
2. **Clone**: Shallow clone (`--depth 1`) with an optional `--branch` flag for the ref. HEAD/default branch is used if no ref is specified.
3. **Integrity verification**: Post-clone check ensures the result has expected structure — catches partial clones from network issues.
4. **Metadata**: Saves `cached_at`, `ref`, `commit`, and `git_url` to `.amplifier_cache_meta.json` inside the clone for later update checking.
5. **Subpath application**: If the URI had `#subdirectory=path`, navigate into that subdirectory within the clone.

The cache path is deterministic: `{repo-name}-{sha256(git_url@ref)[:16]}`. This means the same URI always maps to the same cache directory.

The handler also implements `get_status()` for bandwidth-efficient update detection using `git ls-remote` (no download needed), and `update()` to force a fresh clone.

## 6. The Bundle Registry — Loading, Includes, and Namespace Resolution

`BundleRegistry` is the central brain for bundle management. It handles registration, loading, include resolution, and state persistence. The entry point most callers use is the convenience function:

```bash
sed -n '1002,1021p' amplifier_foundation/registry.py
```

```output
async def load_bundle(
    source: str,
    auto_include: bool = True,
    registry: BundleRegistry | None = None,
) -> Bundle:
    """Convenience function to load a bundle.

    Args:
        source: URI or bundle name.
        auto_include: Whether to load includes.
        registry: Optional registry (creates default if not provided).

    Returns:
        Loaded Bundle.
    """
    registry = registry or BundleRegistry()
    return await registry._load_single(
        source, auto_register=True, auto_include=auto_include
    )
```

The real work happens inside `_load_single()`. Here's its core logic:

```bash
sed -n '284,456p' amplifier_foundation/registry.py
```

```output
    async def _load_single(
        self,
        name_or_uri: str,
        *,
        auto_register: bool = True,
        auto_include: bool = True,
        refresh: bool = False,  # noqa: ARG002 - Reserved for future cache bypass
    ) -> Bundle:
        """Load a single bundle by name or URI.

        Args:
            name_or_uri: Bundle name or URI.
            auto_register: Register URI bundles by extracted name.
            auto_include: Load and compose includes.
            refresh: Bypass cache, fetch fresh (reserved for future use).

        Returns:
            Loaded Bundle.

        Raises:
            BundleNotFoundError: Bundle not found.
            BundleLoadError: Failed to load bundle.
        """
        # Determine if this is a registered name or a URI
        registered_name: str | None = None
        uri: str

        if name_or_uri in self._registry:
            registered_name = name_or_uri
            uri = self._registry[name_or_uri].uri
        else:
            uri = name_or_uri

        # Cycle detection
        if uri in self._loading:
            raise BundleDependencyError(f"Circular dependency detected: {uri}")

        self._loading.add(uri)
        try:
            # Resolve URI to local paths (active_path and source_root)
            resolved = await self._source_resolver.resolve(uri)
            if resolved is None:
                raise BundleNotFoundError(f"Could not resolve URI: {uri}")

            local_path = resolved.active_path

            # Load bundle from path
            bundle = await self._load_from_path(local_path)

            # Track root bundle info for sub-bundle detection
            root_bundle_path: Path | None = None
            root_bundle: Bundle | None = None

            # Detect sub-bundles by walking up to find a root bundle.md/yaml
            # This works for:
            # - git URIs with #subdirectory= fragments (resolved.is_subdirectory=True)
            # - file:// URIs pointing to files within a bundle's directory structure
            # - Any other case where a bundle file is nested within another bundle
            #
            # We try to find a root bundle by walking up from the PARENT of the
            # bundle directory. This skips the current bundle and looks for a root
            # bundle above it in the directory hierarchy.
            if local_path.is_file():
                # Bundle file: start from grandparent (parent of the directory containing the file)
                search_start = local_path.parent.parent
            else:
                # Bundle directory: start from parent directory
                search_start = local_path.parent

            # Use source_root as stop boundary if available, otherwise use cache root
            cache_root = Path.home() / ".amplifier" / "cache"
            stop_boundary = resolved.source_root if resolved.source_root else cache_root

            root_bundle_path = self._find_nearest_bundle_file(
                start=search_start,
                stop=stop_boundary,
            )

            # Compare directories, not file paths - local_path may be a directory while
            # root_bundle_path is always a file. We need to check if they refer to the
            # same bundle location.
            bundle_dir = local_path.parent if local_path.is_file() else local_path
            root_bundle_dir = root_bundle_path.parent if root_bundle_path else None

            if root_bundle_path and root_bundle_dir != bundle_dir:
                # Found a root bundle that's different from our loaded bundle
                root_bundle = await self._load_from_path(root_bundle_path)
                if root_bundle.name:
                    bundle.source_base_paths[root_bundle.name] = resolved.source_root
                    logger.debug(
                        f"Sub-bundle '{bundle.name}' registered root namespace "
                        f"@{root_bundle.name}: -> {resolved.source_root}"
                    )

                    # Register the root bundle itself if not already registered
                    # This ensures root bundles are tracked for version updates
                    # even when only accessed via sub-bundle includes
                    if root_bundle.name not in self._registry:
                        # Construct root bundle URI by stripping #subdirectory= fragment
                        root_uri = uri.split("#")[0] if "#" in uri else uri
                        self._registry[root_bundle.name] = BundleState(
                            uri=root_uri,
                            name=root_bundle.name,
                            version=root_bundle.version,
                            loaded_at=datetime.now(),
                            local_path=str(
                                root_bundle_path.parent
                            ),  # Directory, not file
                            is_root=True,
                            root_name=None,
                        )
                        logger.debug(f"Registered root bundle: {root_bundle.name}")

                # Also register subdirectory bundle's own name if different
                if bundle.name and bundle.name != root_bundle.name:
                    bundle.source_base_paths[bundle.name] = resolved.source_root
                    logger.debug(
                        f"Sub-bundle also registered own namespace "
                        f"@{bundle.name}: -> {resolved.source_root}"
                    )

            # Determine if this is a root bundle or sub-bundle
            # A bundle is a sub-bundle if we found a DIFFERENT root bundle above it
            is_root_bundle = True
            root_bundle_name: str | None = None

            if root_bundle and root_bundle.name and root_bundle.name != bundle.name:
                # Found a different root bundle - this is a sub-bundle
                is_root_bundle = False
                root_bundle_name = root_bundle.name

            # Register bundle for namespace resolution before processing includes.
            # This is needed even when auto_register=False because the bundle's
            # own includes may reference its namespace (self-referencing includes
            # like "design-intelligence:behaviors/design-intelligence").
            if bundle.name and bundle.name not in self._registry:
                self._registry[bundle.name] = BundleState(
                    uri=uri,
                    name=bundle.name,
                    version=bundle.version,
                    loaded_at=datetime.now(),
                    local_path=str(local_path),
                    is_root=is_root_bundle,
                    root_name=root_bundle_name,
                )
                logger.debug(
                    f"Registered bundle for namespace resolution: {bundle.name} "
                    f"(is_root={is_root_bundle}, root_name={root_bundle_name})"
                )

            # Update state for known bundle (pre-registered via well-known bundles, etc.)
            # Handle both: loaded by registered name OR loaded by URI but bundle.name matches registry
            update_name = registered_name or (
                bundle.name if bundle.name in self._registry else None
            )
            if update_name:
                state = self._registry[update_name]
                state.version = bundle.version
                state.loaded_at = datetime.now()
                state.local_path = str(local_path)

            # Load includes and compose
            if auto_include and bundle.includes:
                bundle = await self._compose_includes(bundle, parent_name=bundle.name)

            # Store source URI for update checking (used by check_bundle_status)
            # Must be set AFTER composition since compose() returns a new Bundle
            bundle._source_uri = uri  # type: ignore[attr-defined]

            return bundle

        finally:
            self._loading.discard(uri)
```

This is the most complex method in the codebase, so let's unpack the flow:

1. **Name or URI?** — If `name_or_uri` is a registered name, look up its URI from the internal registry.
2. **Cycle detection** — Track URIs currently being loaded in `self._loading`; raise `BundleDependencyError` if we see the same URI again.
3. **Resolve to local path** — `self._source_resolver.resolve(uri)` downloads/caches remote sources and returns a `ResolvedSource` with `active_path` and `source_root`.
4. **Load from path** — Parse the bundle file (markdown with frontmatter or YAML) into a `Bundle` instance via `_load_from_path()`.
5. **Sub-bundle detection** — Walk up the directory tree from the loaded bundle to find a parent `bundle.md`/`bundle.yaml`. This is how `foundation:behaviors/logging` discovers the foundation root bundle and registers it in `source_base_paths` for `@mention` resolution.
6. **Registry registration** — Register the bundle by name for namespace resolution, even before processing includes (bundles may self-reference their own namespace in includes).
7. **Load includes** — If the bundle has an `includes:` list, recursively load and compose all included bundles via `_compose_includes()`.

### Include Resolution

The `_compose_includes()` method is where bundle composition happens during loading. It processes includes in parallel:

```bash
sed -n '506,581p' amplifier_foundation/registry.py
```

```output
    async def _compose_includes(
        self, bundle: Bundle, parent_name: str | None = None
    ) -> Bundle:
        """Load and compose included bundles with parallelization.

        Args:
            bundle: The bundle to compose includes for.
            parent_name: Name of the parent bundle (for tracking relationships).
        """
        if not bundle.includes:
            return bundle

        # Pre-load any namespace bundles referenced in includes (sequential - has ordering deps)
        # This ensures local_path is populated before we try to resolve namespace:path syntax
        await self._preload_namespace_bundles(bundle.includes)

        # Phase 1: Parse and resolve all include sources first
        include_sources: list[str] = []
        for include in bundle.includes:
            include_source = self._parse_include(include)
            if include_source:
                try:
                    # Resolve namespace:path syntax before loading
                    resolved_source = self._resolve_include_source(include_source)
                    if resolved_source is None:
                        # Distinguish: namespace exists but path not found (error) vs namespace not registered (optional)
                        if ":" in include_source and "://" not in include_source:
                            namespace = include_source.split(":")[0]
                            if self._registry.get(namespace):
                                raise BundleDependencyError(
                                    f"Include resolution failed: '{include_source}'. "
                                    f"Namespace '{namespace}' is registered but the path doesn't exist."
                                )
                        logger.warning(
                            f"Include skipped (unregistered namespace): {include_source}"
                        )
                        continue
                    include_sources.append(resolved_source)
                except BundleNotFoundError:
                    # Includes are opportunistic - but warn so users know
                    logger.warning(f"Include not found (skipping): {include_source}")

        if not include_sources:
            return bundle

        # Phase 2: Load all includes in PARALLEL
        tasks = [
            self._load_single(source, auto_register=True, auto_include=True)
            for source in include_sources
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Collect successful loads
        included_bundles: list[Bundle] = []
        included_names: list[str] = []
        for source, result in zip(include_sources, results):
            if isinstance(result, BaseException):
                logger.warning(f"Include failed (skipping): {source} - {result}")
            else:
                included_bundles.append(result)
                if result.name:
                    included_names.append(result.name)

        if not included_bundles:
            return bundle

        # Record include relationships in registry state
        if parent_name and included_names:
            self._record_include_relationships(parent_name, included_names)

        # Compose: includes first, then current bundle overrides (order matters here)
        result = included_bundles[0]
        for included in included_bundles[1:]:
            result = result.compose(included)

        return result.compose(bundle)
```

Key design choices visible here:

- **Namespace pre-loading**: Before resolving include URIs, namespace bundles (referenced via `foundation:behaviors/...`) are loaded so their `local_path` is available for path construction.
- **Parallel loading**: All includes are loaded concurrently via `asyncio.gather()`.
- **Opportunistic**: Failed includes are warned and skipped, not fatal.
- **Composition order**: Includes are composed together first, then the *current* bundle is composed on top. This means the current bundle's settings override anything from its includes — "later overrides earlier."
- **Recursive**: Each `_load_single()` call may itself trigger `_compose_includes()`, so the entire dependency tree is resolved recursively.

The `_resolve_include_source()` method is particularly interesting — it converts `foundation:behaviors/sessions` into a proper git URI with `#subdirectory=` by looking up the namespace in the registry and constructing the full path.

## 7. Bundle Composition — The Merge Engine

Composition is how small, focused bundles combine into a complete configuration. The `compose()` method uses different merge strategies for each section:

```bash
sed -n '77,185p' amplifier_foundation/bundle.py
```

```output
    def compose(self, *others: Bundle) -> Bundle:
        """Compose this bundle with others (later overrides earlier).

        Creates a new Bundle with merged configuration. For each section:
        - session: deep merge (later overrides)
        - providers/tools/hooks: merge by module ID
        - agents/context: later overrides earlier
        - instruction: later replaces earlier

        Args:
            others: Bundles to compose with.

        Returns:
            New Bundle with merged configuration.
        """
        # Initialize source_base_paths: copy self's or create from self's name/base_path
        initial_base_paths = (
            dict(self.source_base_paths) if self.source_base_paths else {}
        )
        if self.name and self.base_path and self.name not in initial_base_paths:
            initial_base_paths[self.name] = self.base_path

        # Prefix self's context keys with bundle name to avoid collisions during compose
        initial_context: dict[str, Path] = {}
        for key, path in self.context.items():
            if self.name and ":" not in key:
                prefixed_key = f"{self.name}:{key}"
            else:
                prefixed_key = key
            initial_context[prefixed_key] = path

        # Copy pending context (already has namespace prefixes from _parse_context)
        initial_pending_context: dict[str, str] = (
            dict(self._pending_context) if self._pending_context else {}
        )

        result = Bundle(
            name=self.name,
            version=self.version,
            description=self.description,
            includes=list(self.includes),
            session=dict(self.session),
            providers=list(self.providers),
            tools=list(self.tools),
            hooks=list(self.hooks),
            agents=dict(self.agents),
            context=initial_context,
            _pending_context=initial_pending_context,
            instruction=self.instruction,
            base_path=self.base_path,
            source_base_paths=initial_base_paths,
        )

        for other in others:
            # Merge other's source_base_paths first (preserves registry-set values like source_root)
            # This is critical for subdirectory bundles where registry sets source_root mapping
            if other.source_base_paths:
                for ns, path in other.source_base_paths.items():
                    if ns not in result.source_base_paths:
                        result.source_base_paths[ns] = path

            # Also track other's own namespace as fallback (if not already set via source_base_paths)
            if (
                other.name
                and other.base_path
                and other.name not in result.source_base_paths
            ):
                result.source_base_paths[other.name] = other.base_path

            # Metadata: later wins
            result.name = other.name or result.name
            result.version = other.version or result.version
            if other.description:
                result.description = other.description

            # Session: deep merge
            result.session = deep_merge(result.session, other.session)

            # Module lists: merge by module ID
            result.providers = merge_module_lists(result.providers, other.providers)
            result.tools = merge_module_lists(result.tools, other.tools)
            result.hooks = merge_module_lists(result.hooks, other.hooks)

            # Agents: later overrides
            result.agents.update(other.agents)

            # Context: accumulate with bundle prefix to avoid collisions
            # This allows multiple bundles to each contribute context files
            for key, path in other.context.items():
                # Add bundle prefix if not already present
                if other.name and ":" not in key:
                    prefixed_key = f"{other.name}:{key}"
                else:
                    prefixed_key = key
                result.context[prefixed_key] = path

            # Pending context: accumulate (already has namespace prefixes)
            if other._pending_context:
                result._pending_context.update(other._pending_context)

            # Instruction: later replaces
            if other.instruction:
                result.instruction = other.instruction

            # Base path: use other's if set
            if other.base_path:
                result.base_path = other.base_path

        return result
```

The merge strategy per section:

| Section | Strategy | Rationale |
|---|---|---|
| `session` | `deep_merge()` | Nested config like `orchestrator.config.extended_thinking` should merge recursively |
| `providers`, `tools`, `hooks` | `merge_module_lists()` | Same module ID in both → deep merge their configs; different IDs → both kept |
| `agents` | `dict.update()` | Later completely replaces earlier for same agent name |
| `context` | Accumulate with namespace prefix | Keys get prefixed as `bundlename:key` to avoid collisions across bundles |
| `instruction` | Later replaces | Only one system instruction can be active |
| `source_base_paths` | Accumulate (first-write wins) | Each namespace's path is locked in when first seen |

The `deep_merge()` and `merge_module_lists()` helpers are defined in `dicts/merge.py`:

```bash
cat amplifier_foundation/dicts/merge.py
```

```output
"""Deep merge utilities for dictionaries and module lists."""

from __future__ import annotations

from typing import Any


def deep_merge(parent: dict[str, Any], child: dict[str, Any]) -> dict[str, Any]:
    """Deep merge two dictionaries.

    Child values override parent values. For nested dicts, merge recursively.
    For other types (including lists), child replaces parent.

    Args:
        parent: Base dictionary.
        child: Override dictionary.

    Returns:
        Merged dictionary (new dict, inputs not modified).
    """
    result = parent.copy()

    for key, child_value in child.items():
        if key in result:
            parent_value = result[key]
            # Only deep merge if both are dicts
            if isinstance(parent_value, dict) and isinstance(child_value, dict):
                result[key] = deep_merge(parent_value, child_value)
            else:
                result[key] = child_value
        else:
            result[key] = child_value

    return result


def merge_module_lists(
    parent: list[dict[str, Any]],
    child: list[dict[str, Any]],
) -> list[dict[str, Any]]:
    """Merge two lists of module configs by module ID.

    Module configs are dicts with a 'module' key as identifier.
    If both lists have config for the same module ID, deep merge them
    (child overrides parent).

    Args:
        parent: Base list of module configs.
        child: Override list of module configs.

    Returns:
        Merged list of module configs (new list).
    """
    # Index parent configs by module ID
    by_id: dict[str, dict[str, Any]] = {}
    for config in parent:
        module_id = config.get("module")
        if module_id:
            by_id[module_id] = config.copy()

    # Process child configs
    for config in child:
        module_id = config.get("module")
        if not module_id:
            continue

        if module_id in by_id:
            # Deep merge with existing
            by_id[module_id] = deep_merge(by_id[module_id], config)
        else:
            # Add new module
            by_id[module_id] = config.copy()

    return list(by_id.values())
```

`deep_merge()` recursively merges nested dicts (child overrides parent); non-dict values are simply replaced. `merge_module_lists()` indexes modules by their `module` key and deep-merges entries with the same ID — this is how a provider bundle can override a model setting without replacing the entire provider config.

## 8. Module Activation — From URIs to Importable Python

After a bundle is loaded and composed, calling `bundle.prepare()` activates all referenced modules. The `ModuleActivator` class handles the three-step process: download, install dependencies, add to sys.path.

```bash
sed -n '64,97p' amplifier_foundation/modules/activator.py
```

```output
    async def activate(self, module_name: str, source_uri: str) -> Path:
        """Activate a module by downloading and making it importable.

        Args:
            module_name: Name of the module (e.g., "loop-streaming").
            source_uri: URI to download from (e.g., "git+https://...").

        Returns:
            Local path to the activated module.

        Raises:
            ModuleActivationError: If activation fails.
        """
        # Skip if already activated this session
        cache_key = f"{module_name}:{source_uri}"
        if cache_key in self._activated:
            resolved = await self._resolver.resolve(source_uri)
            return resolved.active_path

        # Download module source
        resolved = await self._resolver.resolve(source_uri)
        module_path = resolved.active_path

        # Install dependencies if requested
        if self.install_deps:
            await self._install_dependencies(module_path)

        # Add to sys.path if not already there
        path_str = str(module_path)
        if path_str not in sys.path:
            sys.path.insert(0, path_str)

        self._activated.add(cache_key)
        return module_path
```

Each module activation:
1. Checks if already activated this session (deduplicated by `module_name:source_uri`)
2. Resolves the source URI to a local path (same `SimpleSourceResolver` used for bundles)
3. Installs dependencies via `uv pip install -e` (editable install targeting current Python)
4. Inserts the module path at the front of `sys.path` so it can be imported

The `activate_all()` method parallelizes this across all modules:

```bash
sed -n '108,139p' amplifier_foundation/modules/activator.py
```

```output
    async def activate_all(self, modules: list[dict]) -> dict[str, Path]:
        """Activate multiple modules with parallelization.

        Args:
            modules: List of module specs with 'module' and 'source' keys.

        Returns:
            Dict mapping module names to their local paths.
        """
        # Phase 1: Resolve all sources and check install state
        to_activate = []
        for mod in modules:
            module_name = mod.get("module")
            source_uri = mod.get("source")
            if not module_name or not source_uri:
                continue
            to_activate.append((module_name, source_uri))

        # Phase 2: Parallel activation
        if to_activate:
            tasks = [self.activate(name, uri) for name, uri in to_activate]
            results = await asyncio.gather(*tasks, return_exceptions=True)

            activated = {}
            for (name, _), result in zip(to_activate, results):
                if isinstance(result, Exception):
                    logger.error(f"Failed to activate {name}: {result}")
                else:
                    activated[name] = result
            return activated

        return {}
```

There's also a special `activate_bundle_package()` method that installs a bundle's own Python package before activating its modules. This handles the case where modules import from their parent bundle's package (e.g., a tool module importing from `amplifier_bundle_shadow`).

The `InstallStateManager` in `modules/install_state.py` tracks which modules have been installed by fingerprinting their path content. This avoids redundant `uv pip install` calls on subsequent startups.

## 9. The @Mention System — Semantic Context References

The `@mention` system enables bundle instructions to reference context files using human-readable syntax like `@foundation:context/AGENTS.md`. It has four components: **parser**, **resolver**, **loader**, and **deduplicator**.

### Parser — Extracting Mentions from Text

```bash
cat amplifier_foundation/mentions/parser.py
```

```output
"""@mention extraction from text."""

from __future__ import annotations

import re


def parse_mentions(text: str) -> list[str]:
    """Extract @mentions from text, excluding code blocks.

    Finds patterns like:
    - @bundle:context-name
    - @path/to/file
    - @./relative/path

    Excludes mentions inside:
    - Inline code (`...`)
    - Fenced code blocks (```...```)

    Args:
        text: Text to extract mentions from.

    Returns:
        List of unique mentions (including @ prefix).
    """
    # Remove code blocks first
    text_without_code = _remove_code_blocks(text)

    # Find @mentions
    # Pattern: @ followed by word chars, colons, slashes, dots, hyphens
    # But not email addresses (no @ followed by domain pattern)
    pattern = r"@(?![a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})([a-zA-Z0-9_:./\-]+)"

    matches = re.findall(pattern, text_without_code)

    # Return unique mentions with @ prefix, preserving order
    seen: set[str] = set()
    result: list[str] = []
    for match in matches:
        mention = f"@{match}"
        if mention not in seen:
            seen.add(mention)
            result.append(mention)

    return result


def _remove_code_blocks(text: str) -> str:
    """Remove code blocks from text.

    Removes:
    - Fenced code blocks (```...```)
    - Inline code (`...`)
    """
    # Remove fenced code blocks (including language identifier)
    text = re.sub(r"```[^\n]*\n.*?```", "", text, flags=re.DOTALL)

    # Remove inline code
    text = re.sub(r"`[^`]+`", "", text)

    return text
```

The parser strips code blocks first (so `@mentions` inside code examples aren't resolved), then uses a regex to find `@`-prefixed tokens. It explicitly excludes email addresses with a negative lookahead.

### Resolver — Converting Mentions to File Paths

```bash
sed -n '12,69p' amplifier_foundation/mentions/resolver.py
```

```output
class BaseMentionResolver:
    """Base implementation of MentionResolverProtocol.

    Supports two patterns:
    - @bundle-name:context-name - From bundle's context namespace
    - @path - Relative to current working directory (CWD)

    Apps extend this class to add shortcuts like @user:, @project:.
    """

    def __init__(
        self,
        bundles: dict[str, Bundle] | None = None,
        base_path: Path | None = None,  # Kept for API compatibility; not used for @path
    ) -> None:
        """Initialize resolver.

        Args:
            bundles: Dict mapping bundle names to Bundle instances.
            base_path: Unused (kept for API compatibility). Relative @path
                       mentions always resolve against CWD.
        """
        self.bundles = bundles or {}
        self.base_path = base_path or Path.cwd()  # Stored but not used for @path

    def resolve(self, mention: str) -> Path | None:
        """Resolve an @mention to a file path.

        Args:
            mention: The mention string (including @ prefix).

        Returns:
            Path to the resolved file, or None if not found.
        """
        if not mention.startswith("@"):
            return None

        mention_body = mention[1:]  # Remove @ prefix

        # Pattern 1: @bundle-name:context-name
        if ":" in mention_body:
            namespace, name = mention_body.split(":", 1)
            if bundle := self.bundles.get(namespace):
                return bundle.resolve_context_path(name)
            return None

        # Pattern 2: @path (relative to CWD for user-local files)
        cwd = Path.cwd()
        path = cwd / mention_body
        if path.exists():
            return path

        # Try with .md extension
        path_md = cwd / f"{mention_body}.md"
        if path_md.exists():
            return path_md

        return None
```

Two resolution patterns:

1. **`@namespace:path`** — Look up the namespace in the bundles dict, then call `bundle.resolve_context_path(name)` which checks the bundle's registered context files and constructs paths relative to the bundle's `base_path`.
2. **`@path`** — Simple CWD-relative lookup, with automatic `.md` extension fallback.

### Loader — Recursive Loading with Deduplication

```bash
sed -n '78,199p' amplifier_foundation/mentions/loader.py
```

```output
async def load_mentions(
    text: str,
    resolver: MentionResolverProtocol,
    deduplicator: ContentDeduplicator | None = None,
    relative_to: Path | None = None,
    max_depth: int = 3,
) -> list[MentionResult]:
    """Load @mentioned files recursively with deduplication.

    All mentions are opportunistic - if a file can't be found, it's
    silently skipped (no error raised).

    Args:
        text: Text containing @mentions.
        resolver: Resolver to convert mentions to paths.
        deduplicator: Optional deduplicator for content. If None, creates one.
        relative_to: Base path for relative mentions (defaults to cwd).
        max_depth: Maximum recursion depth to prevent infinite loops (default 3).

    Returns:
        List of MentionResult for each mention found.
    """
    if deduplicator is None:
        deduplicator = ContentDeduplicator()

    results: list[MentionResult] = []
    mentions = parse_mentions(text)

    for mention in mentions:
        result = await _resolve_mention(
            mention=mention,
            resolver=resolver,
            deduplicator=deduplicator,
            relative_to=relative_to,
            max_depth=max_depth,
            current_depth=0,
        )
        results.append(result)

    return results


async def _resolve_mention(
    mention: str,
    resolver: MentionResolverProtocol,
    deduplicator: ContentDeduplicator,
    relative_to: Path | None,
    max_depth: int,
    current_depth: int,
) -> MentionResult:
    """Resolve a single mention and recursively load its mentions."""
    # Resolve mention to path
    path = resolver.resolve(mention)
    if path is None:
        return MentionResult(
            mention=mention,
            resolved_path=None,
            content=None,
            error=None,  # Opportunistic - no error for not found
        )

    # Handle directories: generate listing as content
    if path.is_dir():
        try:
            content = format_directory_listing(path)
            deduplicator.add_file(path, content)
            return MentionResult(
                mention=mention,
                resolved_path=path,
                content=content,
                error=None,
                is_directory=True,
            )
        except (PermissionError, OSError):
            # Can't list directory - return without content
            return MentionResult(
                mention=mention,
                resolved_path=path,
                content=None,
                error=None,
                is_directory=True,
            )

    # Read file
    try:
        content = await read_with_retry(path)
    except (FileNotFoundError, OSError):
        return MentionResult(
            mention=mention,
            resolved_path=path,
            content=None,
            error=None,  # Opportunistic - no error for read failure
        )

    # Check for duplicate content
    if not deduplicator.add_file(path, content):
        return MentionResult(
            mention=mention,
            resolved_path=path,
            content=None,  # Already seen, don't include again
            error=None,
        )

    # Recursively load mentions from this file (if not at max depth)
    if current_depth < max_depth:
        nested_mentions = parse_mentions(content)
        for nested in nested_mentions:
            await _resolve_mention(
                mention=nested,
                resolver=resolver,
                deduplicator=deduplicator,
                relative_to=path.parent,
                max_depth=max_depth,
                current_depth=current_depth + 1,
            )

    return MentionResult(
        mention=mention,
        resolved_path=path,
        content=content,
        error=None,
    )
```

Key behavior:

- **Opportunistic**: If a mention can't be found, it's silently skipped (no error, just `None` content). This is intentional — instructions may reference files that don't exist yet.
- **Recursive**: After loading a file, its content is scanned for nested `@mentions`, which are resolved up to `max_depth=3` levels deep.
- **Deduplicated**: The `ContentDeduplicator` tracks content by SHA-256 hash. If the same content appears at multiple paths, it's only included once — but all paths are tracked for attribution.
- **Directory support**: `@mention` pointing to a directory generates a listing of its contents.

### Deduplicator — Content-Addressed Tracking

```bash
sed -n '11,65p' amplifier_foundation/mentions/deduplicator.py
```

```output
class ContentDeduplicator:
    """Deduplicate content by SHA-256 hash with multi-path attribution.

    Tracks files that have been added and returns only unique content.
    When the same content is found at multiple paths, all paths are tracked
    so users/models know all @mentions that resolved to this content.

    Useful when loading recursive @mentions to avoid including
    the same content multiple times while crediting all sources.
    """

    def __init__(self) -> None:
        """Initialize deduplicator."""
        self._content_by_hash: dict[str, str] = {}
        self._paths_by_hash: dict[str, list[Path]] = {}

    def add_file(self, path: Path, content: str) -> bool:
        """Add a file, tracking its path even if content is duplicate.

        Args:
            path: Path to the file.
            content: File content.

        Returns:
            True if file was added (new content), False if duplicate content
            (but path is still tracked for attribution).
        """
        content_hash = hashlib.sha256(content.encode()).hexdigest()

        if content_hash not in self._content_by_hash:
            # New content
            self._content_by_hash[content_hash] = content
            self._paths_by_hash[content_hash] = [path]
            return True

        # Duplicate content - add path if not already tracked
        if path not in self._paths_by_hash[content_hash]:
            self._paths_by_hash[content_hash].append(path)
        return False

    def get_unique_files(self) -> list[ContextFile]:
        """Get list of unique files with all paths where each was found.

        Returns:
            List of ContextFile instances, one per unique content,
            each with all paths where that content was found.
        """
        return [
            ContextFile(
                content=content,
                content_hash=content_hash,
                paths=self._paths_by_hash[content_hash],
            )
            for content_hash, content in self._content_by_hash.items()
        ]
```

The deduplicator uses two parallel dictionaries indexed by SHA-256: one for content, one for paths. When `add_file()` encounters duplicate content, it returns `False` (signaling the loader to skip it) but still records the additional path. The `get_unique_files()` method then produces `ContextFile` instances with all paths where each unique piece of content was found.

### Context Block Formatting

The final step is `format_context_block()`, which wraps loaded context into XML blocks prepended to the system prompt:

```bash
sed -n '16,75p' amplifier_foundation/mentions/loader.py
```

```output
def format_context_block(
    deduplicator: ContentDeduplicator,
    mention_to_path: dict[str, Path] | None = None,
) -> str:
    """Format all loaded files as XML context blocks for prepending.

    Creates XML-wrapped context blocks that the LLM sees BEFORE the instruction.
    The @mentions in the original instruction remain as semantic references.

    Args:
        deduplicator: Deduplicator containing loaded context files.
        mention_to_path: Optional mapping from @mention strings to resolved paths,
            used to show both @mention and absolute path in XML attributes.

    Returns:
        Formatted context string with XML blocks, or empty string if no files.

    Example output:
        <context_file paths="@AGENTS.md → /home/user/project/AGENTS.md">
        [file content here]
        </context_file>

        <context_file paths="@foundation:context/KERNEL.md → /path/to/KERNEL.md">
        [file content here]
        </context_file>
    """
    unique_files = deduplicator.get_unique_files()
    if not unique_files:
        return ""

    # Build reverse lookup: path -> mention(s) for attribution
    path_to_mentions: dict[Path, list[str]] = {}
    if mention_to_path:
        for mention, path in mention_to_path.items():
            resolved = path.resolve()
            if resolved not in path_to_mentions:
                path_to_mentions[resolved] = []
            path_to_mentions[resolved].append(mention)

    blocks = []
    for cf in unique_files:
        # Build paths attribute showing @mention → absolute path for ALL paths
        # (ContextFile now tracks multiple paths where same content was found)
        path_displays = []
        for p in cf.paths:
            resolved = p.resolve()
            mentions = path_to_mentions.get(resolved, [])
            if mentions:
                # Show each @mention with its resolved path
                for m in mentions:
                    path_displays.append(f"{m} → {resolved}")
            else:
                # No @mention tracked, just show path
                path_displays.append(str(resolved))

        paths_attr = ", ".join(path_displays)
        block = f'<context_file paths="{paths_attr}">\n{cf.content}\n</context_file>'
        blocks.append(block)

    return "\n\n".join(blocks)
```

Each unique context file becomes an XML block with a `paths` attribute showing the `@mention → /absolute/path` mapping. These blocks are prepended to the system prompt, giving the LLM all referenced context before it sees the instruction.

## 10. Session Management — Fork, Slice, and Tracing

Foundation provides utilities for managing session lifecycles: forking sessions at specific turns, slicing conversation histories, and generating traceable sub-session IDs.

### Tracing — W3C-Compatible Sub-Session IDs

```bash
cat amplifier_foundation/tracing.py
```

```output
"""Tracing utilities for amplifier-foundation.

Provides W3C-compatible trace context ID generation for sub-agent
tracing. Apps decide WHEN to create sub-sessions - this module
provides the HOW for generating traceable IDs.

Based on app-cli's battle-tested `_generate_sub_session_id()` from
session_spawner.py, which follows W3C Trace Context principles.

Philosophy: Mechanism not policy. Apps decide when to spawn sub-sessions.
"""

from __future__ import annotations

import re
import uuid

# W3C Trace Context uses 16 hex chars (8 bytes) for span IDs
_SPAN_HEX_LEN = 16
_DEFAULT_PARENT_SPAN = "0" * _SPAN_HEX_LEN

# Pattern to extract parent/child spans from sub-session IDs
_SPAN_PATTERN = re.compile(r"^([0-9a-f]{16})-([0-9a-f]{16})_")
_TRACE_ID_PATTERN = re.compile(r"^[0-9a-f]{32}$")


def generate_sub_session_id(
    agent_name: str | None = None,
    parent_session_id: str | None = None,
    parent_trace_id: str | None = None,
) -> str:
    """Generate a sub-session ID with W3C Trace Context lineage.

    Creates hierarchical IDs that can be traced back to parent sessions
    following W3C Trace Context principles:
    - Parent span ID (16 hex chars) extracted from parent session or trace
    - New child span ID (16 hex chars) for this session
    - Agent name suffix for readability (sanitized for filesystem safety)

    Format: {parent-span}-{child-span}_{agent-name}
    Example: 1234567890abcdef-fedcba0987654321_zen-architect

    Based on app-cli's battle-tested implementation in session_spawner.py.

    Args:
        agent_name: Name of the sub-agent (for human readability)
        parent_session_id: Parent session's ID (for span extraction)
        parent_trace_id: Parent trace ID if using distributed tracing

    Returns:
        Sub-session ID with embedded trace context

    Example:
        # With parent context
        sub_id = generate_sub_session_id(
            agent_name="researcher",
            parent_session_id="abc123def456-7890abcdef123456_planner",
        )
        # "7890abcdef123456-fedcba0987654321_researcher"

        # First-level sub-session (no parent span)
        sub_id = generate_sub_session_id(agent_name="analyzer")
        # "0000000000000000-fedcba0987654321_analyzer"

        # Using trace ID for parent span
        sub_id = generate_sub_session_id(
            agent_name="worker",
            parent_trace_id="12345678901234567890123456789012",
        )
        # "3456789012345678-fedcba0987654321_worker"
    """
    # Sanitize agent name for filesystem safety
    raw_name = (agent_name or "").lower()

    # Replace any non-alphanumeric characters with hyphens
    sanitized = re.sub(r"[^a-z0-9]+", "-", raw_name)
    # Collapse multiple hyphens
    sanitized = re.sub(r"-{2,}", "-", sanitized)
    # Remove leading/trailing hyphens and dots
    sanitized = sanitized.strip("-").lstrip(".")

    # Default to "agent" if empty after sanitization
    if not sanitized:
        sanitized = "agent"

    # Extract parent span ID following W3C Trace Context principles
    parent_span = _DEFAULT_PARENT_SPAN

    if parent_session_id:
        # If parent has our format, extract its child span (becomes our parent span)
        match = _SPAN_PATTERN.match(parent_session_id)
        if match:
            # Extract the child span from parent (second group)
            parent_span = match.group(2)

    # If no parent span found and we have a trace ID, derive parent span from trace
    # Extract middle 16 chars (positions 8-24) from 32-char trace ID
    if parent_span == _DEFAULT_PARENT_SPAN and parent_trace_id and _TRACE_ID_PATTERN.fullmatch(parent_trace_id):
        # Take middle 16 characters (8-24) of the 32-char trace ID
        parent_span = parent_trace_id[8:24]

    # Generate new span ID for this child session
    child_span = uuid.uuid4().hex[:_SPAN_HEX_LEN]

    return f"{parent_span}-{child_span}_{sanitized}"
```

The ID format is `{parent-span}-{child-span}_{agent-name}`. When spawning a hierarchy of sub-agents, each child extracts the parent's child span as its own parent span, creating a traceable chain. The agent name is sanitized for filesystem safety (used as directory names for session storage).

### Turn Slicing

The `slice_to_turn()` function in `session/slice.py` cuts a conversation at a specific turn boundary. A "turn" is defined as a user message plus all subsequent non-user messages:

```bash
sed -n '52,125p' amplifier_foundation/session/slice.py
```

```output
def slice_to_turn(
    messages: list[dict[str, Any]],
    turn: int,
    *,
    handle_orphaned_tools: str = "complete",
) -> list[dict[str, Any]]:
    """Slice messages to include only up to turn N (1-indexed).

    Turn N includes the Nth user message and all responses until the
    next user message (or end of conversation).

    Args:
        messages: Full message list from transcript.
        turn: Turn number (1-indexed). Turn 1 = first user message + response.
        handle_orphaned_tools: How to handle tool_use without tool_result:
            - "complete": Add synthetic error result (default)
            - "remove": Remove the orphaned tool_use content
            - "error": Raise ValueError

    Returns:
        Sliced message list with orphaned tools handled.

    Raises:
        ValueError: If turn < 1 or turn > max_turns, or if handle_orphaned_tools
            is "error" and orphaned tools are found.

    Example:
        >>> messages = [
        ...     {"role": "user", "content": "Q1"},
        ...     {"role": "assistant", "content": "A1"},
        ...     {"role": "user", "content": "Q2"},
        ...     {"role": "assistant", "content": "A2"},
        ... ]
        >>> sliced = slice_to_turn(messages, 1)
        >>> len(sliced)
        2
    """
    if turn < 1:
        raise ValueError(f"Turn must be >= 1, got {turn}")

    boundaries = get_turn_boundaries(messages)
    max_turns = len(boundaries)

    if max_turns == 0:
        raise ValueError("No user messages found in conversation")

    if turn > max_turns:
        raise ValueError(
            f"Turn {turn} exceeds max turns ({max_turns}). "
            f"Valid range: 1-{max_turns}"
        )

    # Find end index: start of turn N+1, or end of messages
    if turn < max_turns:
        end_idx = boundaries[turn]  # Start of next turn (0-indexed, so turn N+1 = boundaries[turn])
    else:
        end_idx = len(messages)  # Include all messages

    sliced = messages[:end_idx]

    # Handle orphaned tool calls
    orphaned = find_orphaned_tool_calls(sliced)
    if orphaned:
        if handle_orphaned_tools == "error":
            raise ValueError(
                f"Orphaned tool calls at fork boundary: {orphaned}. "
                "These tool_use blocks have no matching tool_result."
            )
        elif handle_orphaned_tools == "remove":
            sliced = _remove_orphaned_tool_calls(sliced, orphaned)
        else:  # "complete" is default
            sliced = add_synthetic_tool_results(sliced, orphaned)

    return sliced
```

Slicing is tricky because cutting mid-turn can leave "orphaned" tool calls — `tool_use` blocks from the assistant with no corresponding `tool_result`. The function handles this with three strategies: add a synthetic error result (default), remove the orphaned calls, or raise an error.

### Session Forking

The `fork_session()` function creates a new session from an existing one at a specific turn, writing a new transcript, metadata (with parent lineage), and sliced events. There's also an `fork_session_in_memory()` variant for testing or in-process forking without filesystem I/O.

## 11. PreparedBundle and Session Creation — Putting It All Together

After loading, composing, and preparing a bundle, the `PreparedBundle` is the final object that can create live sessions. Let's trace the full lifecycle.

### Bundle.prepare() — From Bundle to PreparedBundle

The `prepare()` method on Bundle orchestrates module activation:

```bash
sed -n '213,323p' amplifier_foundation/bundle.py
```

```output
    async def prepare(
        self,
        install_deps: bool = True,
        source_resolver: Callable[[str, str], str] | None = None,
    ) -> PreparedBundle:
        """Prepare bundle for execution by activating all modules.

        Downloads and installs all modules specified in the bundle's mount plan,
        making them importable. Returns a PreparedBundle containing the mount plan
        and a module resolver for use with AmplifierSession.

        This is the turn-key method for apps that want to load a bundle and
        execute it without managing module resolution themselves.

        Args:
            install_deps: Whether to install Python dependencies for modules.
            source_resolver: Optional callback (module_id, original_source) -> resolved_source.
                Allows app-layer source override policy to be applied before activation.
                If provided, each module's source is passed through this resolver,
                enabling settings-based overrides without foundation knowing about settings.

        Returns:
            PreparedBundle with mount_plan and create_session() helper.

        Example:
            bundle = await load_bundle("git+https://...")
            prepared = await bundle.prepare()
            async with prepared.create_session() as session:
                response = await session.execute("Hello!")

            # Or manually:
            session = AmplifierSession(config=prepared.mount_plan)
            await session.coordinator.mount("module-source-resolver", prepared.resolver)
            await session.initialize()

            # With source overrides (app-layer policy):
            def resolve_with_overrides(module_id: str, source: str) -> str:
                return overrides.get(module_id) or source
            prepared = await bundle.prepare(source_resolver=resolve_with_overrides)
        """
        from amplifier_foundation.modules.activator import ModuleActivator

        # Get mount plan
        mount_plan = self.to_mount_plan()

        # Create activator with bundle's base_path so relative module paths
        # like ./modules/foo resolve relative to the bundle, not cwd
        activator = ModuleActivator(install_deps=install_deps, base_path=self.base_path)

        # CRITICAL: Install bundle packages BEFORE activating modules
        # Modules may import from their parent bundle's package (e.g., tool-shadow
        # imports from amplifier_bundle_shadow). These packages must be installed
        # before modules can be activated.
        if install_deps:
            # Install this bundle's package (if it has pyproject.toml)
            if self.base_path:
                await activator.activate_bundle_package(self.base_path)

            # Install packages from all included bundles (from source_base_paths)
            for namespace, bundle_path in self.source_base_paths.items():
                if bundle_path and bundle_path != self.base_path:
                    await activator.activate_bundle_package(bundle_path)

        # Collect all modules that need activation
        modules_to_activate = []

        # Helper to apply source resolver if provided
        def resolve_source(mod_spec: dict) -> dict:
            if source_resolver and "module" in mod_spec and "source" in mod_spec:
                resolved = source_resolver(mod_spec["module"], mod_spec["source"])
                if resolved != mod_spec["source"]:
                    # Copy to avoid mutating original
                    mod_spec = {**mod_spec, "source": resolved}
            return mod_spec

        # Session orchestrator and context
        session_config = mount_plan.get("session", {})
        if isinstance(session_config.get("orchestrator"), dict):
            orch = session_config["orchestrator"]
            if "source" in orch:
                modules_to_activate.append(resolve_source(orch))
        if isinstance(session_config.get("context"), dict):
            ctx = session_config["context"]
            if "source" in ctx:
                modules_to_activate.append(resolve_source(ctx))

        # Providers, tools, hooks
        for section in ["providers", "tools", "hooks"]:
            for mod_spec in mount_plan.get(section, []):
                if isinstance(mod_spec, dict) and "source" in mod_spec:
                    modules_to_activate.append(resolve_source(mod_spec))

        # Activate all modules and get their paths
        module_paths = await activator.activate_all(modules_to_activate)

        # Save install state to disk for fast subsequent startups
        activator.finalize()

        # Create resolver from activated paths with activator for lazy activation
        # This enables child sessions to activate agent-specific modules on-demand
        resolver = BundleModuleResolver(module_paths, activator=activator)

        # Get bundle package paths for inheritance by child sessions
        bundle_package_paths = activator.bundle_package_paths

        return PreparedBundle(
            mount_plan=mount_plan,
            resolver=resolver,
            bundle=self,
            bundle_package_paths=bundle_package_paths,
        )
```

The flow:

1. **Compile mount plan** via `to_mount_plan()` — converts the Bundle's sections into the dict format AmplifierSession expects.
2. **Install bundle packages** — editable-install (`uv pip install -e`) the bundle's own Python package and all included bundles' packages. This must happen *before* activating modules since modules may import from these packages.
3. **Apply source overrides** — if a `source_resolver` callback was provided (app-layer policy), each module's source URI is passed through it for potential override.
4. **Collect modules** — gathers all modules from session (orchestrator, context), providers, tools, and hooks.
5. **Activate all** — parallel download, install, and sys.path insertion.
6. **Create resolver** — wraps the module paths in a `BundleModuleResolver` that the kernel uses to find modules at runtime.

### PreparedBundle.create_session() — The Final Step

```bash
sed -n '790,888p' amplifier_foundation/bundle.py
```

```output
    async def create_session(
        self,
        session_id: str | None = None,
        parent_id: str | None = None,
        approval_system: Any = None,
        display_system: Any = None,
    ) -> Any:
        """Create an AmplifierSession with the resolver properly mounted.

        This is a convenience method that handles the full setup:
        1. Creates AmplifierSession with mount plan
        2. Mounts the module resolver
        3. Initializes the session

        Note: Session spawning capability registration is APP-LAYER policy.
        Apps should register their own spawn capability that adapts the
        task tool's contract to foundation's spawn mechanism. See the
        end_to_end example for a reference implementation.

        Args:
            session_id: Optional session ID (for resuming existing session).
            parent_id: Optional parent session ID (for lineage tracking).
            approval_system: Optional approval system for hooks.
            display_system: Optional display system for hooks.

        Returns:
            Initialized AmplifierSession ready for execute().

        Example:
            prepared = await bundle.prepare()
            async with prepared.create_session() as session:
                response = await session.execute("Hello!")
        """
        from amplifier_core import AmplifierSession

        session = AmplifierSession(
            self.mount_plan,
            session_id=session_id,
            parent_id=parent_id,
            approval_system=approval_system,
            display_system=display_system,
        )

        # Mount the resolver before initialization
        await session.coordinator.mount("module-source-resolver", self.resolver)

        # Register bundle package paths for inheritance by child sessions
        # These are src/ directories from bundles like python-dev that need to be
        # on sys.path for their modules to import shared code
        if self.bundle_package_paths:
            session.coordinator.register_capability(
                "bundle_package_paths", list(self.bundle_package_paths)
            )

        # Initialize the session (loads all modules)
        await session.initialize()

        # Resolve any pending namespaced context references now that source_base_paths is available
        self.bundle.resolve_pending_context()

        # Register system prompt factory for dynamic @mention reprocessing
        # The factory is called on EVERY get_messages_for_request(), enabling:
        # - AGENTS.md changes to be picked up immediately
        # - Bundle instruction changes to take effect mid-session
        # - All @mentioned files to be re-read fresh each turn
        if (
            self.bundle.instruction
            or self.bundle.context
            or self.bundle._pending_context
        ):
            from amplifier_foundation.mentions import BaseMentionResolver
            from amplifier_foundation.mentions import ContentDeduplicator

            # Register resolver and deduplicator as capabilities for tools to use
            # (e.g., filesystem tool's read_file can resolve @mention paths)
            # Note: These are created once for capability registration, but the factory
            # creates fresh instances each call for accurate file re-reading
            bundles_for_resolver = self._build_bundles_for_resolver(self.bundle)
            initial_resolver = BaseMentionResolver(
                bundles=bundles_for_resolver,
                base_path=self.bundle.base_path or Path.cwd(),
            )
            initial_deduplicator = ContentDeduplicator()
            session.coordinator.register_capability(
                "mention_resolver", initial_resolver
            )
            session.coordinator.register_capability(
                "mention_deduplicator", initial_deduplicator
            )

            # Create and register the system prompt factory
            factory = self._create_system_prompt_factory(self.bundle, session)
            context_manager = session.coordinator.get("context")
            if context_manager and hasattr(
                context_manager, "set_system_prompt_factory"
            ):
                await context_manager.set_system_prompt_factory(factory)

        return session
```

The session creation sequence:

1. **Create AmplifierSession** with the compiled mount plan.
2. **Mount the module resolver** so the kernel can find and import modules during initialization.
3. **Register bundle package paths** so child sessions can inherit them.
4. **Initialize** — this is where the kernel loads all providers, tools, hooks, orchestrator, and context modules.
5. **Resolve pending context** — namespaced context refs that couldn't be resolved during parsing are now resolved using the fully-populated `source_base_paths`.
6. **Register system prompt factory** — a closure that is called on *every* `get_messages_for_request()`. This is critical: it re-reads all context files and re-processes `@mentions` every turn, enabling live updates. If you edit `AGENTS.md` mid-session, the change is picked up on the very next turn.

### Spawning Sub-Sessions

`PreparedBundle.spawn()` creates child sessions with composed bundles. The parent's bundle is composed with the child bundle (parent provides tools/providers, child provides the specialized instruction and agents), a new `AmplifierSession` is created, parent messages can optionally be injected for context inheritance, and the instruction is executed. After execution, the child session is cleaned up and the result returned.

## 12. Supporting Infrastructure

### Exceptions

```bash
cat amplifier_foundation/exceptions.py
```

```output
"""Exception hierarchy for amplifier-foundation."""


class BundleError(Exception):
    """Base exception for all bundle-related errors."""


class BundleNotFoundError(BundleError):
    """Bundle could not be located at the specified source."""


class BundleLoadError(BundleError):
    """Bundle exists but could not be loaded (parse error, invalid format)."""


class BundleValidationError(BundleError):
    """Bundle loaded but validation failed (missing required fields, etc)."""


class BundleDependencyError(BundleError):
    """Bundle dependency could not be resolved (circular deps, missing deps)."""
```

A clean four-leaf hierarchy rooted in `BundleError`. Each exception represents a distinct failure mode in the bundle lifecycle: not found, can't parse, invalid structure, or dependency issue.

### Validation

```bash
sed -n '40,76p' amplifier_foundation/validator.py
```

```output
class BundleValidator:
    """Validates bundle structure and configuration.

    Validates:
    - Required fields (name)
    - Module list structure
    - Session configuration
    - Resource references

    Apps may extend for additional validation rules.
    """

    def validate(self, bundle: Bundle) -> ValidationResult:
        """Validate a bundle.

        Args:
            bundle: Bundle to validate.

        Returns:
            ValidationResult with errors and warnings.
        """
        result = ValidationResult()

        # Required fields
        self._validate_required_fields(bundle, result)

        # Module lists
        self._validate_module_lists(bundle, result)

        # Session config
        self._validate_session(bundle, result)

        # Resources
        self._validate_resources(bundle, result)

        return result

```

Validation checks required fields, module list structure (each entry must have a `module` key), session config format, and resource references. There's also a `validate_completeness()` method for "mountable" bundles that additionally requires a session orchestrator, context module, and at least one provider.

### Serialization — JSON Sanitization

```bash
sed -n '18,88p' amplifier_foundation/serialization.py
```

```output
def sanitize_for_json(value: Any, *, max_depth: int = 50) -> Any:
    """Recursively sanitize a value to ensure it's JSON-serializable.

    Handles common cases from LLM responses:
    - Non-serializable objects (returns None or extracts useful text)
    - Nested dicts and lists
    - Objects with __dict__

    Based on app-cli's `_sanitize_value()` pattern.

    Args:
        value: Any value that may or may not be serializable
        max_depth: Maximum recursion depth (prevents infinite loops)

    Returns:
        Sanitized value that's JSON-serializable

    Example:
        # Sanitize LLM response for persistence
        clean_response = sanitize_for_json(llm_response)
        json.dumps(clean_response)  # Now safe
    """
    if max_depth <= 0:
        return None

    # Handle None and primitives (always serializable)
    if value is None or isinstance(value, (bool, int, float, str)):
        return value

    # Handle dicts recursively
    if isinstance(value, dict):
        return {
            k: sanitize_for_json(v, max_depth=max_depth - 1)
            for k, v in value.items()
            if sanitize_for_json(v, max_depth=max_depth - 1) is not None
        }

    # Handle lists recursively
    if isinstance(value, list):
        sanitized = []
        for item in value:
            clean_item = sanitize_for_json(item, max_depth=max_depth - 1)
            if clean_item is not None:
                sanitized.append(clean_item)
        return sanitized

    # Handle tuples (convert to list)
    if isinstance(value, tuple):
        return sanitize_for_json(list(value), max_depth=max_depth - 1)

    # Try objects with __dict__ (like Pydantic models)
    if hasattr(value, "__dict__"):
        try:
            return sanitize_for_json(vars(value), max_depth=max_depth - 1)
        except Exception:
            pass

    # Try model_dump for Pydantic v2
    if hasattr(value, "model_dump"):
        try:
            return sanitize_for_json(value.model_dump(), max_depth=max_depth - 1)
        except Exception:
            pass

    # Last resort: try to serialize directly
    try:
        json.dumps(value)
        return value
    except (TypeError, ValueError):
        logger.debug(f"Skipping non-serializable value of type {type(value).__name__}")
        return None
```

LLM API responses often contain non-serializable objects (Pydantic models, thinking blocks, custom API objects). `sanitize_for_json()` recursively walks any value, extracting serializable data from `__dict__` or `model_dump()`, and silently dropping anything that can't be JSON-encoded. The companion `sanitize_message()` adds special handling for known problematic fields like `thinking_block` (extracts text) and `content_blocks` (skips entirely).

### File I/O — Cloud-Sync Safe Operations

```bash
sed -n '23,66p' amplifier_foundation/io/files.py
```

```output
async def read_with_retry(
    path: Path,
    max_retries: int = 3,
    initial_delay: float = 0.1,
) -> str:
    """Read file content with retry logic for cloud sync delays.

    OneDrive, Dropbox, and Google Drive can cause transient I/O errors
    when files aren't locally cached. This function automatically retries
    with exponential backoff.

    Args:
        path: Path to file to read.
        max_retries: Maximum number of retry attempts.
        initial_delay: Initial delay in seconds before first retry.

    Returns:
        File content as string.

    Raises:
        FileNotFoundError: If file doesn't exist.
        OSError: If file can't be read after all retries.
    """
    delay = initial_delay
    last_error: OSError | None = None

    for attempt in range(max_retries):
        try:
            return path.read_text(encoding="utf-8")
        except OSError as e:
            last_error = e
            if e.errno == 5 and attempt < max_retries - 1:
                if attempt == 0:
                    logger.warning(
                        f"File I/O error reading {path} - retrying. "
                        "This may be due to cloud-synced files (OneDrive, Dropbox, etc.). "
                        "Consider enabling 'Always keep on this device' for the data folder."
                    )
                await asyncio.sleep(delay)
                delay *= 2
            else:
                raise

    raise last_error  # type: ignore[misc]
```

OneDrive, Dropbox, and Google Drive can cause `errno 5` (I/O error) when files aren't locally cached. `read_with_retry()` retries with exponential backoff (0.1s, 0.2s, 0.4s). The `write_with_backup()` function uses an atomic temp-file-plus-rename pattern so files are never partially written — crucial for session state that must survive crashes.

### Dict Navigation

```bash
cat amplifier_foundation/dicts/navigation.py
```

```output
"""Dictionary navigation utilities."""

from __future__ import annotations

from typing import Any


def get_nested(
    data: dict[str, Any],
    path: list[str],
    default: Any = None,
) -> Any:
    """Get a value from a nested dictionary by path.

    Args:
        data: Dictionary to navigate.
        path: List of keys to traverse.
        default: Value to return if path not found.

    Returns:
        Value at path, or default if not found.

    Example:
        >>> get_nested({'a': {'b': {'c': 1}}}, ['a', 'b', 'c'])
        1
        >>> get_nested({'a': 1}, ['x', 'y'], default='not found')
        'not found'
    """
    current = data
    for key in path:
        if not isinstance(current, dict):
            return default
        if key not in current:
            return default
        current = current[key]
    return current


def set_nested(
    data: dict[str, Any],
    path: list[str],
    value: Any,
) -> None:
    """Set a value in a nested dictionary by path.

    Creates intermediate dicts as needed.

    Args:
        data: Dictionary to modify (modified in place).
        path: List of keys to traverse.
        value: Value to set at path.

    Example:
        >>> d = {}
        >>> set_nested(d, ['a', 'b', 'c'], 1)
        >>> d
        {'a': {'b': {'c': 1}}}
    """
    if not path:
        return

    current = data
    for key in path[:-1]:
        if key not in current or not isinstance(current[key], dict):
            current[key] = {}
        current = current[key]

    current[path[-1]] = value
```

`get_nested()` and `set_nested()` provide safe traversal of deeply nested dicts — the kind that show up everywhere in mount plan configurations. `set_nested()` creates intermediate dicts as needed, making it easy to set deep values without checking each level.

## 13. The Public API — What Gets Exported

The package's `__init__.py` curates a clean public API, organized by category:

```bash
cat amplifier_foundation/__init__.py
```

```output
"""Amplifier Foundation - Bundle composition mechanism layer.

Foundation provides an ultra-thin mechanism layer for bundle composition
that sits between amplifier-core (kernel) and applications.

Core concept: Bundle = composable unit that produces mount plans.

One mechanism: `includes:` (declarative) + `compose()` (imperative)

Philosophy: Mechanism not policy, ruthless simplicity.

Note: This library is PURE MECHANISM. It loads bundles from URIs without
knowing about any specific bundle (including "foundation"). The foundation
bundle content co-located in this repo is just content - it's discovered
and loaded the same way any other bundle would be.
"""

from __future__ import annotations

# Core classes
from amplifier_foundation.bundle import Bundle

# Reference implementations
from amplifier_foundation.cache.disk import DiskCache

# Protocols
from amplifier_foundation.cache.protocol import CacheProviderProtocol
from amplifier_foundation.cache.simple import SimpleCache

# Dict utilities
from amplifier_foundation.dicts.merge import deep_merge
from amplifier_foundation.dicts.merge import merge_module_lists
from amplifier_foundation.dicts.navigation import get_nested
from amplifier_foundation.dicts.navigation import set_nested

# Exceptions
from amplifier_foundation.exceptions import BundleDependencyError
from amplifier_foundation.exceptions import BundleError
from amplifier_foundation.exceptions import BundleLoadError
from amplifier_foundation.exceptions import BundleNotFoundError
from amplifier_foundation.exceptions import BundleValidationError

# I/O utilities
from amplifier_foundation.io.files import read_with_retry
from amplifier_foundation.io.files import write_with_backup
from amplifier_foundation.io.files import write_with_retry
from amplifier_foundation.io.frontmatter import parse_frontmatter
from amplifier_foundation.io.yaml import read_yaml
from amplifier_foundation.io.yaml import write_yaml

# Mention utilities
from amplifier_foundation.mentions.deduplicator import ContentDeduplicator
from amplifier_foundation.mentions.loader import load_mentions
from amplifier_foundation.mentions.models import ContextFile
from amplifier_foundation.mentions.models import MentionResult
from amplifier_foundation.mentions.parser import parse_mentions
from amplifier_foundation.mentions.protocol import MentionResolverProtocol
from amplifier_foundation.mentions.resolver import BaseMentionResolver

# Path utilities
from amplifier_foundation.paths.construction import construct_agent_path
from amplifier_foundation.paths.construction import construct_context_path
from amplifier_foundation.paths.discovery import find_bundle_root
from amplifier_foundation.paths.discovery import find_files
from amplifier_foundation.paths.resolution import ParsedURI
from amplifier_foundation.paths.resolution import normalize_path
from amplifier_foundation.paths.resolution import parse_uri
from amplifier_foundation.registry import BundleRegistry
from amplifier_foundation.registry import BundleState
from amplifier_foundation.registry import UpdateInfo
from amplifier_foundation.registry import load_bundle

# Serialization utilities
from amplifier_foundation.serialization import sanitize_for_json
from amplifier_foundation.serialization import sanitize_message
from amplifier_foundation.sources.protocol import SourceHandlerProtocol
from amplifier_foundation.sources.protocol import SourceHandlerWithStatusProtocol
from amplifier_foundation.sources.protocol import SourceResolverProtocol
from amplifier_foundation.sources.protocol import SourceStatus
from amplifier_foundation.sources.resolver import SimpleSourceResolver

# Tracing utilities
from amplifier_foundation.tracing import generate_sub_session_id

# Updates - bundle update checking and updating
from amplifier_foundation.updates import BundleStatus
from amplifier_foundation.updates import check_bundle_status
from amplifier_foundation.updates import update_bundle
from amplifier_foundation.validator import BundleValidator
from amplifier_foundation.validator import ValidationResult
from amplifier_foundation.validator import validate_bundle
from amplifier_foundation.validator import validate_bundle_or_raise

__all__ = [
    # Core
    "Bundle",
    "BundleRegistry",
    "BundleState",
    "UpdateInfo",
    "BundleValidator",
    "ValidationResult",
    "load_bundle",
    "validate_bundle",
    "validate_bundle_or_raise",
    # Exceptions
    "BundleError",
    "BundleNotFoundError",
    "BundleLoadError",
    "BundleValidationError",
    "BundleDependencyError",
    # Protocols
    "MentionResolverProtocol",
    "SourceResolverProtocol",
    "SourceHandlerProtocol",
    "SourceHandlerWithStatusProtocol",
    "SourceStatus",
    "CacheProviderProtocol",
    # Updates
    "BundleStatus",
    "check_bundle_status",
    "update_bundle",
    # Reference implementations
    "BaseMentionResolver",
    "SimpleSourceResolver",
    "SimpleCache",
    "DiskCache",
    # Mentions
    "parse_mentions",
    "load_mentions",
    "ContentDeduplicator",
    "ContextFile",
    "MentionResult",
    # I/O
    "read_yaml",
    "write_yaml",
    "parse_frontmatter",
    "read_with_retry",
    "write_with_retry",
    "write_with_backup",
    # Serialization
    "sanitize_for_json",
    "sanitize_message",
    # Tracing
    "generate_sub_session_id",
    # Dicts
    "deep_merge",
    "merge_module_lists",
    "get_nested",
    "set_nested",
    # Paths
    "parse_uri",
    "ParsedURI",
    "normalize_path",
    "construct_agent_path",
    "construct_context_path",
    "find_files",
    "find_bundle_root",
]
```

Everything is organized into logical groups:

- **Core** — `Bundle`, `BundleRegistry`, `load_bundle`, validation
- **Exceptions** — the full hierarchy
- **Protocols** — extension points for custom source handlers, mention resolvers, cache providers
- **Updates** — bundle update checking (`check_bundle_status`, `update_bundle`)
- **Reference implementations** — `BaseMentionResolver`, `SimpleSourceResolver`, `SimpleCache`, `DiskCache`
- **Mentions** — parser, loader, deduplicator, models
- **I/O** — YAML, frontmatter, retry-safe file operations
- **Serialization** — JSON sanitization
- **Tracing** — sub-session ID generation
- **Dicts** — merge and navigation
- **Paths** — URI parsing, path construction, file discovery

## 14. The Complete Data Flow

To tie it all together, here's the end-to-end flow when an application loads and runs a bundle:

```
Application Code
    │
    ├─ load_bundle("git+https://github.com/org/repo@main")
    │       │
    │       ├─ BundleRegistry._load_single()
    │       │       │
    │       │       ├─ parse_uri() → ParsedURI(scheme=git+https, ref=main, ...)
    │       │       │
    │       │       ├─ SimpleSourceResolver.resolve()
    │       │       │       └─ GitSourceHandler.resolve() → shallow clone to ~/.amplifier/cache/
    │       │       │           └─ ResolvedSource(active_path, source_root)
    │       │       │
    │       │       ├─ _load_from_path() → read bundle.md, parse_frontmatter()
    │       │       │       └─ Bundle.from_dict(frontmatter_data)
    │       │       │           instruction = markdown body
    │       │       │
    │       │       ├─ Sub-bundle detection: walk up to find root bundle.md
    │       │       │       └─ Register root in source_base_paths
    │       │       │
    │       │       └─ _compose_includes() → parallel recursive loading
    │       │               ├─ Load each include via _load_single() (recursive)
    │       │               └─ Compose: includes.compose(current_bundle)
    │       │
    │       └─ Composed Bundle (with all includes merged)
    │
    ├─ bundle.compose(provider_bundle) → Final composed Bundle
    │
    ├─ bundle.prepare()
    │       │
    │       ├─ to_mount_plan() → dict for AmplifierSession
    │       ├─ ModuleActivator: install bundle packages
    │       ├─ ModuleActivator.activate_all() → parallel download + install
    │       └─ BundleModuleResolver(module_paths)
    │               └─ PreparedBundle(mount_plan, resolver, bundle)
    │
    └─ prepared.create_session()
            │
            ├─ AmplifierSession(mount_plan)
            ├─ Mount BundleModuleResolver as "module-source-resolver"
            ├─ session.initialize() → kernel loads all modules
            ├─ Resolve pending context references
            ├─ Register system prompt factory
            │       └─ Called every turn: re-reads files, re-processes @mentions
            │
            └─ session.execute("instruction")
                    └─ LLM sees: [context blocks] + [system instruction]
                        with access to all activated tools
```

## Design Principles

Throughout this codebase, several principles are consistently applied:

1. **Mechanism, not policy** — Foundation provides the *how* without dictating the *what*. It loads bundles without knowing about specific bundles. Apps decide which bundles to load, how to resolve agent names, when to spawn sub-sessions.

2. **Composition over inheritance** — No class hierarchies for customization. Everything is declarative YAML composed via `includes:` and `compose()`.

3. **Protocol-based extensibility** — Custom source handlers, mention resolvers, and cache providers plug in via protocols without subclassing.

4. **Text-first** — Bundles are YAML/Markdown files. Versionable, diffable, human-readable. No imperative code needed to define a bundle.

5. **Opportunistic resilience** — Failed includes are warned and skipped. @mentions to nonexistent files silently resolve to nothing. Cloud-synced files get automatic retries.

6. **Parallelism by default** — Include loading, module activation, and source resolution all happen concurrently via `asyncio.gather()`.

7. **Content-addressed deduplication** — @mentions use SHA-256 hashing so the same content is never loaded twice, regardless of how many paths reference it.
