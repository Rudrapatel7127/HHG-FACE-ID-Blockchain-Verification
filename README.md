HH Goa 2026 — Task 3: Face Identification & Blockchain Verification
Overview

This project implements a complete, end-to-end pipeline that takes a single face photograph as input, performs a genuine (non-hardcoded) reverse-image search across the public web to locate real social media content featuring that face, and then creates a tamper-evident, blockchain-verified record of the discovered match — demonstrating the full flow from raw image to independently re-verifiable on-chain proof.

The system is built around three core capabilities, chained into a single automated 7-stage pipeline:

1. Face Identification

The input image is processed using InsightFace (the buffalo_l model, running on ONNX Runtime) to detect the face and generate a 512-dimensional facial embedding — a numerical fingerprint of the face's unique features. This embedding is what powers every downstream comparison in the pipeline; no face data is matched by raw pixels, only by this encoded representation.

2. Web / Social Media Search

The face image is sent to the Google Cloud Vision API using its WEB_DETECTION feature, which performs a real reverse-image search against Google's indexed web content. This step returns actual URLs of web pages and social media posts where the image — or visually similar images — appear online. This search is fully dynamic per run: no results are pre-selected or hardcoded, satisfying the "genuine search step" requirement of the task. A configurable search budget (default 10 requests per run, hard cap of 1000) prevents runaway API usage during testing or demoing.

Each candidate result returned by the search is then independently re-verified: the pipeline downloads the actual candidate page, extracts its image, re-runs face detection/encoding on that image, and computes a face-distance score against the original input embedding (threshold: 0.8). Only a genuinely matching face is carried forward — this guards against false positives from the web search step.

3. Blockchain Verification

Once a verified match is found, the pipeline computes two cryptographic hashes:

A content hash (SHA-256 of the matched image bytes)
A record hash (a canonical hash combining the source URL, platform, content hash, retrieval timestamp, and face-match distance)

This record is then written to the Ethereum blockchain — the Sepolia public testnet by default, with a local eth-tester simulated chain available as an offline-friendly fallback. The final pipeline stage independently recomputes both hashes from scratch and compares them against the on-chain record, proving the data has not been altered since it was originally recorded — this is the "re-verification" step that demonstrates tamper-evidence, not just storage.

Pipeline Stages
[1/7] Load input image
[2/7] Detect face
[3/7] Generate face encoding (512-d embedding)
[4/7] Perform genuine reverse-image search (Google Cloud Vision)
[5/7] Verify candidate face (re-encode + compare against threshold)
[6/7] Record fingerprint on blockchain
[7/7] Re-verify blockchain record (tamper-evidence check)
Interfaces

The pipeline is exposed two ways:

CLI (main.py) — staged console output showing each of the 7 stages as they complete, plus final match/verification results.
Streamlit GUI — a browser-based interface for interactive, screen-recordable demonstration: upload an image, click run, and watch each stage complete live, ending in a clear results panel (match found/not found, source, platform, face distance, content hash, record hash, transaction hash, verification status).
Tech Stack
Component	Technology
Face detection/recognition	InsightFace (buffalo_l), ONNX Runtime
Reverse-image/web search	Google Cloud Vision API (WEB_DETECTION)
Blockchain	Ethereum — Sepolia testnet (primary) / local eth-tester (fallback)
Blockchain interaction	web3.py
GUI	Streamlit
Language	Python
Setup / Installation
bash
git clone <your-repo-url>
cd hhgoa-task3
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # macOS/Linux
pip install -r requirements.txt

Copy .env.example to .env and fill in:

GOOGLE_CLOUD_VISION_API_KEY=your_key_here
BLOCKCHAIN_RPC_URL=https://sepolia.infura.io/v3/your-project-id
BLOCKCHAIN_PRIVATE_KEY=0xyour-funded-private-key
BLOCKCHAIN_CONTRACT_ADDRESS=0xYourVerificationRegistryAddress
BLOCKCHAIN_CHAIN_ID=11155111
LOCAL_BLOCKCHAIN_RPC_URL=http://127.0.0.1:8545

.env is git-ignored — never commit real keys.

How to Run

CLI:

bash
python main.py path/to/face_image.jpg --from-address 0xYourAddress

GUI:

bash
streamlit run app/gui.py

Then open the local URL Streamlit prints, upload an image, and click Run Pipeline.

Tests:

bash
python -m pytest -q
Which Blockchain We Used

Ethereum, via [Sepolia testnet / local eth-tester — pick whichever you actually used in your final recording]. [If local: "Sepolia support is implemented but the demo/recording uses the local eth-tester chain for reliability; see Known Limitations."]

Known Limitations
Reverse-image search results depend on Google's public web index — faces with little online presence may correctly return "no match."
Sepolia recording requires a funded testnet wallet and valid RPC endpoint; the pipeline falls back to a local simulated chain when these aren't configured.
Google Cloud Vision API usage is capped by a per-run budget (default 10 requests) to control cost/quota.
