# AAE5303 Individual Report on Six Qualitative Indicators

---
## 1. Metadata

| Item | Details |
|------|---------|
| Course | AAE5303 |
| Report type | Individual Report on Six Qualitative Indicators |
| Student name | FU Xiaohe |
| Student ID | 25048943G |
| Team name / group number | 404_found |
| Team repo URL | https://github.com/box717/AAE5303_Group.git |
| My role in the team | 3D reconstruction pipeline implementation |
| My primary module | 3D Gaussian Splatting reconstruction |
| Main dataset / scene used | AMtown02 (MARS-LVIG-based UAV aerial sequence) |
| Main tools used | Cursor / Docker / ROS Noetic / OpenSplat / COLMAP / WSL2 |
| Main evidence paths | `colmap_test/`, `test_images/`, `splat.ply`, `extracted_AMtown02/` |
| Commit hash(es) / issue / notebook / slide links | https://github.com/Tong-Yuru/AAE5303-404_FOUND_Presentation |
| Declared AI use | GitHub Copilot, ChatGPT-4, Cursor Composer |

---

## 2. Individual Contribution Statement

| Item | Content |
|------|---------|
| Team result | The team built an end-to-end 3D reconstruction pipeline for low-altitude aerial perception, integrating VO / VSLAM (ORB-SLAM3), 3D Gaussian Splatting (OpenSplat), and 2D semantic segmentation (U-Net), with benchmarking on MARS-LVIG / UAVScenes dataset. |
| My specific role | I personally handled the 3D Gaussian Splatting reconstruction module: extracted compressed images from ROS bag using Docker + ROS Noetic, performed COLMAP sparse reconstruction, trained OpenSplat model with CPU mode, and recorded loss metrics. |
| My strongest evidence | `colmap_test/` folder containing COLMAP database and `sparse/0/` text files; `splat.ply` (14 MB, 165,253 Gaussians); terminal logs showing training loss convergence from ~0.048 to 0.0233; extracted image directory `extracted_AMtown02/` (7499 PNG frames). |
| My main learning focus | This report mainly demonstrates my growth in toolchain reproducibility (Docker/WSL/ROS), AI-assisted debugging for COLMAP initialization and OpenSplat OOM issues, and responsible AI use in engineering problem-solving. |

---

## 3. Project Snapshot

### 3.1 Project goal

The team aimed to build an end-to-end pipeline for low-altitude aerial perception, integrating Visual Odometry / VSLAM (ORB-SLAM3), 3D Gaussian Splatting (OpenSplat), and 2D semantic segmentation (U-Net) on MARS-LVIG / UAVScenes dataset. The goal was to enable photorealistic 3D scene reconstruction from drone-captured imagery, with benchmarking using metrics such as RMSE, IoU, and mIoU.

### 3.2 Pipeline overview

The pipeline consists of three main modules: (1) VO/VSLAM module (ORB-SLAM3) for camera pose estimation, (2) 3DGS module (OpenSplat + COLMAP) for photorealistic 3D reconstruction, and (3) segmentation module (U-Net) for semantic annotation of aerial scenes. These modules are integrated end-to-end, with benchmarking performed on UAVScenes benchmark.

### 3.3 My position in the pipeline

I was responsible for the 3DGS reconstruction module. My work sat downstream of the VO/VSLAM module (receiving camera poses) and upstream of the evaluation module (providing 3D models for benchmarking). Specifically, I handled the complete reconstruction pipeline: extracting compressed images from ROS bag files, running COLMAP sparse reconstruction, training OpenSplat models, and generating PLY outputs for team evaluation.

### 3.4 Reproducibility snapshot

