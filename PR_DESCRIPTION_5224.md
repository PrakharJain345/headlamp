# PR Description: Multiplexer Stability and Terminal Shell Fallback

## Summary
This PR addresses issue #5224 by improving the robustness of the WebSocket multiplexer and terminal shell probing. It prevents a critical issue where failed shell probes (or any malformed WebSocket message) could trigger a cluster-wide disconnection, dropping all active resource-watching streams for the session.

## Related Issue
Fixes #5224

## Changes
- **Backend (`backend/cmd/multiplexer.go`)**:
    - **Fatal vs Non-Fatal Error Isolation**: Refactored `HandleClientWebSocket` main loop to distinguish between fatal connection errors and non-fatal parsing/authentication errors. The loop now logs and continues processing instead of breaking the entire session.
    - **JSON Unmarshal Robustness**: Updated `sendIfNewResourceVersion` to gracefully handle non-JSON messages (common in terminal binary streams or non-standard cluster responses) by skipping version checks instead of returning a fatal error.
    - **Auth Resilience**: Ensured that missing or invalid tokens for one cluster do not terminate the multiplexed session for other clusters.
- **Backend Tests (`backend/cmd/multiplexer_test.go`)**:
    - Updated tests to match the new `readClientMessage` signature and verified that the main loop remains active when encountering invalid JSON or missing tokens.
- **Frontend (`frontend/src/components/common/Terminal.tsx`)**:
    - **Improved Shell Failure Detection**: Updated `onData` to treat any message on the server error channel (`Channel.ServerError`) received before a successful connection as an initialization failure.
    - **Reliable Fallback**: This ensures that the terminal correctly tries alternative shells (e.g., `sh`) even if the Kubernetes cluster returns non-standard error codes or exit statuses during initialization.

## Steps to Test
1. **Multiplexer Stability**:
    - Open Headlamp with multiple resource watches (e.g., Pods, Services, Deployments).
    - Send a malformed JSON message to the `/wsMultiplexer` endpoint (or simulate a corrupted message).
    - Verify that only the specific stream associated with the error fails (if any), while the rest of the dashboard remains connected and functional.
2. **Terminal Shell Fallback**:
    - Open a terminal for a pod that does not have `bash` installed.
    - Verify that the terminal correctly detects the failure and automatically falls back to alternative shells (e.g., `sh`).
    - Verify that this fallback process does not cause other parts of the dashboard to disconnect.

## Screenshots
*(N/A for backend changes, but verifies visual stability of the dashboard during terminal failures)*

## Notes for the Reviewer
The primary fix is in the backend's WebSocket handling. By moving away from a "fail-fast" approach for the entire session, we ensure that stream-specific issues (like binary terminal data leaking into a JSON stream) are isolated and do not compromise the overall user experience.
