# Developer Certificate of Origin (DCO)

## What is the DCO?

The Developer Certificate of Origin (DCO) is a lightweight way for contributors to certify that they wrote or otherwise have the right to submit the code they are contributing to the project.

## How to Comply

### Commit Messages

All commits must include a "Signed-off-by" line:

```
Signed-off-by: Your Name <your.email@example.com>
```

### Example Commit

```
feat(detect): add Arc browser detection on macOS

This commit adds detection for the Arc browser on macOS by checking
for the Arc.app bundle in standard application directories.

Signed-off-by: Aleksei Gurianov <gurianov@gmail.com>
```

### Adding the Sign-Off

Using Git:

```bash
git commit -s -m "your commit message"
```

The `-s` flag automatically adds the `Signed-off-by` line using your Git user configuration.

Or manually:

```bash
git commit -m "your commit message

Signed-off-by: Your Name <your.email@example.com>"
```

### Amending a Commit

If you forgot the sign-off:

```bash
git commit --amend --signoff
```

## Certification Statement

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I have the right to submit it under the open source license indicated in the file; or

(b) The contribution is based upon previous work that, to the best of my knowledge, is covered under an appropriate open source license and I have the right under that license to submit that work with modifications, whether created in whole or in part by me, under the same open source license (unless I am permitted to submit under a different license), as indicated in the file; or

(c) The contribution was provided directly to me by some other person who certified (a), (b) or (c) and I have not modified it.

(d) I understand and agree that this project and the contribution are public and that a record of the contribution (including all personal information I submit with it, including my sign-off) is maintained indefinitely and may be redistributed consistent with this project or the open source license(s) involved.

## Enforcement

Contributions without proper DCO sign-offs will not be merged. GitHub Actions will automatically verify DCO compliance on pull requests.

## More Information

For more details about the DCO, see: https://developercertificate.org/