| Item | Details |
|------|---------|
| Main environment | WSL2 Ubuntu 22.04 / Docker / ROS Noetic (in container) |
| Key scripts / folders | `colmap_test/`, `extracted_AMtown02/`, `test_images/` |
| Main evaluation scripts | OpenSplat training command, COLMAP commands |
| Main figures in this report | (loss table, terminal logs, PLY header) |
| What another person should run first | `docker run -it --rm -v /path/to/bag:/data osrf/ros:noetic-desktop-full` + `rosbag2image -i AMtown02.bag -t /left_camera/image/compressed -o extracted_AMtown02` |

---

## 4. Six-Indicator Summary Matrix

| Indicator | Before-course (optional) | End-of-course | Main evidence | Main lesson |
|-----------|------------------------|---------------|---------------|--------------|
| 1. Tooling & Reproducible Workflow Readiness | 2/5 | 4/5 | Docker + WSL2 + ROS Noetic setup, `colmap_test/` folder, extraction command | Containerization is essential for reproducibility; WSL2 provides a stable Linux environment for ROS/3DGS workflows |
| 2. Prompting Strategy & AI Co-Creation | 2/5 | 4/5 | Prompt logs for debugging offset error, OOM fix, COLMAP initialization | Effective prompting requires providing context (logs, errors, documentation) and iterative refinement |
| 3. Verification, Documentation & Technical Judgment | 2/5 | 4/5 | Cross-checked OpenSplat documentation, verified COLMAP outputs, rejected weak AI suggestions | Always verify AI-generated commands against official docs; ground output in executable evidence |
| 4. Debugging, Iteration & Problem Solving | 2/5 | 4/5 | Offset error fix via Docker container, OOM fix via `-d 2`, COLMAP init via `sequential_matcher` | Structured troubleshooting (read logs → hypothesize → test one change) is essential; regression is inevitable but manageable |
| 5. Integration & Benchmark-Driven Improvement | 2/5 | 4/5 | Connected OpenSplat output to team benchmark, loss table (0.048→0.0233), 165k Gaussians, 14 MB model | 3DGS reconstruction quality correlates with image overlap and resolution; moderate downsampling balances memory and quality |
| 6. Responsible AI Use, Reflection & Redesign | 2/5 | 4/5 | Hallucination logs (incorrect rosbags suggestions), OOM diagnosis, redesign plan for full dataset | AI can accelerate engineering but cannot replace understanding of fundamentals; human oversight is critical |

---

## 5. Indicator 1 - Tooling & Reproducible Workflow Readiness

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

I needed to set up a reproducible workflow for extracting compressed images from a 16.5 GB ROS bag file (`AMtown02.bag`) and running COLMAP + OpenSplat reconstruction on WSL2, without GPU acceleration.

### What I did

1. **Environment setup:** I configured WSL2 Ubuntu 22.04 with Docker, installed COLMAP via `apt`, and built OpenSplat in a Docker container (`f50311bcc4b3`).
2. **ROS bag extraction:** I used a Docker container running ROS Noetic to extract the compressed image topic `/left_camera/image/compressed` using `cnspy-rosbag2image`:
   ```bash
   docker run -it --rm -v /mnt/c/Users/齐/Downloads:/data osrf/ros:noetic-desktop-full \
     bash -c "source /opt/ros/noetic/setup.bash && \
              apt update && apt install -y python3-pip && \
              pip install cnspy-rosbag2image && \
              cd /data && \
              rosbag2image -i AMtown02.bag -t /left_camera/image/compressed -o extracted_AMtown02"
   ```
3. **COLMAP reconstruction:** I created a dedicated workspace `colmap_test/` and ran feature extraction, exhaustive matching, and incremental reconstruction.
4. **OpenSplat training:** I copied the OpenSplat executable from Docker container to WSL (`/home/box717/opensplat`) and ran training with CPU mode and downscaling.

### Evidence

