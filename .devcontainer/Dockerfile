FROM mcr.microsoft.com/devcontainers/go:1-1.23-bookworm

ARG HUGO_VERSION=0.136.5

# Install Hugo extended, matching the version used in .github/workflows/buildDeploy.yaml
RUN curl -fsSL -o /tmp/hugo.tar.gz \
        "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
    && tar -xzf /tmp/hugo.tar.gz -C /usr/local/bin hugo \
    && rm -f /tmp/hugo.tar.gz
