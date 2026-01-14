# hms-canary

A minimal HMS microservice repository used for testing container image build workflows and CI/CD pipeline changes before applying them to production HMS service repositories.

## Purpose

The `hms-canary` repository serves as a **test bed for validating changes to the HMS container image build system**. It contains a simple Go-based HTTP service with the minimal structure needed to exercise the full build pipeline:

- Building Docker container images
- Running unit tests
- Running integration tests  
- Running chart-testing (CT) tests
- Scanning for vulnerabilities
- Publishing to Artifactory

This repository allows the HMS build team and service developers to safely test modifications to build workflows before merging changes that affect all HMS service repositories.

## When to Use hms-canary

Use this repository to test changes to:

- **hms-build-image-workflows** - Container image build and release workflows
- **hms-build-metadata-action** - Build metadata generation for images
- **hms-nightly-integration** - Nightly integration test workflows
- **hms-build-workflow-dispatcher** - Workflow dispatch and scheduling logic

**Do not use this repository** for testing Helm chart build workflows - use [hms-canary-charts](https://github.com/Cray-HPE/hms-canary-charts) instead.

## Quick Start

### Prerequisites

- **Docker** (for building images and running tests)
- **Go 1.19+** (for running unit tests and building locally)
- **Helm 3.10.0+** (for Helm-related tests)
- **chart-testing 3.4.0+** (for chart validation tests)
- **make** command
- **git**
- **bash** or **zsh** shell

### Building an Image

```bash
git clone https://github.com/Cray-HPE/hms-canary.git
cd hms-canary

# Build the default Docker image
make image

# Or build with additional Docker arguments
make image DOCKER_ARGS="-v /path/to/cache:/cache"
```

### Running Tests

```bash
# Run all tests (image, unit, integration, snyk, ct, and build CT image)
make all

# Run specific test suites
make unittest          # Unit tests
make integration       # Integration tests
make snyk             # Snyk vulnerability scan
make ct               # Chart-testing tests
make ct_image         # Build chart-testing test image
```

### Viewing Available Targets

```bash
make help  # Shows all available make targets
```

## Testing Build Workflow Changes

### General Workflow

When you have a change to the HMS build system that you want to test:

1. **Identify what to test:** Determine which build system repository/component you're modifying (see [When to Use](#when-to-use-hms-canary))

2. **Create a test branch in hms-canary:**
   ```bash
   git checkout -b test/my-workflow-changes
   ```

3. **Update workflow YAML files** to reference your feature branch in the build system repository

4. **Push and trigger:** GitHub Actions will automatically run when you push, or you can manually trigger via the GitHub Actions UI

5. **Verify:** Check workflow logs and artifacts

6. **Iterate:** Make adjustments and re-push as needed

7. **Apply to production:** Once validated, apply changes to production workflows

### Example: Testing Changes to hms-build-image-workflows

**Scenario:** You've made changes to `hms-build-image-workflows` and want to test them before merging.

1. **Create a test branch:**
   ```bash
   git checkout -b test/image-workflows-update
   ```

2. **Edit `.github/workflows/build_and_release_image.yaml` to reference your feature branch:**
   ```yaml
   name: build_and_release_image
   on:
     push:
       branches: [ main, develop ]
     pull_request:
   
   jobs:
     build_and_release:
       uses: Cray-HPE/hms-build-image-workflows/.github/workflows/build_and_release_image.yaml@feature/my-changes
       with:
         image-name: hms-canary
         artifactory-component: canary-testing
       secrets:
         SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
         ARTIFACTORY_ALGOL60_USERNAME: ${{ secrets.ARTIFACTORY_ALGOL60_USERNAME }}
         # ... other required secrets
   ```

3. **Push changes:**
   ```bash
   git add .github/workflows/build_and_release_image.yaml
   git commit -m "test: validate image-workflows changes"
   git push origin test/image-workflows-update
   ```

4. **Monitor the workflow:**
   - Go to https://github.com/Cray-HPE/hms-canary/actions
   - Click on the workflow run that corresponds to your push
   - Watch for success or failure

5. **Verify results:**
   - Check if the image was built successfully
   - Verify it was published to Artifactory at the expected path
   - Review vulnerability scan results
   - Check PR comments for artifact links (if testing PR workflows)

6. **Troubleshoot if needed:**
   - Review workflow logs for errors
   - Make fixes in the build system repository's feature branch
   - Update the reference in hms-canary if needed
   - Re-push and re-test

### Example: Testing Changes to hms-build-changed-charts-action

**Scenario:** You've made changes to the chart building action and want to test.

Note: For chart building action tests, also see [hms-canary-charts](https://github.com/Cray-HPE/hms-canary-charts).

1. **Create a test branch:**
   ```bash
   git checkout -b test/chart-action-update
   ```

2. **Edit workflows that use `hms-build-changed-charts-action`:**
   ```yaml
   - name: Build changed charts
     uses: Cray-HPE/hms-build-changed-charts-action@feature/my-changes
   ```

3. **Push and test as described above**

## Important Notes

### Credentials and Secrets

The `.github/workflows/` files reference secrets that must be available in respective repositories:

- `SNYK_TOKEN` - For vulnerability scanning
- `ARTIFACTORY_ALGOL60_USERNAME` - For publishing to Artifactory
- `ARTIFACTORY_ALGOL60_TOKEN` - For publishing to Artifactory
- `ARTIFACTORY_ALGOL60_READONLY_USERNAME` - For pulling from Artifactory
- `ARTIFACTORY_ALGOL60_READONLY_TOKEN` - For pulling from Artifactory
- `COSIGN_GCP_PROJECT_ID` - For image signing
- `COSIGN_GCP_SA_KEY` - For image signing
- `COSIGN_KEY` - For image signing

These secrets should be configured at the repository level. If secrets are missing, workflows will fail when attempting to publish artifacts.

### Artifactory Publishing

When artifacts are successfully built, they are published to Artifactory:

- **Unstable:** `csm-docker/unstable/canary-testing/`
- **Stable:** `csm-docker/stable/canary-testing/`

Note: By default, hms-canary produces unstable builds. To produce stable builds, you would need to create git tags.

### Version Management

The version of hms-canary is stored in the `.version` file at the repository root. 

Use semantic versioning in the `.version` file:
- **MAJOR** - Breaking changes or New features (e.g., 1.0.0 → 2.0.0)
- **MINOR** - Small Changes (e.g., 1.0.0 → 1.1.0)
- **PATCH** - Bug fixes (e.g., 1.0.0 → 1.0.1)

Update this when making significant changes to the canary service:

Example Usage:

```bash
echo "1.2.3" > .version
git add .version
git commit -m "bump: version 1.2.2 → 1.2.3"
git push
```

## Common Issues and Troubleshooting

### Issue: "Workflow not found"
**Cause:** The feature branch doesn't exist in the build system repository yet or the path is incorrect.  
**Solution:** Verify the branch name is correct and the workflow file path matches exactly.

### Issue: "Secret X is not available"
**Cause:** Required secrets are not configured in this repository.  
**Solution:** Add the missing secrets to the repository settings, or contact the HMS build team.

### Issue: "Failed to publish to Artifactory"
**Cause:** Artifactory credentials are invalid or the repository doesn't exist.  
**Solution:** Verify credentials are correct and check with Artifactory administrators that the repository exists.

### Issue: "Vulnerability scan failed"
**Cause:** The built image contains vulnerabilities above the configured severity threshold.  
**Solution:** Fix the vulnerabilities in the base image or Dockerfile, then rebuild.

### Issue: "Build matrix failed"
**Cause:** One of the build or test steps failed.  
**Solution:** Click into the failed job and review the logs for specific error messages.

## Best Practices

1. **Always create a feature branch** - Never push to main or develop directly
2. **Test the full pipeline** - Don't just check that individual steps run; verify end-to-end
3. **Verify Artifactory artifacts** - Make sure images are published to the correct location
4. **Review all logs** - Check for warnings even if a workflow succeeded
5. **Test error cases** - Break things intentionally to ensure error handling works
6. **Document findings** - Note any unexpected behaviors in your PR description
7. **Clean up test branches** - Delete test branches after testing is complete
8. **Use meaningful branch names** - Use `test/` prefix and descriptive names like `test/build-env-upgrade`

## Contributing to hms-canary

The hms-canary repository welcomes contributions! Here's how to contribute:

### Reporting Issues

If you find issues with hms-canary:
1. Check if the issue already exists in GitHub Issues
2. Create a new issue with:
   - Clear description of the problem
   - Steps to reproduce
   - Expected vs. actual behavior
   - Environment details (OS, Docker version, Go version, etc.)

### Submitting Changes

1. **Fork the repository** (if you're an external contributor)
2. **Create a feature branch:** `git checkout -b feature/your-feature-name`
3. **Make changes** following the existing code style
4. **Run all tests locally:** `make all`
5. **Commit with clear messages:** `git commit -m "feat: add your feature"`
6. **Push to your fork:** `git push origin feature/your-feature-name`
7. **Create a Pull Request** with:
   - Clear description of changes
   - Reference to any related issues
   - Evidence that tests pass locally

### Pull Request Guidelines

- Keep PRs focused on a single concern
- Update tests for any new functionality
- Update the `.version` file if making significant changes
- Ensure all make targets succeed (make all)
- Provide clear commit messages
- Reference related issues using GitHub issue syntax

### Code Style

- Follow existing Go code patterns in the repository
- Use meaningful variable and function names
- Add comments for complex logic
- Keep functions focused and testable

## Related Repositories

- **hms-canary-charts** - Test Helm chart build workflows - https://github.com/Cray-HPE/hms-canary-charts
- **hms-build-image-workflows** - Production image build workflows - https://github.com/Cray-HPE/hms-build-image-workflows
- **hms-build-metadata-action** - Build metadata action - https://github.com/Cray-HPE/hms-build-metadata-action
- **hms-build-environment** - Base build environment image - https://github.com/Cray-HPE/hms-build-environment
- **hms-nightly-integration** - Integration test workflows - https://github.com/Cray-HPE/hms-nightly-integration

## For More Information

- [hms-build-image-workflows README](https://github.com/Cray-HPE/hms-build-image-workflows/blob/main/README.md) - Detailed workflow documentation
- [hms-build-metadata-action README](https://github.com/Cray-HPE/hms-build-metadata-action/blob/main/README.md) - Metadata action documentation
- [hms-build-environment README](https://github.com/Cray-HPE/hms-build-environment/blob/main/README.md) - Build environment documentation

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