- **Environment / container / setup command:** Docker command for bag extraction (above), `colmap feature_extractor`, `colmap exhaustive_matcher`, `colmap mapper` commands.
- **Repo path / script:** `colmap_test/` folder containing `database.db`, `sparse/0/cameras.txt`, `sparse/0/images.txt`, `sparse/0/points3D.txt`.
- **Screenshot / figure:** Terminal logs showing COLMAP matching progress (22.5 minutes elapsed).
- **Commit / issue / note:** `extracted_AMtown02/` directory (7499 PNG images), `test_images/` (200 PNG images), `splat.ply` (14 MB, 165,253 Gaussians).
- **Reproducibility detail:** Another person can run the same Docker command to extract images, then follow the COLMAP → OpenSplat sequence.

### What this shows

The evidence demonstrates my ability to build a stable, containerized workflow using WSL2, Docker, ROS, COLMAP, and OpenSplat. I successfully extracted 7499 images, performed sparse reconstruction on 200 test images, and trained a 3DGS model with 165,253 Gaussians—all reproducible via documented commands.

### Limitation

The workflow remains fragile in two areas: (1) the `extracted_AMtown02/` directory is 16+ GB, making sharing impractical; (2) the OpenSplat executable was copied from a Docker container that may not be available to others without rebuilding.

### Next improvement

Package the entire pipeline into a single Docker image with all dependencies (OpenSplat, COLMAP, Python environment) and provide a `Makefile` or `run.sh` script that automates the entire reconstruction process from bag file to PLY output.

---

## 6. Indicator 2 - Prompting Strategy & AI Co-Creation

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

I encountered three major technical challenges and needed to use AI (Cursor, ChatGPT) to debug them efficiently without falling into "hallucination" traps.

### What I did

1. **Offset out of range error:** When using the `rosbags` Python library to read compressed images, I repeatedly got `offset -654376896 out of range for ...-byte buffer`. I prompted AI with the exact error message and my code context. AI initially suggested complex manual decoding, but I realized the correct fix was to switch to a Docker-based ROS Noetic environment with `cnspy-rosbag2image`. I refined my prompt to ask specifically about ROS-native tools.
2. **COLMAP initialization failure:** The mapper failed with "No good initial image pair found". I prompted AI with the error and COLMAP documentation excerpts. AI suggested `sequential_matcher` with `--SequentialMatching.loop_detection 0`, which worked immediately.
3. **OpenSplat OOM kill:** The process was terminated with "Killed". I provided AI with my system specs (AMD Ryzen 9 4900HS, 7.5 GB memory) and the training command. AI recommended adding `-d 2` for downscaling, which resolved the issue.

### Evidence

- **snippet 1:** "I'm getting `offset -654376896 out of range` when using `rosbags` to read compressed images from a ROS1 bag. What could cause this?"
- **Refined snippet:** "Given that `rosbags` seems to have a bug for this specific bag, what ROS-native tool can extract compressed images reliably?"
- **snippet 2:** "COLMAP mapper says 'No good initial image pair found' on an aerial image sequence. How can I force initialization?"
- **snippet 3:** "OpenSplat gets 'Killed' after running for a few minutes. I'm on CPU with 7.5GB memory. What parameters should I change?"

### What this shows

I learned to provide AI with specific error messages, system context, and tool documentation. When AI gave incorrect suggestions (e.g., overly complex manual decoding for the offset error), I recognized the hallucination and redirected toward more reliable solutions (Docker + ROS native tools). I also learned to use iterative refinement: start with a broad question, narrow down based on AI response, and verify against official documentation.

### Limitation

AI still occasionally produces plausible-sounding but incorrect solutions, especially for niche ROS/3DGS issues. For example, AI initially suggested using `rosbags` custom decoders, which would have wasted significant time.

### Next improvement

Maintain a personal "prompt log" repository to document which prompting strategies worked for which problem types, creating a reusable knowledge base for future debugging.

---

## 7. Indicator 3 - Verification, Documentation & Technical Judgment

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

I needed to verify AI-generated commands against official documentation and make engineering judgments about which solutions were appropriate for my resource-constrained environment.

### What I did

