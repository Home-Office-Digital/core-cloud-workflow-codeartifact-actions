# core-cloud-workflow-codeartifact-actions

A single reusable GitHub composite action, `codeartifact-connect`, for authenticating to AWS CodeArtifact — used for both publishing to and pulling from CodeArtifact repositories.

The action handles AWS OIDC authentication and resolving the CodeArtifact repository endpoint/token.

CodeArtifact supports eight package formats — `npm`, `pypi`, `maven`, `nuget`, `ruby`, `swift`, `cargo`, and `generic`

## Actions

### Pre-requisites

Create a Github secret called CODEARTIFACT_AWS_ACCOUNT_ID and populate it with your AWS Account ID.

If using the Core Cloud shared CodeArtifact repo, also create a Github secret called SHARED_CODEARTIFACT_AWS_ACCOUNT_ID and populate it with the account ID. For security reasons, this can only be found in the Core Cloud docs.

### Example workflow for publishing

Resolves credentials for publishing, then you run the publish command for your language. Only one `format` block would appear in a real workflow. The actual format-specific usage examples are just a guide.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - uses: Home-Office-Digital/core-cloud-workflow-codeartifact-actions/actions/codeartifact-connect@main
    with:
      codeartifact-domain: your-domain
      codeartifact-repository: your-repo
      role-account-id: ${{ secrets.CODEARTIFACT_AWS_ACCOUNT_ID }}
      role-name: your-role-name
      # Uncomment if using shared CodeArtifact repo
      # codeartifact-domain-owner: ${{ secrets.SHARED_CODEARTIFACT_AWS_ACCOUNT_ID }}
      format: pypi # npm | pypi | maven | nuget | ruby | swift | cargo | generic

  # pypi
  - run: |
      python -m twine upload --non-interactive \
        --repository-url "$CODEARTIFACT_REPOSITORY_ENDPOINT" \
        -u aws -p "$CODEARTIFACT_AUTH_TOKEN" dist/*

  # npm
  - run: |
      npm config set "//${CODEARTIFACT_REPOSITORY_ENDPOINT#https://}:_authToken" "$CODEARTIFACT_AUTH_TOKEN"
      npm publish --registry "$CODEARTIFACT_REPOSITORY_ENDPOINT"

  # maven — settings.xml server id must match the id used in altDeploymentRepository
  - run: |
      mkdir -p ~/.m2
      cat > ~/.m2/settings.xml <<EOF
      <settings>
        <servers>
          <server>
            <id>codeartifact</id>
            <username>aws</username>
            <password>${CODEARTIFACT_AUTH_TOKEN}</password>
          </server>
        </servers>
      </settings>
      EOF
      mvn deploy -DaltDeploymentRepository="codeartifact::default::${CODEARTIFACT_REPOSITORY_ENDPOINT}"

  # nuget
  - run: |
      dotnet nuget add source "$CODEARTIFACT_REPOSITORY_ENDPOINT" \
        -n codeartifact -u aws -p "$CODEARTIFACT_AUTH_TOKEN" --store-password-in-clear-text
      dotnet nuget push "**/*.nupkg" --source codeartifact --api-key AWS

  # ruby (rubygems) — AWS docs use a named key in ~/.gem/credentials
  - run: |
      mkdir -p ~/.gem
      printf -- '---\n:codeartifact: Bearer %s\n' "$CODEARTIFACT_AUTH_TOKEN" > ~/.gem/credentials
      chmod 0600 ~/.gem/credentials
      gem push your-gem-1.0.0.gem --host "$CODEARTIFACT_REPOSITORY_ENDPOINT" --key codeartifact

  # swift
  - run: |
      swift package-registry login "$CODEARTIFACT_REPOSITORY_ENDPOINT" --token "$CODEARTIFACT_AUTH_TOKEN"
      swift package-registry publish your-package 1.0.0

  # cargo
  - run: |
      mkdir -p ~/.cargo
      cat >> ~/.cargo/config.toml <<EOF
      [registries.codeartifact]
      index = "sparse+${CODEARTIFACT_REPOSITORY_ENDPOINT}"
      EOF
      cat >> ~/.cargo/credentials.toml <<EOF
      [registries.codeartifact]
      token = "Bearer ${CODEARTIFACT_AUTH_TOKEN}"
      EOF
      cargo publish --registry codeartifact

  # generic — no package-manager client exists for this format, so publishing goes through the AWS CLI directly authenticated by the assumed IAM role, not the CodeArtifact bearer token.
  - run: |
      aws codeartifact publish-package-version \
        --domain your-domain --domain-owner "$CODEARTIFACT_AWS_ACCOUNT_ID" \
        --repository your-repo --format generic \
        --namespace your-namespace --package your-package --package-version 1.0.0 \
        --asset-content ./dist/your-package-1.0.0.tar.gz --asset-name your-package-1.0.0.tar.gz
    env:
      CODEARTIFACT_AWS_ACCOUNT_ID: ${{ secrets.CODEARTIFACT_AWS_ACCOUNT_ID }}

  # generic, pushing to the shared CodeArtifact repo instead — domain-owner must be the shared account, not your own
  - run: |
      aws codeartifact publish-package-version \
        --domain shared-domain --domain-owner "$SHARED_CODEARTIFACT_AWS_ACCOUNT_ID" \
        --repository shared-repo --format generic \
        --namespace your-namespace --package your-package --package-version 1.0.0 \
        --asset-content ./dist/your-package-1.0.0.tar.gz --asset-name your-package-1.0.0.tar.gz
    env:
      SHARED_CODEARTIFACT_AWS_ACCOUNT_ID: ${{ secrets.SHARED_CODEARTIFACT_AWS_ACCOUNT_ID }}
