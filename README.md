author: "Repository Operations & Automation Engineering"
date: "2026-11-07"
repository_reference: "SCRIPT-GIT-BUNDLE-P2P-V1"
status: "Active Automation Script"
Bash Automation Script: Air-Gapped Git Bundle Sync (git-p2p-sync.sh)
This script automates generating and applying incremental Git bundle files for sneakernet and air-gapped transport. It operates in two modes: --export to write incremental changes to external media, and --import to verify and integrate incoming bundles safely into a staging branch.
1. Script Implementation
#!/usr/bin/env bash
# ==============================================================================
# Script Name : git-p2p-sync.sh
# Description : Export and import incremental Git bundles for offline sync.
# Usage       : ./git-p2p-sync.sh --export /path/to/media [target_branch]
#               ./git-p2p-sync.sh --import /path/to/media/bundle_file.bundle
# ==============================================================================

set -euo pipefail

# Print usage instructions
usage() {
    cat << EOF
Usage: $(basename "$0") [OPTION] [PATH] [BRANCH]

Options:
  --export <media_dir> [branch]   Export commits since last tag into a bundle.
                                  Defaults to current branch if unspecified.
  --import <bundle_file>          Verify and merge a bundle file into p2p-incoming.
  -h, --help                      Show this help message.
EOF
    exit 1
}

# Ensure we are inside a Git repository
check_git_repo() {
    if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
        echo "[ERROR] Must be executed within a valid Git repository." >&2
        exit 1
    fi
}

# Handle export workflow
export_bundle() {
    local target_dir="$1"
    local branch="${2:-$(git rev-parse --abbrev-ref HEAD)}"
    
    if [ ! -d "$target_dir" ]; then
        echo "[ERROR] Target output directory does not exist: $target_dir" >&2
        exit 1
    fi

    local repo_name
    repo_name="$(basename "$(git rev-parse --show-toplevel)")"
    local timestamp
    timestamp="$(date +%Y%m%d_%H%M%S)"
    local last_sync_tag="p2p-sync-last"
    local bundle_name="${repo_name}_${timestamp}.bundle"
    local bundle_path="${target_dir}/${bundle_name}"

    echo "[INFO] Preparing export for branch '$branch' in repository '$repo_name'..."

    # Determine commit range
    if git rev-parse "$last_sync_tag" >/dev/null 2>&1; then
        echo "[INFO] Found previous sync tag '$last_sync_tag'. Generating incremental bundle..."
        git bundle create "$bundle_path" "${last_sync_tag}..${branch}"
    else
        echo "[WARNING] No previous sync tag found. Generating full history bundle..."
        git bundle create "$bundle_path" "$branch"
    fi

    # Update or create the sync tracking tag
    git tag -f "$last_sync_tag" "$branch"
    echo "[SUCCESS] Bundle generated: $bundle_path"
    echo "[INFO] Sync tag '$last_sync_tag' updated to HEAD."
}

# Handle import workflow
import_bundle() {
    local bundle_file="$1"

    if [ ! -f "$bundle_file" ]; then
        echo "[ERROR] Bundle file not found: $bundle_file" >&2
        exit 1
    fi

    echo "[INFO] Verifying bundle integrity..."
    if ! git bundle verify "$bundle_file"; then
        echo "[ERROR] Bundle verification failed. Rejecting update." >&2
        exit 1
    fi

    local staging_branch="p2p-incoming"
    echo "[INFO] Fetching bundle commits into staging branch '$staging_branch'..."
    git fetch "$bundle_file" "*:refs/heads/$staging_branch"

    echo "[INFO] Attempting merge from '$staging_branch' into current branch..."
    if git merge "$staging_branch" --no-commit --no-ff; then
        echo "[SUCCESS] Merge staged cleanly. Verify changes and run 'git commit'."
    else
        echo "[WARNING] Merge conflicts detected. Resolve manually, then commit."
    fi
}

# Main Execution Switchboard
main() {
    check_git_repo

    if [ $# -lt 2 ]; then
        usage
    fi

    case "$1" in
        --export)
            export_bundle "$2" "${3:-}"
            ;;
        --import)
            import_bundle "$2"
            ;;
        -h|--help)
            usage
            ;;
        *)
            echo "[ERROR] Invalid option: $1" >&2
            usage
            ;;
    esac
}

main "$@"

2. Usage Instructions
Step 1: Grant Execution Permissions
chmod +x git-p2p-sync.sh

Step 2: Export Incremental Bundle to USB / External Media
To create an incremental bundle file directly on portable media:
./git-p2p-sync.sh --export /media/usb_drive main

Step 3: Import Bundle File on Target Air-Gapped Host
To verify and merge the incoming bundle file:
./git-p2p-sync.sh --import /media/usb_drive/repo_name_20261107_100000.bundle

# git-p2p-sync.sh
This script automates generating and applying incremental Git bundle files for sneakernet and air-gapped transport. It operates in two modes: --export to write incremental changes to external media, and --import to verify and integrate incoming bundles safely into a staging branch.