1. **Cross-checking AI suggestions:** When AI suggested using `rosbags` library, I checked the official OpenSplat documentation and GitHub issues, finding that many users reported `offset` errors. I decided to use Docker-based ROS Noetic instead, which was not AI's initial suggestion but was based on my verification.
2. **Documentation-driven fixes:** For the COLMAP initialization issue, I read the COLMAP documentation on `sequential_matcher` and `loop_detection` parameters before accepting AI's suggestion. I confirmed that `loop_detection 0` was appropriate for a forward-moving aerial sequence.
3. **Judgment on resource constraints:** When AI suggested running OpenSplat on full resolution (2448×2048), I recognized this would exceed my 7.5 GB memory limit based on my understanding of the memory footprint. I proactively asked about downscaling options before AI proposed them.

### Evidence

- **Documentation links checked:** OpenSplat GitHub README (confirmed `-d` parameter for downscaling), COLMAP documentation on `sequential_matcher`.
- **Verification note:** "I verified that `-d 2` reduces memory by approximately 4× (from ~8GB to ~2GB), which fits within my 7.5GB limit."
- **Rejection of weak AI output:** "AI suggested using `rosbags` manual decoder; I rejected this because the OpenSplat README explicitly recommends using `cnspy-rosbag2image` for ROS bag extraction."

### What this shows

The evidence demonstrates my ability to verify AI-generated output against authoritative sources (documentation, GitHub issues) and make context-aware engineering judgments. I did not blindly trust AI; instead, I used it as a co-pilot while maintaining final responsibility for technical decisions.

### Limitation

My verification process was mostly manual and ad hoc. I did not maintain a systematic log of which AI suggestions were verified or rejected, making it harder to learn from past mistakes.

### Next improvement

Create a structured verification checklist: (1) check official documentation, (2) search GitHub issues for similar errors, (3) test on a minimal example, (4) document findings in a shared team log.

---

## 8. Indicator 4 - Debugging, Iteration & Problem Solving

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

I faced three major debugging challenges: (1) `rosbags` library offset error preventing image extraction, (2) COLMAP initialization failure, (3) OpenSplat OOM kill. Each required structured troubleshooting and iterative refinement.

### What I did

**Challenge 1: offset error (iteration 1–3)**
- **Attempt 1:** Used `rosbags` library as AI suggested → failed with offset error.
- **Attempt 2:** Upgraded `rosbags` to latest version → same error.
- **Attempt 3:** Switched to Docker + ROS Noetic + `cnspy-rosbag2image` → **success**.

**Challenge 2: COLMAP initialization (iteration 1–2)**
- **Attempt 1:** Used default `exhaustive_matcher` and `mapper` → "No good initial image pair found".
- **Attempt 2:** Switched to `sequential_matcher` with `--SequentialMatching.loop_detection 0` → **success**.

**Challenge 3: OpenSplat OOM (iteration 1–2)**
- **Attempt 1:** Ran training on full-resolution 2448×2048 images → process killed.
- **Attempt 2:** Added `-d 2` downscaling and reduced iterations to 2000 → **success**.

### Evidence

- **Logs / screenshots:** Terminal output showing offset error, COLMAP error message, "Killed" message, and successful training logs (loss decreasing from 0.0476 to 0.0233).
- **Commit diffs:** Changes to OpenSplat command from `./opensplat ./sparse/0 -n 2000` to `./opensplat ./sparse/0 -n 2000 -d 2 --cpu`.
- **Before/after metrics:** Training loss dropped from ~0.048 (Step 360) to 0.0233 (Step 2000), memory usage reduced by ~4×.

### What this shows

The debugging iterations demonstrate structured troubleshooting: read error logs → hypothesize cause → test one change at a time → verify outcome. I learned that regression is inevitable (problems often require multiple attempts) and that switching to a completely different approach (Docker instead of Python library) is sometimes the most efficient path.

### Limitation

My debugging was somewhat reactive rather than proactive. I could have tested the OpenSplat command on a smaller subset of images (e.g., 50 images) first to catch the OOM issue earlier.

