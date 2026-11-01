#!/bin/bash
set -e # Exit on error

# ==========================================
# DEFAULT CONFIGURATION & EXAMPLES
# ==========================================
DEFAULT_USER="$USER"
DEFAULT_HOST="legacy.internal.example"
DEFAULT_REMOTE_PATH="/home/tomcat-writable/webapp/logs"
DEFAULT_DEST_DIR="$HOME/Downloads"

echo "=== Log Fetcher ==="

echo ""
REMOTE_USER=""
while [ -z "$REMOTE_USER" ]; do
    ACTIVE_DEFAULT="${DEFAULT_USER:-$USER}"

    read -p "Enter Remote User [default: $ACTIVE_DEFAULT]: " INPUT_USER
    REMOTE_USER="${INPUT_USER:-$ACTIVE_DEFAULT}"
    [ -z "$REMOTE_USER" ] && echo "--> Error: User cannot be empty."
done
echo "Configured Remote User: $REMOTE_USER"

echo ""
REMOTE_HOST=""
while [ -z "$REMOTE_HOST" ]; do
    # If DEFAULT_HOST is set, prefix it with "default: ". If empty, use the example.
    PROMPT_CUE="${DEFAULT_HOST:+default: $DEFAULT_HOST}"
    PROMPT_CUE="${PROMPT_CUE:-example: remote-host.example}"

    read -p "Enter Remote Host [$PROMPT_CUE]: " INPUT_HOST
    REMOTE_HOST="${INPUT_HOST:-$DEFAULT_HOST}"
    [ -z "$REMOTE_HOST" ] && echo "--> Error: Host cannot be empty."
done
echo "Configured Remote Host: $REMOTE_HOST"

echo ""
REMOTE_PATH=""
while [ -z "$REMOTE_PATH" ]; do
    PROMPT_CUE="${DEFAULT_REMOTE_PATH:+default: $DEFAULT_REMOTE_PATH}"
    PROMPT_CUE="${PROMPT_CUE:-example: /path/to/logs or /path/to/logs/file.log}"

    read -p "Enter Remote File or Folder [$PROMPT_CUE]: " INPUT_PATH
    REMOTE_PATH="${INPUT_PATH:-$DEFAULT_REMOTE_PATH}"
    # Strip trailing slashes (but keep "/" itself)
    while [ "${#REMOTE_PATH}" -gt 1 ] && [ "${REMOTE_PATH%/}" != "$REMOTE_PATH" ]; do
        REMOTE_PATH="${REMOTE_PATH%/}"
    done
    [ -z "$REMOTE_PATH" ] && echo "--> Error: Remote path cannot be empty."
done
echo "Configured Remote Path: $REMOTE_PATH"

echo ""
# Use DEFAULT_DEST_DIR if set; otherwise, fall back to the current working directory ($PWD)
ACTIVE_DEFAULT="${DEFAULT_DEST_DIR:-$PWD}"
read -p "Enter Local Destination Directory [default: $ACTIVE_DEFAULT]: " INPUT_DEST
LOCAL_DEST_DIR="${INPUT_DEST:-$ACTIVE_DEFAULT}"

# Verify local destination exists and is writable
mkdir -p "$LOCAL_DEST_DIR"
if [ ! -w "$LOCAL_DEST_DIR" ]; then
    echo "--> Error: Destination directory '$LOCAL_DEST_DIR' is not writable."
    exit 1
fi
echo "Configured Local Dest Dir: $LOCAL_DEST_DIR"

# Config and paths
REMOTE_PARENT=$(dirname "${REMOTE_PATH}")
REMOTE_BASE=$(basename "${REMOTE_PATH}")
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
FOLDER_NAME="$(basename "${REMOTE_PARENT}")_${REMOTE_BASE}_${TIMESTAMP}"
ARCHIVE_NAME="${FOLDER_NAME}.tar.gz"

REMOTE_ARCHIVE="/home/${REMOTE_USER}/${ARCHIVE_NAME}"
LOCAL_ARCHIVE="${LOCAL_DEST_DIR}/${ARCHIVE_NAME}"
UNPACK_DIR="${LOCAL_DEST_DIR}/${FOLDER_NAME}"

# Display Execution Summary early for user review
echo ""
echo "=== Execution Summary ==="
echo "Remote User:            $REMOTE_USER"
echo "Remote Host:            $REMOTE_HOST"
echo "Remote Path:            $REMOTE_PATH"
echo "Remote Archive (kept):  $REMOTE_ARCHIVE"
echo "Local Dest Dir:         $LOCAL_DEST_DIR"
echo "Local Unpack Folder:    $UNPACK_DIR"
echo "========================="

# Ask for confirmation before execution
echo ""
CONFIRM=""
while [ -z "$CONFIRM" ]; do
    read -p "Proceed with these settings? (y/n): " CONFIRM
    case "$CONFIRM" in
        [Yy]* )
            ;;
        [Nn]* )
            echo "Execution cancelled by user."
            exit 0
            ;;
        * )
            echo "--> Error: Please enter 'y' for yes or 'n' for no."
            CONFIRM=""
            ;;
    esac
done

# Create remote archive (-t forces TTY for sudo password)
# Archive relative to the parent so entries are "<base>" (file) or "<base>/..." (folder)
echo ""
echo "==> Creating archive on ${REMOTE_HOST}..."
ssh -t "${REMOTE_USER}@${REMOTE_HOST}" "
  sudo tar -czvf '${REMOTE_ARCHIVE}' -C '${REMOTE_PARENT}' '${REMOTE_BASE}' && \
  sudo chown ${REMOTE_USER}:${REMOTE_USER} '${REMOTE_ARCHIVE}'
"

# Download archive
echo ""
echo "==> Downloading archive..."
scp "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_ARCHIVE}" "${LOCAL_ARCHIVE}"

# Create specific folder for extraction
mkdir -p "${UNPACK_DIR}"

# Unpack archive into the dedicated folder
# A folder's contents go directly into UNPACK_DIR; a single file is extracted as-is
if tar -tzf "${LOCAL_ARCHIVE}" | grep -q "^${REMOTE_BASE}/"; then
    STRIP_COMPONENTS=1
else
    STRIP_COMPONENTS=0
fi
echo ""
echo "==> Unpacking to ${UNPACK_DIR}..."
tar -xzvf "${LOCAL_ARCHIVE}" -C "${UNPACK_DIR}" --strip-components "${STRIP_COMPONENTS}"

echo ""
echo "==> Done!"
