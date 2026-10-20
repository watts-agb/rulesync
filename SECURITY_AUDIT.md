# Security audit

**Repository:** `watts-agb/rulesync`  
**Assessment:** Static security review  
**Conclusion:** Do not treat this repository as safe to use without addressing the high-severity credential exposure described below. The review found no evidence of hidden CLI telemetry, but the application and its automation do make expected outbound requests, some of which transmit data.

## Findings

### High — npm credentials can be sent to an HTTP registry

**Evidence:** `/home/runner/work/rulesync/rulesync/src/lib/npm-client.ts:73-80, 88-100, 139-144`; registry URL validation at `/home/runner/work/rulesync/rulesync/src/config/config.ts:73-94`.

The registry URL validation accepts `http://` URLs and only warns. The npm client resolves a token from `NPM_TOKEN` by default (or from an environment variable configured by `tokenEnv`) and attaches it to registry requests. A configured HTTP registry can therefore receive the token in cleartext; network observers may also capture it.

**Recommendation:** Reject HTTP registries, or at minimum never attach credentials unless the registry URL uses HTTPS. Treat any token sent to an HTTP registry as exposed and rotate it.

### Medium — publishing workflow does not verify that the release PR was merged

**Evidence:** `/home/runner/work/rulesync/rulesync/.github/workflows/publish.yml:16`; upstream workflow gate at `/home/runner/work/rulesync/rulesync/.github/workflows/publish-assets.yml:25-30`.

The publisher accepts a successful `Publish Assets` run triggered by a pull-request event with a source branch beginning `release/v`, but does not independently verify that the PR was merged or that the upstream `build-assets` job succeeded. The upstream build job is skipped for an unmerged PR. A skipped job may still leave its workflow run successful, allowing a closed, unmerged fork PR with a matching branch name to satisfy the publisher's gate. The publisher checks out `main` and publishes that package, so this does not allow the contributor to supply package contents, but may publish a version outside the intended release gate if that version is not already on npm.

**Recommendation:** Gate publishing directly on a merged release PR and successful build job; verify the originating PR state and upstream job conclusion before publishing.

## Data flows and outbound requests

The review found expected network activity:

- The npm client contacts the configured registry and tarball host. Depending on configuration, it can send an npm token; see the high-severity finding above.
- GitHub API/releases and configured Git remotes are used for repository-related operations.
- The scheduled/manual security-scan workflow sends generated repository XML to OpenRouter and emails high-severity findings to its configured recipient through Resend.
- CI uses package registries and a configured CI tracing service.

No evidence of hidden CLI telemetry was found in the reviewed paths. This is not proof that no data can ever leave the system: outbound behavior depends on user configuration, environment variables, workflow secrets, and the services those settings point to. Review those settings and avoid placing sensitive repository content in workflows that transmit it to third parties unless that transfer is intended.

## Scope and limitations

The static review covered tracked source, npm/GitHub fetching and update paths, filesystem/path handling, frontmatter parsing, package/dependency configuration, scripts, relevant tests and documentation, and CI/release workflows. The findings above are the high-confidence issues identified in that review.

No files, tests, or workflows were executed as part of the security review; no external services were contacted, and live dependency advisories were not checked. This report is not a guarantee that the repository is free of vulnerabilities. Re-audit after the findings are addressed and review dependency advisories and actual deployment/workflow configuration before use.
