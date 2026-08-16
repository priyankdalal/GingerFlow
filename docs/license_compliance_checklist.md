# GingerFlow License Compliance Checklist

This checklist is for engineering release readiness and is not legal advice. Validate final interpretation with your legal/compliance team.

## Current product data-handling note

In the current application state, the core GingerFlow application does not collect or transmit workflow data, file contents, credentials, plugin data, execution payloads, or telemetry to the GingerFlow developer or distributing organization. No data is currently shared with the app developer by the core application.

This does not prevent a workflow from communicating with services explicitly configured by its owner. APIs, databases, FTP servers, email servers, cloud services, command-line tools, and Python plugins may transmit data by design. Review those integrations, their credentials, their dependencies, and their service terms separately.

Safe operation still requires proper planning, secure deployment, least-privilege access, trusted plugins, protected credentials, backups, monitoring, and owner approval. This note describes current implementation behavior; it is not a privacy policy or legal advice.

## 1. Choose Qt licensing path

- [ ] Decide between open-source Qt (LGPL/GPL) and commercial Qt.
- [ ] Record decision owner, date, and rationale.
- [ ] If commercial Qt is selected, store proof of license/subscription and allowed usage scope.

## 2. If using open-source Qt (LGPL path)

- [ ] Use dynamic linking for Qt libraries in release builds.
- [ ] Do not prevent users from replacing Qt shared libraries.
- [ ] Include Qt license notices and copyright notices in distribution.
- [ ] Include full text of required Qt licenses in distribution package.
- [ ] Document Qt version and modules used (Core, Gui, Widgets, etc.).
- [ ] If Qt itself is modified, prepare source/modification disclosure as required by license.

## 3. Third-party dependency inventory

- [ ] Generate Software Bill of Materials (SBOM) or equivalent dependency list.
- [ ] Include direct and transitive dependencies for app and worker artifacts.
- [ ] Capture package name, version, license, homepage, and notice requirements.
- [ ] Flag copyleft dependencies for legal review.
- [ ] Verify plugin dependencies and native libraries separately.

## 4. Build and packaging obligations

- [ ] Verify release uses approved compiler/runtime toolchain.
- [ ] Package all required notices in installer/archive.
- [ ] Add "Third-Party Notices" section to app documentation.
- [ ] Preserve license files in installed location and in release artifacts.
- [ ] Ensure windeployqt/macdeployqt/linux packaging retains license files.

## 5. Product UI and documentation obligations

- [ ] Provide an "About" or "Licenses" entry in the app for third-party attributions.
- [ ] Add license/notice references in README and release notes.
- [ ] Publish source offer/instructions if required by selected licenses.

## 6. Internal policy and approvals

- [ ] Run internal OSS/compliance scanner (if your company has one).
- [ ] Obtain legal sign-off before external distribution.
- [ ] Obtain security sign-off for all externally sourced binaries.
- [ ] Keep an audit trail of approvals and artifact hashes.

## 7. Release gate (must-pass)

- [ ] No unknown or missing dependency licenses.
- [ ] No blocked licenses per company policy.
- [ ] All notice files present in installer/archive.
- [ ] Qt licensing obligations validated for chosen model.
- [ ] Legal/compliance approval recorded.

## 8. Recommended repo artifacts

- [ ] Add a top-level THIRD_PARTY_NOTICES.md.
- [ ] Add a top-level licenses/ directory containing bundled license texts.
- [ ] Add CI step that fails builds when dependency licenses are unresolved.

## 9. Quick decision matrix

- Open-source Qt (LGPL): no purchase required, but distribution obligations apply.
- Commercial Qt: paid, reduces open-source compliance constraints and provides commercial terms/support.

## 10. Release sign-off template

- Release version:
- Date:
- Qt version and modules:
- License model selected:
- Third-party notice bundle path:
- Legal approver:
- Security approver:
- Engineering approver:
- Final status: Approved / Blocked
