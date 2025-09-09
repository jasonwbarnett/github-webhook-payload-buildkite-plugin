# GitHub Webhook Payload Buildkite Plugin

[![GitHub Release](https://img.shields.io/github/release/jasonwbarnett/github-webhook-payload-buildkite-plugin.svg)](https://github.com/jasonwbarnett/github-webhook-payload-buildkite-plugin/releases)

A Buildkite plugin that retrieves GitHub webhook payload data from the Buildkite GraphQL API and sets the `BUILDKITE_COMMIT_BEFORE` environment variable. This plugin demonstrates how to access the original GitHub webhook payload data that may not be exposed by Buildkite's built-in environment variables.

This plugin is particularly useful when you need to:
- Compare changes between commits in your build pipeline
- Run differential testing or analysis
- Generate changelogs or release notes
- Perform optimized builds based on file changes
- Access GitHub webhook data that Buildkite doesn't expose by default

## Example

The plugin must be used in the same step that consumes the data it provides:

```yaml
steps:
  - label: ":gear: Build with commit comparison"
    plugins:
      - jasonwbarnett/github-webhook-payload#v1.0.0: ~
    command: |
      echo "Current commit: $BUILDKITE_COMMIT"
      echo "Previous commit: $BUILDKITE_COMMIT_BEFORE"
      
      # Example: Run tests only on changed files
      git diff --name-only $BUILDKITE_COMMIT_BEFORE $BUILDKITE_COMMIT | grep '\.js$' | xargs npm test
```

### Multiple Steps

If you need the webhook data in multiple steps, include the plugin in each step:

```yaml
steps:
  - label ":test_tube: Run tests on changed files"
    plugins:
      - jasonwbarnett/github-webhook-payload#v1.0.0: ~
    command: |
      git diff --name-only $BUILDKITE_COMMIT_BEFORE $BUILDKITE_COMMIT | grep '\.test\.js$' | xargs npm test

  - label: ":package: Build only if source changed"
    plugins:
      - jasonwbarnett/github-webhook-payload#v1.0.0: ~
    command: |
      if git diff --name-only $BUILDKITE_COMMIT_BEFORE $BUILDKITE_COMMIT | grep -q '^src/'; then
        npm run build
      else
        echo "No source changes detected, skipping build"
      fi
```

## How it works

1. The plugin queries the Buildkite GraphQL API to retrieve the original GitHub webhook payload for the current build
2. It extracts the `before` SHA from the GitHub webhook payload (data not available in Buildkite's standard environment variables)
3. Sets the `BUILDKITE_COMMIT_BEFORE` environment variable for use in the current build step
4. Provides detailed logging and error handling for troubleshooting

This approach demonstrates how to access any field from the original GitHub webhook payload that Buildkite doesn't expose through its built-in environment variables. The plugin can be extended to extract other webhook data such as pull request information, repository details, or custom GitHub App data.

:warning: **Note**: This plugin only works with builds triggered by GitHub webhooks. For manually triggered builds or builds from other sources, the plugin will log an informational message and continue without setting the environment variable.

## Prerequisites

- A Buildkite API token with GraphQL access must be available as `BUILDKITE_API_TOKEN`
- The build must be triggered by a GitHub webhook
- `curl` and `jq` must be available in the build environment

You can use the `buildkite/set-buildkite-api-token` plugin to securely provide the API token:

```yaml
steps:
  - plugins:
      - buildkite/set-buildkite-api-token#v1.0.0: ~
      - jasonwbarnett/github-webhook-payload#v1.0.0: ~
```

## Configuration

### `debug` (optional, boolean)

Enable debug logging to troubleshoot issues with webhook payload retrieval.

Default: `false`

**Example:**
```yaml
steps:
  - plugins:
      - jasonwbarnett/github-webhook-payload#v1.0.0:
          debug: true
```

## Environment Variables

### Set by the plugin

- `BUILDKITE_COMMIT_BEFORE`: The SHA of the commit before the current commit, extracted from the GitHub webhook payload

### Required

- `BUILDKITE_API_TOKEN`: Buildkite API token with GraphQL access
- `BUILDKITE_BUILD_ID`: Build ID (automatically provided by Buildkite)

## Error Handling

The plugin includes comprehensive error handling and will:

- Validate API responses and JSON parsing
- Check for valid SHA format (40-character hex string)
- Log detailed error messages when webhook payload cannot be retrieved
- Continue the build even if the webhook payload is unavailable (non-fatal errors)

Common scenarios where the plugin will log warnings but continue:
- Build not triggered by webhook
- Webhook payload doesn't contain commit information
- Network issues accessing the Buildkite API

## Development

To test the plugin locally:

```bash
# Set required environment variables
export BUILDKITE_API_TOKEN="your-api-token"
export BUILDKITE_BUILD_ID="your-build-id"

# Run the environment hook
./hooks/environment
```

To run tests:

```bash
# Run the plugin directly
./hooks/environment

# Test with different build IDs
BUILDKITE_BUILD_ID="test-build-id" ./hooks/environment
```

## License

MIT (see [LICENSE](LICENSE))