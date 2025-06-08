# Plan for Supporting Long Videos in SAM2 Demo

This document outlines the tasks required to adapt the original SAM2 demo application to handle long videos (2 hours or more). The demo currently limits uploads to 70 MB and truncates videos to roughly 10 seconds for segmentation. The following tasks describe backend and frontend changes needed to efficiently process full-length videos while running on modest hardware.

## 1. Backend Updates

### 1.1 Video Handling
- **Remove hardcoded duration limit**: Update `MAX_UPLOAD_VIDEO_DURATION` in `demo/backend/server/app_conf.py` to allow long clips. Allow configuration via environment variable.
- **Segmented processing**: Instead of loading the full video into memory, use `ffmpeg` to split the input into smaller chunks (e.g., 30–60 s). Process each chunk sequentially and store intermediate results to disk.
- **Resume context**: Extend `InferenceAPI` to keep state across chunks. Cached memory tree representations (e.g., features) should be saved after processing each segment so the following chunk can resume propagation without recomputing past results.
- **Progress and cancellation**: Expose endpoints to query progress and cancel long jobs. Propagate session statistics (processed frames, estimated remaining time) via GraphQL.
- **Output management**: Implement routines to join processed segments into final tracks. Provide endpoints to download per-object clips, optionally at reduced bitrate for easier transfer.

### 1.2 Resource Management
- **GPU memory**: Ensure each chunk fits into available GPU VRAM (8 GB on a 4060). Offload intermediate tensors to CPU when not needed. Clear caches between segments.
- **Session expiration**: Increase allowed session TTL for long runs. Persist session states on disk to survive backend restarts.
- **Background worker**: Use a queue (e.g., Celery or simple thread pool) to process long-running jobs asynchronously. This keeps API responsive while heavy computation happens in the background.

## 2. Frontend Updates

### 2.1 Upload Workflow
- **Chunked upload**: Instead of uploading the entire two‑hour video, let the browser slice the file into manageable pieces (using `Blob.slice`) and upload sequentially. Display progress to the user.
- **Input validation**: Show warnings for extremely large files and provide an estimated processing time before submission.
- **Session management**: Once the first chunk is processed, start showing initial results while remaining chunks upload in the background.

### 2.2 Interaction and Playback
- **Extended timeline**: Replace the 10‑second preview timeline with a scrollable timeline covering the full video. Allow the user to jump to any frame for adding prompts or reviewing output.
- **Tracking indicators**: Display current processing status (e.g., “chunk 3/120”). Offer controls to pause/resume propagation.
- **Result export**: Provide a UI to download extracted clips for each tracked object once processing completes. Support a playlist or zipped archive for convenience.

## 3. Deployment Considerations
- **Docker setup**: Update `docker-compose.yaml` to expose environment variables controlling chunk length, session TTL, and storage paths. Mount a data volume with enough space for intermediate files.
- **Hardware usage**: Document how to adjust batch sizes or chunk length based on available VRAM. Include notes on using CPU fallback when GPU memory is insufficient.

## 4. Suggested Milestones
1. **Backend configuration**: make duration limit configurable and ensure long videos load without truncation.
2. **Chunked processing pipeline**: implement segmentation per chunk and verify results on ~5 minute videos.
3. **Front‑end chunked upload and progress UI**.
4. **End‑to‑end test** with a full soccer match.

Once these steps are completed, the demo should be capable of handling two‑hour videos with the ability to track players and the ball throughout the entire match.