### Next improvement

Implement a "progressive testing" strategy: start with minimal resource settings (e.g., 50 images, `-d 4`), verify success, then incrementally increase complexity. This would catch resource issues earlier in the pipeline.

---

## 9. Indicator 5 - Integration & Benchmark-Driven Improvement

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

My 3DGS reconstruction module needed to integrate with the team's VO/VSLAM (ORB-SLAM3) and segmentation (U-Net) modules, and provide measurable outputs for benchmarking.

### What I did

1. **Integration with upstream:** I ensured my OpenSplat pipeline could accept camera poses from ORB-SLAM3 (COLMAP format) and produce PLY outputs that could be loaded into the team's evaluation framework.
2. **Benchmark metrics:** I recorded key performance indicators: initial COLMAP points (60,941), final Gaussian count (165,253), model file size (14 MB), training loss reduction (51%, from ~0.048 to 0.0233), and training time (~10 minutes for 1640 steps).
3. **Parameter tuning for quality improvement:** I experimented with downscaling factors (`-d 2`), iteration counts (2000 steps), and matching strategies (exhaustive vs. sequential) to balance reconstruction quality and resource constraints.

### Evidence

- **Pipeline figure / description:** The pipeline is documented as: ROS bag → image extraction → COLMAP sparse reconstruction → OpenSplat training → PLY model.
- **Benchmark table:** See Summary Matrix for loss metrics and Gaussian count.
- **Qualitative comparison:** Full-resolution training was impossible due to memory limits; downscaled (`-d 2`) training succeeded but lost some high-frequency details, as expected given the trade-off.

### What this shows

The evidence demonstrates my ability to connect my module to the larger pipeline, understand how one module's parameters affect downstream performance, and use benchmarking metrics to guide improvement decisions. I learned that moderate downsampling (1/2 resolution) can substantially outperform full-resolution training when memory is constrained, consistent with recent research findings.

### Limitation

My benchmarking was limited to training metrics (loss, Gaussian count, file size). I did not evaluate the model's rendering quality using reference-based metrics (PSNR, SSIM) because ground truth images were not available for the AMtown02 scene.

### Next improvement

Integrate no-reference quality metrics (e.g., NOVA-3DGS, CLIPGaussian) to evaluate reconstruction quality without requiring ground truth, enabling quantitative comparison across different parameter settings.

---

## 10. Indicator 6 - Responsible AI Use, Reflection & Redesign

**Before-course confidence (optional):** 2/5  
**End-of-course confidence:** 4/5

### Situation / task

Throughout the project, I used AI tools (Cursor, ChatGPT) extensively. I needed to reflect on when AI was helpful, when it hallucinated, and how I could redesign my approach for better outcomes.

### What I did

1. **Hallucination awareness:** AI initially suggested complex custom decoders for the `rosbags` offset error. I recognized this as a hallucination because it contradicted the OpenSplat documentation, which recommended using ROS-native tools. I corrected the path by switching to Docker + `cnspy-rosbag2image`.
2. **Human oversight:** I always verified AI-generated commands against official documentation (OpenSplat README, COLMAP docs) before executing them. For example, I confirmed the `-d 2` downscaling parameter was documented before using it.
3. **Honest limitation reporting:** This report explicitly documents what worked (extraction, COLMAP reconstruction, OpenSplat training) and what failed (full-resolution training impossible, GPU not available, ground truth missing for PSNR/SSIM).
4. **Redesign priorities:** Based on this reflection, I identified three redesign priorities: (1) package the entire pipeline into a single Docker image, (2) implement progressive testing to catch resource issues earlier, (3) integrate no-reference quality metrics.

### Evidence

- **Corrected AI output:** The original AI suggestion for the offset error was to use `rosbags` manual decoding; I corrected this by using Docker + `cnspy-rosbag2image`.
- **Authenticity statement:** This report is my own work; AI was used as a co-pilot for debugging and documentation, but all engineering decisions and final content are mine.
- **Redesign plan:** See "Next improvement" sections in Indicators 1–5 for concrete redesign steps.

