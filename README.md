# AAE5303 Robust Control Technology - Reflective Essay

## 1. AI Usage Experience

Throughout this project, I used AI tools (mainly ChatGPT-4 and Cursor) as my primary coding and debugging assistant. My goal was to build a complete 3D Gaussian Splatting reconstruction pipeline from an ROS bag file containing UAV aerial imagery. The pipeline involved extracting compressed images from a 16.5 GB ROS bag, running COLMAP sparse reconstruction, and training an OpenSplat model on CPU.

I started by asking AI for a high-level plan: “What tools can extract compressed image topics from a ROS1 bag file?” AI suggested the `rosbags` Python library, which I tried. When that failed with an `offset out of range` error, I pasted the exact error message and asked for alternatives. AI then proposed using a Docker container with ROS Noetic and the `cnspy-rosbag2image` tool. I followed this advice and successfully extracted 7499 images.

Later, when COLMAP failed to find an initial image pair (“No good initial image pair found”), I asked AI how to force initialization for an aerial sequence. AI recommended using `sequential_matcher` with `--SequentialMatching.loop_detection 0`. I verified this against COLMAP documentation and it worked.

When OpenSplat training was killed by the system due to memory exhaustion, I asked AI: “OpenSplat gets 'Killed' after a few minutes on CPU with 7.5 GB memory. What parameters should I change?” AI suggested adding `-d 2` to downscale images and reducing iterations to 2000. I applied these changes, and training completed successfully, producing a 165,253‑Gaussian PLY model.

In each case, I used AI as a collaborative partner: I provided error logs and context, AI proposed solutions, and I tested them. I also learned to refine my prompts when initial answers were too generic or incorrect.

## 2. Understanding AI Limitations

AI was not always correct. The first suggestion — using the `rosbags` library — turned out to be unsuitable for my specific bag file, causing persistent offset errors. AI also initially suggested writing a custom decoder to work around the offset problem, which would have been time‑consuming and unnecessary. I recognized this as a potential hallucination because the official OpenSplat documentation recommended using ROS‑native tools instead.

Another limitation: AI did not automatically consider my hardware constraints. It proposed running OpenSplat on full‑resolution images (2448×2048), which would exceed my 7.5 GB memory. I had to proactively ask about memory‑saving parameters. This taught me that AI lacks awareness of the user’s specific environment unless explicitly told.

I also noticed that AI sometimes gave overly optimistic time estimates. For example, it said COLMAP matching on 200 images would take “a few minutes”, but it actually took 22 minutes. These discrepancies reminded me to always benchmark and not trust AI’s performance predictions.

## 3. Engineering Validation

Before executing any AI‑suggested command, I performed my own validation:

- **For the Docker extraction command:** I checked that `cnspy-rosbag2image` was indeed a legitimate ROS tool by searching its documentation and relevant GitHub issues. I also inspected the bag file using `rosbag info` inside the container to confirm the topic name `/left_camera/image/compressed` was correct.
- **For COLMAP’s `sequential_matcher`:** I read the COLMAP documentation on the `--SequentialMatching.loop_detection` parameter and confirmed that setting it to 0 was appropriate for a forward‑moving aerial sequence without loop closures.
- **For OpenSplat’s `-d 2` downscaling:** I verified the parameter in the OpenSplat README and tested it first on a subset of 50 images to ensure memory usage remained within limits.

I also kept a record of all commands and their outputs in a log file. The final model (`splat.ply`) was loaded into an online viewer for visual validation, confirming that the reconstruction produced recognizable building outlines.

## 4. Problem‑Solving Process

Three major problems arose, and each required systematic debugging:

**Problem 1: `rosbags` offset error**  
- *Symptom:* `offset -654376896 out of range` when reading compressed images.  
- *Approach:* I searched online, found similar issues, and realized the library might be incompatible with my bag’s chunk structure.  
- *Solution:* Switched to Docker + ROS Noetic + `cnspy-rosbag2image`. After the first successful extraction of a few frames, I ran the full extraction overnight.  
- *Learning:* Sometimes a complete change of toolchain is more efficient than patching a failing library.