```

### Example workflow for pulling

Resolves credentials for installing, then you run the install command for your language. Only one `format` block would appear in a real workflow — they're all shown here for reference.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: Home-Office-Digital/core-cloud-workflow-codeartifact-actions/actions/codeartifact-connect@main
    with:
      codeartifact-domain: your-domain
      codeartifact-repository: your-code-artifact-repo-name
      role-account-id: ${{ secrets.AWS_ACCOUNT_ID }}
      role-name: your-role
      # Uncomment if using shared CodeArtifact repo
      # codeartifact-domain-owner: ${{ secrets.SHARED_CODEARTIFACT_AWS_ACCOUNT_ID }}
      format: pypi # npm | pypi | maven | nuget | ruby | swift | cargo | generic

  # pypi — note the trailing "simple/", which pip's index requires but the
  # raw CodeArtifact endpoint does not include
  - run: |
      pip install --index-url "https://aws:${CODEARTIFACT_AUTH_TOKEN}@${CODEARTIFACT_REPOSITORY_ENDPOINT#https://}simple/" your-package

  # npm
  - run: |
      npm config set "//${CODEARTIFACT_REPOSITORY_ENDPOINT#https://}:_authToken" "$CODEARTIFACT_AUTH_TOKEN"
      npm install your-package --registry "$CODEARTIFACT_REPOSITORY_ENDPOINT"

  # maven — same settings.xml server entry as publishing; add a <repository>
  # in your pom.xml (or settings.xml <mirror>) pointing at the same id/URL
  - run: |
      mkdir -p ~/.m2
      cat > ~/.m2/settings.xml <<EOF
      <settings>
        <servers>
          <server>
            <id>codeartifact</id>
            <username>aws</username>
            <password>${CODEARTIFACT_AUTH_TOKEN}</password>
          </server>
        </servers>
      </settings>
      EOF
      mvn dependency:get -Dartifact=com.example:your-artifact:1.0.0 -DremoteRepositories=codeartifact::default::${CODEARTIFACT_REPOSITORY_ENDPOINT}

  # nuget
  - run: |
      dotnet nuget add source "$CODEARTIFACT_REPOSITORY_ENDPOINT" \
        -n codeartifact -u aws -p "$CODEARTIFACT_AUTH_TOKEN" --store-password-in-clear-text
      dotnet restore --source codeartifact

  # ruby (rubygems)
  - run: |
      gem install your-gem --host "https://aws:${CODEARTIFACT_AUTH_TOKEN}@${CODEARTIFACT_REPOSITORY_ENDPOINT#https://}"

  # swift
  - run: |
      swift package-registry login "$CODEARTIFACT_REPOSITORY_ENDPOINT" --token "$CODEARTIFACT_AUTH_TOKEN"
      # then reference the package as a registry dependency in Package.swift

  # cargo
  - run: |
      mkdir -p ~/.cargo
      cat >> ~/.cargo/config.toml <<EOF
      [registries.codeartifact]
      index = "sparse+${CODEARTIFACT_REPOSITORY_ENDPOINT}"
      EOF
      cat >> ~/.cargo/credentials.toml <<EOF
      [registries.codeartifact]
      token = "Bearer ${CODEARTIFACT_AUTH_TOKEN}"
      EOF
      cargo add your-crate --registry codeartifact

  # generic — same as publishing, this authenticates via the assumed IAM role
  # directly rather than the CodeArtifact bearer token
  - run: |
      aws codeartifact get-package-version-asset \
        --domain your-domain --domain-owner "$CODEARTIFACT_AWS_ACCOUNT_ID" \
        --repository your-code-artifact-repo-name --format generic \
        --namespace your-namespace --package your-package --package-version 1.0.0 \
        --asset-name your-package-1.0.0.tar.gz ./your-package-1.0.0.tar.gz
    env:
      CODEARTIFACT_AWS_ACCOUNT_ID: ${{ secrets.CODEARTIFACT_AWS_ACCOUNT_ID }}

  # generic, pulling from the shared CodeArtifact repo instead — domain-owner must be the shared account, not your own
  - run: |
      aws codeartifact get-package-version-asset \
        --domain shared-domain --domain-owner "$SHARED_CODEARTIFACT_AWS_ACCOUNT_ID" \
        --repository shared-repo --format generic \
        --namespace your-namespace --package your-package --package-version 1.0.0 \
        --asset-name your-package-1.0.0.tar.gz ./your-package-1.0.0.tar.gz
    env:
      SHARED_CODEARTIFACT_AWS_ACCOUNT_ID: ${{ secrets.SHARED_CODEARTIFACT_AWS_ACCOUNT_ID }}
```
