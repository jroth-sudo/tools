# build-openssl

Automates the process of downloading, building, and installing OpenSSL from source -- with flexible version/architecture options, consistent naming, and a symlink strategy that simplifies downstream integration.

---

## Why I Built It

This script was built to address a recurring need: our software depends on OpenSSL and we often need to build/test against multiple versions (including both current and legacy releases).

While some platforms provide packaged versions, they’re typically limited to one or two releases. In some environments, packages don’t exist at all. Even where they did exist, mixing native packages with source builds would have broken our standardized directory layout. On top of that, manually building OpenSSL was tedious and error-prone: inconsistent naming, too reliant on memory or notes, etc. To support our full compatibility matrix and keep things clean, we needed a consistent, self-managed solution.

This solution removed that friction. It helped to:

- Save time
- Standardize how and where OpenSSL was installed
- Simplify symlink management across dev hosts
- Minimize changes needed in our connecting build scripts

---

## Design Overview

It installs each build into version-specific directories like:

```
/usr/foo/openssl-3.5.1-prod/
```

It then creates a symlink with a more general name:

```
openssl-35-prod -> openssl-3.5.1-prod/
```

so downstream tools can reference a stable name regardless of patch version.

Platform-specific flags, install paths, and behavior are handled using `uname` detection and `case/esac` logic. Furthermore, the script adapts based on the environment, with guardrails for things like security restrictions (e.g. SIP on macOS) or compiler quirks.

---

## Notable Features

- **Versioned install layout**  
  Each OpenSSL version -- down to the patch level -- is installed under a unique path. Example:  
  `openssl-3.5.1-prod/`

  The script derives this structure automatically from the version string:

  ```bash
  # $vers typically comes from a hardcoded default or --version override
  vers_major_minor=$(echo "$vers" | cut -d. -f1-2)
  vers_link_id=$(echo "$vers_major_minor" | sed 's/\.//g')
  # install path using full version
  destdir=$destdirroot/openssl-${vers}-prod
  ```

- **Flexible symlink scheme**  
  Generalized names like `openssl-35-prod` point to the full version directory, keeping build logic simple even when patch versions change.

- **32-bit and 64-bit aware**  
  Architecture-specific builds use suffixes like `.32` or `.64` to ensure separation (e.g. `openssl-35-prod.64`). This allows both variants to coexist cleanly on the same system when needed plus lets the build scripts automatically select the appropriate bit-ness.

- **Cross-platform awareness**  
  The script detects platform and CPU details, adjusting install paths and config flags accordingly.

- **Source cleanup**  
  Temporary build directories are cleaned up automatically after install. This helps keep systems tidy by default (though I occasionally comment it out when debugging a failed build).

---

## Reflection

This wasn’t a flashy tool -- but it was very helpful. It took a repetitive, error-prone task and turned it into a one-liner with predictable results.

In the end, it demonstrated something I value in systems work: a small utility that quietly tightens up process and drift.

If I were to revisit it, I might add a `--no-cleanup` flag. And consider displaying/confirming the install values. That could be a cool sanity check, though it would chip away at the fully automated nature I enjoy.
