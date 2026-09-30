---
title: Git Raw Architecture Model
model_version: 1
source_revision: "Repository source tree reviewed for this document; see GIT-VERSION-GEN"
status: reverse-engineered
---

# Raw Architecture Model

Machine- and human-readable source model. The YAML block is the structured
payload; paths and symbols in `evidence` point to repository evidence. `verified`
records directly observed behavior or structure. `not_applicable` explicitly
marks concepts that do not describe Git. `inferred` records cautious
architectural interpretation, not a project guarantee. Optional components and
features may be absent from a particular build. See
[Architecture Design](ArchitectureDesign.md) for the narrative view.

```yaml
model:
  name: git-source-tree-architecture
  kind: reverse-engineered-architecture
  scope:
    includes:
      - native Git command suite and shared C implementation
      - repository object, ref, index, config, and worktree data
      - client/server transport processes and optional front ends
      - build, test, CI, and release support
    excludes:
      - exhaustive per-command specifications
      - a deployment-specific server topology
      - claims about all optional build configurations
    evidence_method:
      verified: direct source, test, build-rule, or repository-manual evidence
      not_applicable: explicitly rejected architecture assumption
      inferred: interpretation identified as non-normative

  architecture:
    style:
      - process-oriented CLI and plumbing/porcelain command suite
      - native C libraries and command implementations
      - filesystem-backed distributed object/ref repository
      - client/server protocols are optional invocation modes
    not_applicable:
      - relational database
      - SQL schema
      - ORM-managed persistence entities
      - mandatory always-on application server
      - core HTTP controller/router framework
    inferred:
      statement: Shared subsystems and format compatibility favor incremental evolution over a monolithic application architecture.
      rationale: Commands compose reusable repository, object, ref, index, history, diff, and transport APIs.
    evidence:
      - path: git.c
        symbols: [cmd_main, run_argv, commands, run_builtin, execv_dashed_external]
      - path: common-main.c
        symbols: [main]
      - path: common-init.c
        symbols: [init_git]
      - path: Documentation/gitdatamodel.adoc
      - path: Documentation/gitrepository-layout.adoc

  components:
    - id: process-runtime
      responsibility: Shared process initialization, top-level options, command dispatch, child lifecycle, diagnostics, pager, tracing.
      language: C
      evidence:
        - path: common-main.c
          symbols: [main]
        - path: common-init.c
          symbols: [init_git]
        - path: git.c
          symbols: [cmd_main, handle_options, run_argv, run_builtin]
        - path: usage.c
          symbols: [die, error, warning, usage]
    - id: builtin-commands
      responsibility: Porcelain workflows and plumbing operations selected from a command registry.
      language: C
      evidence:
        - path: git.c
          symbols: [commands, get_builtin, handle_builtin]
        - path: builtin.h
          symbols: [builtin command declarations]
        - path: builtin/add.c
          symbols: [cmd_add]
        - path: builtin/commit.c
          symbols: [cmd_commit, cmd_status]
        - path: builtin/fetch.c
          symbols: [cmd_fetch]
        - path: builtin/push.c
          symbols: [cmd_push]
    - id: repository-context
      responsibility: Locate repository/worktree, gitdir, common directory, setup policy, and path prefix.
      language: C
      evidence:
        - path: setup.c
          symbols: [setup_git_directory, setup_git_directory_gently, enter_repo, ensure_valid_ownership]
        - path: repository.c
          symbols: [repository initialization and lifecycle]
        - path: repository.h
          symbols: [struct repository]
    - id: object-database
      responsibility: Hash, read, write, parse, enumerate, and verify Git objects across loose, packed, and alternate sources.
      evidence:
        - path: object-file.c
          symbols: [format_object_header, check_object_signature, odb_loose_path]
        - path: object-name.c
          symbols: [repo_get_oid, repo_get_oid_with_flags]
        - path: odb.c
          symbols: [object database source management]
        - path: odb.h
          symbols: [struct object_database]
        - path: packfile.c
          symbols: [pack access]
        - path: Documentation/gitformat-loose.adoc
        - path: Documentation/gitformat-pack.adoc
    - id: references
      responsibility: Resolve direct and symbolic refs, select storage backend, and transact ref/reflog changes.
      evidence:
        - path: refs.c
          symbols: [repo_dwim_ref, refs_update_ref]
        - path: refs/files-backend.c
          symbols: [files ref backend]
        - path: reftable/table.c
          symbols: [reftable storage]
        - path: refs/refs-internal.h
          symbols: [struct ref_storage_be]
        - path: Documentation/ref-storage-format.adoc
    - id: index-and-worktree
      responsibility: Maintain staging entries, filesystem stat cache, worktree/index/tree comparison, checkout and unpack.
      evidence:
        - path: read-cache.c
          symbols: [read_index_from, write_locked_index, add_to_index, refresh_index]
        - path: builtin/add.c
          symbols: [cmd_add]
        - path: unpack-trees.c
          symbols: [tree/index/worktree unpack operations]
        - path: Documentation/gitformat-index.adoc
    - id: revision-history
      responsibility: Resolve revision arguments and traverse commit graphs under ordering, exclusion, and simplification options.
      evidence:
        - path: revision.c
          symbols: [setup_revisions, prepare_revision_walk]
        - path: object-name.c
          symbols: [repo_get_oid, repo_get_oid_committish]
        - path: commit.c
          symbols: [repo_parse_commit_gently]
        - path: Documentation/gitrevisions.adoc
    - id: diff-merge
      responsibility: Compare repository snapshots and worktree/index state; generate diffs and apply/merge changes.
      evidence:
        - path: diff.c
          symbols: [diff orchestration]
        - path: diffcore-rename.c
          symbols: [rename detection]
        - path: xdiff/
          symbols: [text diff implementation]
        - path: merge-ort.c
          symbols: [merge-ort strategy]
        - path: sequencer.c
          symbols: [sequenced operations]
    - id: transport
      responsibility: Select remote transport, negotiate refs and object wants, transfer packs, and report update status.
      evidence:
        - path: transport.c
          symbols: [transport_fetch_refs, transport_push]
        - path: fetch-pack.c
          symbols: [fetch-pack protocol client]
        - path: send-pack.c
          symbols: [send_pack]
        - path: upload-pack.c
          symbols: [upload_pack, upload_pack_v2]
        - path: builtin/upload-pack.c
          symbols: [cmd_upload_pack]
        - path: builtin/receive-pack.c
          symbols: [receive-pack command, receive hooks and policy]
        - path: Documentation/gitprotocol-pack.adoc
        - path: Documentation/gitprotocol-v2.adoc
    - id: configuration
      responsibility: Parse layered config, expose callback and typed-access APIs, and provide protected-config access.
      evidence:
        - path: config.c
          symbols: [config_with_options, do_git_config_sequence, git_protected_config]
        - path: Documentation/config.adoc
        - path: Documentation/config/safe.adoc
    - id: auxiliary-frontends
      responsibility: Optional GUI, CGI, scripting, credential, and compatibility interfaces outside the core command execution path.
      optional: true
      evidence:
        - path: gitweb/gitweb.perl
          symbols: [CGI application]
        - path: git-gui/
          symbols: [Tcl/Tk GUI]
        - path: gitk-git/
          symbols: [Tcl/Tk history viewer]
        - path: git-svn.perl
          symbols: [git-svn]
        - path: git-p4.py
          symbols: [git-p4]
        - path: contrib/credential/
          symbols: [credential helper integrations]

  entry_points:
    - id: main-git-cli
      invocation: "git [global-options] <command> [args]"
      flow: "common-main.c:main -> init_git -> git.c:cmd_main -> run_argv -> builtin or git-command"
      evidence:
        - common-main.c:main
        - common-init.c:init_git
        - git.c:cmd_main
        - git.c:run_argv
    - id: dashed-command
      invocation: "git-<command> [args]"
      flow: "Standalone/helper wrapper enters the associated command implementation."
      evidence:
        - git.c:execv_dashed_external
        - Makefile:ALL_COMMANDS_TO_INSTALL
        - bin-wrappers/
    - id: upload-pack
      invocation: "git-upload-pack <repository>"
      role: "Server-side fetch/clone service process."
      evidence:
        - builtin/upload-pack.c:cmd_upload_pack
        - upload-pack.c:upload_pack
    - id: receive-pack
      invocation: "git-receive-pack <repository>"
      role: "Server-side push service process."
      evidence:
        - builtin/receive-pack.c:receive-pack command
        - Documentation/gitprotocol-pack.adoc
    - id: http-backend
      invocation: "CGI environment supplied by an external web server"
      role: "Optional smart HTTP backend; not a core listener or router."
      optional: true
      evidence:
        - http-backend.c
        - Documentation/git-http-backend.adoc
    - id: gitweb
      invocation: "CGI environment supplied by an external web server"
      role: "Optional Perl repository browser."
      optional: true
      evidence:
        - gitweb/gitweb.perl
    - id: gui
      invocation: "wish-based Tcl/Tk applications"
      optional: true
      evidence:
        - git-gui/
        - gitk-git/gitk

  interfaces:
    - id: command-line
      kind: process interface
      data: [argv, environment, stdin, stdout, stderr, exit status]
      notes: "Command-specific options, output, and exit meanings vary."
      evidence: [git.c:cmd_main, builtin/*.c, Documentation/gitcli.adoc]
    - id: object-store
      kind: on-disk binary formats
      data: [loose objects, packs, pack indexes, auxiliary object metadata]
      evidence: [Documentation/gitformat-loose.adoc, Documentation/gitformat-pack.adoc]
    - id: reference-store
      kind: backend API and on-disk ref representation
      backends: [files, reftable]
      evidence: [refs/refs-internal.h:struct ref_storage_be, Documentation/ref-storage-format.adoc]
    - id: index-file
      kind: checksummed binary file
      evidence: [read-cache.c:read_index_from, read-cache.c:write_locked_index, Documentation/gitformat-index.adoc]
    - id: git-wire-protocol
      kind: packet-line framed process stream
      transports: [ssh, git, HTTP(S), file/local]
      evidence: [Documentation/gitprotocol-common.adoc, Documentation/gitprotocol-pack.adoc, Documentation/gitprotocol-v2.adoc]
    - id: helper-protocols
      kind: configured child-process contracts
      examples: [credential helpers, filters, hooks, remote helpers, merge tools]
      evidence: [credential.c, convert.c, hook.c, remote.c, Documentation/githooks.adoc]

  data_stores:
    - id: object-database
      location: "$GIT_DIR/objects or common-directory objects"
      formats: [loose zlib files, packfiles and indexes, optional auxiliary indexes]
      consistency: "Objects are content-addressed; a repository can be shallow, alternate-backed, or partial/promisor."
      not_database: "File/object store, not SQL or ORM."
      evidence: [Documentation/gitrepository-layout.adoc, Documentation/gitformat-loose.adoc, Documentation/gitformat-pack.adoc, odb.c]
    - id: refs
      location: "$GIT_DIR/refs, packed-refs, reftable storage, or common directory"
      content: "Mutable names to object IDs or symbolic targets; backend-specific."
      evidence: [Documentation/gitrepository-layout.adoc, Documentation/git-refs.adoc, refs.c, refs/files-backend.c, reftable/table.c]
    - id: reflogs
      location: "$GIT_DIR/logs or worktree/common directory as applicable"
      content: "Local ref-change records; not transferred as ordinary remote history."
      evidence: [Documentation/gitdatamodel.adoc, Documentation/gitrepository-layout.adoc, refs.c]
    - id: index
      location: "$GIT_DIR/index or worktree-specific gitdir"
      content: "Sorted staging entries, stat cache, conflict stages and extensions."
      evidence: [Documentation/gitformat-index.adoc, read-cache.c]
    - id: configuration
      location: "System, XDG/global, user/global, local, worktree, command-line sources"
      content: "Text key/value configuration with include and scope behavior."
      evidence: [config.c:do_git_config_sequence, Documentation/config.adoc]
    - id: worktree
      location: "Filesystem checkout when repository is non-bare"
      content: "User-visible files; separate from object storage and index."
      evidence: [Documentation/gitrepository-layout.adoc, setup.c:setup_git_directory]

  entities:
    - name: Object
      variants: [blob, tree, commit, annotated-tag]
      invariant: "Object ID is hash of typed contents; objects are immutable."
      evidence: [Documentation/gitdatamodel.adoc, object-file.c:format_object_header]
    - name: TreeEntry
      fields: [mode, name, object_id]
      targets: [blob, tree, commit-gitlink]
      evidence: [Documentation/gitdatamodel.adoc, tree.c]
    - name: Commit
      fields: [tree_id, parent_ids, author, committer, message]
      evidence: [Documentation/gitdatamodel.adoc, commit.c:repo_parse_commit_gently]
    - name: Ref
      forms: [direct object ID, symbolic ref]
      namespaces: [refs/heads, refs/tags, refs/remotes, other refs]
      evidence: [Documentation/gitdatamodel.adoc, refs.c:repo_dwim_ref]
    - name: IndexEntry
      fields: [path, mode, object_id, stage, cached stat data]
      stages: [0, 1, 2, 3]
      evidence: [Documentation/gitdatamodel.adoc, Documentation/gitformat-index.adoc, read-cache.h]
    - name: ReflogEntry
      fields: [object ID, timestamp, message]
      scope: local
      evidence: [Documentation/gitdatamodel.adoc]

  flows:
    - id: builtin-command
      steps:
        - "Initialize shared process state."
        - "Parse global options and select command."
        - "Resolve alias, built-in, or external git-command."
        - "Perform repository/worktree setup according to command flags."
        - "Invoke command implementation and shared repository APIs."
        - "Write output/diagnostics and propagate process status."
      evidence: [common-main.c:main, git.c:cmd_main, git.c:run_builtin]
    - id: stage-and-commit
      steps:
        - "Read worktree paths and apply pathspec/attributes/conversion."
        - "Hash/write blob objects."
        - "Update index entries under lock."
        - "Write tree from index."
        - "Write commit object with tree and parent(s)."
        - "Transactionally update current branch/ref and reflog."
      evidence: [builtin/add.c:cmd_add, read-cache.c:add_to_index, builtin/write-tree.c:cmd_write_tree, builtin/commit.c:cmd_commit, refs.c:refs_update_ref]
    - id: revision-walk
      steps:
        - "Parse revision arguments and options."
        - "Resolve names, refs, object IDs, and revision operators."
        - "Parse commits and seed traversal."
        - "Prepare queues and apply exclusions/order/filters."
        - "Read parent and tree edges as requested; emit result."
      evidence: [revision.c:setup_revisions, revision.c:prepare_revision_walk, object-name.c:repo_get_oid]
    - id: fetch
      steps:
        - "Select transport and invoke/connect to upload-pack."
        - "Advertise refs/capabilities and negotiate wants/haves."
        - "Receive pack and index incoming objects."
        - "Validate connectivity and update selected refs."
      evidence: [fetch-pack.c, transport.c:transport_fetch_refs, builtin/upload-pack.c:cmd_upload_pack, Documentation/gitprotocol-pack.adoc]
    - id: push
      steps:
        - "Discover remote refs and negotiate update commands."
        - "Send required pack and requested ref changes."
        - "Receive-pack checks policy, connectivity, and hooks."
        - "Commit ref transaction and report per-ref status."
      evidence: [send-pack.c:send_pack, builtin/receive-pack.c, refs.c:refs_update_ref, Documentation/gitprotocol-pack.adoc]
    - id: atomic-file-update
      steps:
        - "Create an exclusive sibling lock/temp file."
        - "Write and validate replacement contents."
        - "Rename into destination on commit, or remove temporary state on rollback."
      evidence: [lockfile.c:hold_lock_file_for_update_timeout_mode, lockfile.c:commit_lock_file, read-cache.c:write_locked_index]

  configuration_surfaces:
    - surface: "git config files"
      scopes: [system, XDG, global, local, worktree, command]
      features: [includes, typed accessors, protected config]
      evidence: [config.c:do_git_config_sequence, Documentation/config.adoc]
    - surface: environment
      examples: [GIT_DIR, GIT_WORK_TREE, GIT_CONFIG_NOSYSTEM, GIT_CONFIG_PARAMETERS, GIT_PROTOCOL]
      evidence: [environment.h, config.c, Documentation/git.adoc, Documentation/gitprotocol-v2.adoc]
    - surface: attributes-and-ignore
      files: [.gitattributes, .gitignore, repository exclude files]
      evidence: [attr.c, dir.c, Documentation/gitattributes.adoc, Documentation/gitignore.adoc]
    - surface: hooks
      location: "$GIT_DIR/hooks or core.hooksPath"
      evidence: [hook.c:run_hooks_opt, Documentation/githooks.adoc]
    - surface: build-options
      examples: [config.mak, configure options, Meson options, NO_CURL, NO_PERL, NO_TCLTK]
      evidence: [Makefile, configure.ac, meson.build, INSTALL]

  trust_boundaries:
    - id: local-filesystem
      concern: "Repository ownership and filesystem permissions influence config and worktree access."
      controls: [safe.directory, safe.bareRepository, ownership checks]
      evidence: [setup.c:ensure_valid_ownership, Documentation/config/safe.adoc]
    - id: network-transport
      concern: "Protocol peers send untrusted framed commands, object data, and paths."
      controls: [protocol parsing, object validation, service policy, transport configuration]
      warning: "git:// protocol is unauthenticated by itself."
      evidence: [Documentation/gitprotocol-pack.adoc, Documentation/gitprotocol-v2.adoc, builtin/receive-pack.c]
    - id: server-authorization
      concern: "SSH/HTTP identity and hosting permissions are supplied outside core Git."
      controls: [SSH account policy, web server configuration, receive rules, hooks]
      evidence: [Documentation/gitprotocol-pack.adoc, Documentation/git-http-backend.adoc, Documentation/githooks.adoc]
    - id: subprocess
      concern: "Hooks, filters, credential helpers, remote helpers, pagers, and tools execute external programs."
      evidence: [run-command.c, hook.c, convert.c, credential.c, Documentation/githooks.adoc]
    - id: config-and-repository-content
      concern: "Repository data and configuration may select paths, commands, protocols, or transformations."
      controls: [protected config, safe repository checks, protocol allow-list, parser validation]
      evidence: [config.c:git_protected_config, setup.c, Documentation/config/protocol.adoc, SECURITY.md]

  error_categories:
    - category: usage
      behavior: "Command usage diagnostic; process status and text follow shared usage helper and command contract."
      evidence: [usage.c:usage, git.c:handle_options]
    - category: fatal
      behavior: "Unrecoverable local error reported through die()/die_errno()."
      evidence: [usage.c:die, lockfile.c:unable_to_lock_die]
    - category: recoverable
      behavior: "Reported with error()/warning() or returned by subsystem; caller may continue or propagate status."
      evidence: [usage.c:error, refs.c:refs_update_ref]
    - category: protocol
      behavior: "Packet error or per-ref status terminates or rejects exchange/update."
      evidence: [Documentation/gitprotocol-pack.adoc, builtin/receive-pack.c]
    - category: subprocess
      behavior: "Child exit status, start failure, and hook-specific rejection are handled by caller."
      evidence: [run-command.c, hook.c:run_hooks_opt, builtin/receive-pack.c]

  patterns:
    - name: command-registry
      evidence: [git.c:commands, git.c:run_builtin]
    - name: porcelain-over-plumbing
      evidence: [Documentation/git.adoc, Documentation/gitcore-tutorial.adoc, builtin/]
    - name: backend-vtable
      examples: [reference storage, transport]
      evidence: [refs/refs-internal.h:struct ref_storage_be, transport.h:struct transport_vtable]
    - name: transaction-and-lockfile
      examples: [ref transaction, index lockfile]
      evidence: [refs.c:refs_update_ref, lockfile.c:commit_lock_file, read-cache.c:write_locked_index]
    - name: callback-and-iterator
      examples: [configuration, refs, object traversal]
      evidence: [config.c:config_with_options, refs.c, odb.h]
    - name: child-process-adapter
      examples: [hooks, filters, credential helpers, remote helpers]
      evidence: [run-command.c, hook.c, credential.c, remote.c]
    - name: caching-and-lazy-loading
      examples: [parsed objects, object sources, commit graph/index]
      evidence: [repository.h:struct repository, odb.h:struct object_database, revision.c]

  build_test_topology:
    build_systems:
      - name: GNU Make
        role: primary project build and install rules
        evidence: [Makefile, INSTALL]
      - name: autoconf
        role: optional feature detection and generated config
        evidence: [configure.ac, INSTALL]
      - name: Meson
        role: supported alternative build
        evidence: [meson.build, Documentation/technical/build-systems.adoc]
      - name: CMake
        role: supplemental build-system support
        evidence: [contrib/buildsystems/CMakeLists.txt]
    tests:
      integration: "t/t*.sh shell tests using test-lib.sh and TAP-compatible output"
      unit: "C unit tests under t/unit-tests and helper programs under t/helper"
      additional: [interop, performance, fuzzing, doc checks, lint]
      evidence: [t/README, t/Makefile, ci/run-build-and-tests.sh, ci/test-documentation.sh]
    ci:
      providers: [GitHub Actions, GitLab CI, Cirrus CI]
      dimensions: [Linux, macOS, Windows, compiler, build system, hash format, ref backend, sanitizers]
      evidence: [.github/workflows/main.yml, .gitlab-ci.yml, .cirrus.yml, ci/]

  deployment_runtime_modes:
    - mode: local-client
      persistent_service: false
      description: "CLI command process accesses local worktree/repository; ordinary default mode."
      evidence: [README.md, INSTALL, git.c:cmd_main]
    - mode: ssh-hosted
      persistent_service: optional
      description: "SSH remote execution starts upload-pack or receive-pack under external account policy."
      evidence: [Documentation/gitprotocol-pack.adoc, builtin/upload-pack.c, builtin/receive-pack.c]
    - mode: git-daemon
      persistent_service: optional
      description: "Configured git:// listener exposes selected Git service commands; protocol is unauthenticated."
      evidence: [daemon.c, Documentation/git-daemon.adoc, Documentation/gitprotocol-pack.adoc]
    - mode: smart-http
      persistent_service: external-web-server
      description: "External HTTP server invokes git-http-backend in CGI-style request environment."
      evidence: [http-backend.c, Documentation/git-http-backend.adoc]
    - mode: gitweb
      persistent_service: external-web-server
      description: "Optional Perl CGI browsing interface."
      evidence: [gitweb/gitweb.perl, Documentation/gitweb.adoc]
    - mode: installed-suite
      persistent_service: false
      description: "Executables, wrappers, scripts, templates, and manuals installed below configured prefix."
      evidence: [Makefile:install, INSTALL, GIT-VERSION-GEN]

  risks_and_observations:
    - risk: "Large compatibility surface across commands, formats, and options makes regressions cross-cutting."
      status: inferred
      evidence: [Documentation/technical/build-systems.adoc, t/]
    - risk: "Filesystem semantics and timestamp resolution vary by platform."
      status: verified
      evidence: [Documentation/technical/racy-git.adoc, compat/, t/t0050-filesystem.sh]
    - risk: "Optional integrations and platform configurations are not all built or tested in every job."
      status: verified
      evidence: [INSTALL, .github/workflows/main.yml, .gitlab-ci.yml]
    - risk: "Subprocess configuration and hooks cross security boundaries."
      status: verified
      evidence: [hook.c, run-command.c, Documentation/githooks.adoc, SECURITY.md]
    - risk: "Multiple object/ref/index representations multiply parser and migration cases."
      status: inferred
      evidence: [Documentation/gitformat-pack.adoc, Documentation/ref-storage-format.adoc, Documentation/gitformat-index.adoc]

  evidence_index:
    core_docs:
      - README.md
      - INSTALL
      - SECURITY.md
      - Documentation/gitdatamodel.adoc
      - Documentation/gitrepository-layout.adoc
      - Documentation/gitformat-loose.adoc
      - Documentation/gitformat-pack.adoc
      - Documentation/gitformat-index.adoc
      - Documentation/ref-storage-format.adoc
      - Documentation/gitprotocol-pack.adoc
      - Documentation/gitprotocol-v2.adoc
      - Documentation/gitprotocol-http.adoc
      - Documentation/config.adoc
      - Documentation/githooks.adoc
    implementation:
      - common-main.c
      - common-init.c
      - git.c
      - setup.c
      - repository.c
      - object-file.c
      - object-name.c
      - odb.c
      - refs.c
      - refs/files-backend.c
      - reftable/
      - read-cache.c
      - revision.c
      - transport.c
      - fetch-pack.c
      - send-pack.c
      - upload-pack.c
      - builtin/receive-pack.c
    operations:
      - Makefile
      - configure.ac
      - meson.build
      - t/README
      - t/Makefile
      - ci/
      - .github/workflows/
      - .gitlab-ci.yml
      - .cirrus.yml
```