**Problem 2: COLMAP initialization failure**  
- *Symptom:* “No good initial image pair found”.  
- *Approach:* I tested different matching strategies: first exhaustive (failed), then vocabulary tree (too slow), finally sequential matching.  
- *Solution:* `colmap sequential_matcher --SequentialMatching.loop_detection 0` solved the issue.  
- *Learning:* For drone video sequences, sequential matching with loop detection disabled is much more reliable than exhaustive matching.

**Problem 3: OpenSplat “Killed” due to OOM**  
- *Symptom:* Process terminated after a few minutes.  
- *Approach:* I monitored memory usage with `htop` and saw it spike to >7 GB. I then looked for parameters that reduce memory footprint.  
- *Solution:* Added `-d 2` (half resolution) and limited iterations to 2000.  
- *Learning:* Downscaling is a practical trade‑off when GPU is unavailable; it still produced a usable model (165k Gaussians, 14 MB).

Each problem was solved through iterative hypothesis testing, not by a single AI answer. I made sure to change only one variable at a time and record outcomes.

## 5. Learning Growth

Through this project, I grew significantly in several areas:

- **Toolchain proficiency:** I learned to use WSL2, Docker, ROS Noetic, COLMAP, and OpenSplat. I can now set up a reproducible 3D reconstruction environment from scratch.
- **Debugging discipline:** I developed a routine: read error logs → search for known issues → formulate hypothesis → test one fix → document results. This saved me from random trial‑and‑error.
- **AI co‑creation skills:** I learned how to write effective prompts, recognise hallucinations, and when to override AI suggestions with my own judgment. I also learned to provide context (hardware specs, error logs, documentation links) to get better answers.
- **Integration mindset:** My work on 3DGS had to interface with the team’s ORB‑SLAM3 and U‑Net modules. I learned to design outputs (PLY files, loss logs) that other team members could easily consume for benchmarking.

A specific moment of growth was when I realised that AI’s first answer is not always the best; I started treating AI as a junior engineer who needs guidance and verification. This changed my approach from “ask and do” to “propose, verify, adapt”.

## 6. Critical Reflection

If I were to repeat this project, I would change several things:

- **Early prototyping:** I should have first tested the OpenSplat command on 50 images instead of 200. That would have caught the OOM issue earlier and saved time.
- **Better logging:** I did not save the full training loss log, which would have allowed me to generate professional loss curves. In the future, I will always run `tee` to capture terminal output.
- **GPU consideration:** Even a modest GPU would have reduced training time from ~20 minutes to <1 minute. If resources permit, I will prioritise GPU‑enabled environments.
- **Team integration earlier:** I completed the reconstruction in isolation before sharing the PLY model. Earlier integration with the team’s evaluation script would have revealed missing metrics (e.g., rendering PSNR) earlier.

Most importantly, I reflected on my reliance on AI. While AI helped me overcome many technical hurdles, it also lulled me into skipping some verification steps (e.g., checking the bag file’s ROS version before extraction). I will balance AI assistance with more systematic checks in the future.

## 7. Evidence

Key evidence of my work includes:

- **Extracted images:** `extracted_AMtown02/` (7499 PNG frames) – partial screenshot available.
- **COLMAP workspace:** `colmap_test/` containing `database.db` and `sparse/0/` (cameras.txt, images.txt, points3D.txt).
- **OpenSplat training log:** Terminal output showing loss decreasing from 0.0476 (Step 360) to 0.0233 (Step 2000).
- **Final model:** `splat.ply` (14 MB, 165,253 Gaussians) – viewable online.
- **Commands log:** All commands were saved in a text file `commands_history.txt`.

*All evidence is available in the team repository under the path `3dgs_module/colmap_test/` and in the shared drive.*