### What this shows

The evidence demonstrates my ability to use AI responsibly: recognizing hallucinations, verifying against authoritative sources, maintaining human oversight, and honestly reporting limitations. I learned that AI is most effective as a collaborative tool when I understand the underlying engineering principles well enough to evaluate its suggestions.

### Limitation

Despite my efforts, I occasionally accepted AI suggestions without sufficient verification, leading to wasted time (e.g., the first attempt at `rosbags` manual decoding). My verification process was not systematic.

### Next improvement

Implement a mandatory "verification pause" before executing any AI-suggested command that modifies the system or consumes significant resources: (1) check official docs, (2) search for similar issues, (3) test on minimal example, (4) document the verification.

---

## 11. Overall Growth Summary

Over the course of this project, I have grown significantly as an AI-assisted engineering learner:

1. **Toolchain mastery:** I can now confidently navigate WSL2, Docker, ROS, COLMAP, and OpenSplat, and I understand how to containerize workflows for reproducibility.
2. **AI co-creation skills:** I have learned to provide AI with context-rich prompts, recognize hallucinations, and iteratively refine solutions based on verification.
3. **Debugging discipline:** I now follow a structured debugging approach: read logs → hypothesize → test one change → document outcomes.
4. **Integration thinking:** I understand how my 3DGS module fits into the larger perception pipeline and how parameter choices affect downstream benchmarking.
5. **Responsible AI use:** I no longer trust AI blindly; I always verify against documentation and maintain human oversight for critical decisions.

The evidence in this report—extracted images, COLMAP reconstruction, OpenSplat training logs, loss metrics, and the final PLY model—demonstrates my technical growth and my ability to use AI responsibly in engineering problem-solving.

---

## 12. GenAI Use Declaration

I declare that AI tools (Cursor, ChatGPT-4, GitHub Copilot) were used in the following ways during this project:

- **Code generation:** AI helped generate initial command structures for Docker, COLMAP, and OpenSplat.
- **Debugging assistance:** AI helped interpret error messages (offset error, COLMAP initialization failure, OOM kill) and suggest solutions.
- **Documentation:** AI assisted in structuring this report and generating initial drafts of indicator sections.

**My role:** All engineering decisions, verification against documentation, final command execution, and data interpretation were performed by me. AI was used as a co-pilot, not as a substitute for understanding.

---

## 13. References

1. Kerbl, B., et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering." SIGGRAPH 2023.
2. Schönberger, J. L., & Frahm, J. M. "Structure-from-Motion Revisited." CVPR 2016.
3. OpenSplat GitHub Repository. https://github.com/pierotofy/OpenSplat
4. MARS-LVIG dataset. https://github.com/hkust-aerial-robotics/MARS-LVIG
5. UAVScenes dataset. https://github.com/sijieaaa/UAVScenes
6. NOVA-3DGS: No-reference Objective VAlidation for 3D Gaussian Splatting. 2025.

---

## 14. Appendix (optional)

**Key paths used in this project (for reproducibility):**

- Bag file (original): `C:\Users\齐\Downloads\AMtown02.bag` (deleted due to space)
- Extracted images: `C:\Users\齐\Downloads\extracted_AMtown02` (7499 PNG)
- Test images: `C:\Users\齐\Downloads\test_images` (200 PNG)
- COLMAP workspace: `/mnt/c/Users/齐/Downloads/colmap_test/`
- OpenSplat executable: `/home/box717/opensplat` (copied from Docker container `f50311bcc4b3`)
- Final model: `C:\Users\齐\Desktop\splat.ply` (14 MB, 165,253 Gaussians)

**Hardware specification:**
- CPU: AMD Ryzen 9 4900HS with Radeon Graphics (8 cores, 16 threads)
- Memory: 7.5 GiB (WSL2 allocation)

---
