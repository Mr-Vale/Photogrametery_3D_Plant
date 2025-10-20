# Photogrammetry 3D Plant – Turntable Pipeline (Windows + NVIDIA GPU)

This repo contains a Python script to build 3D models from multi-camera turntable image sets using COLMAP, optimized for Windows PCs with NVIDIA GPUs.

It assumes images are named using this convention:
- "camera{X} - {axis} - angle{XXX}.jpeg"
- Example: "camera2 - Y - angle045.jpeg"

Where:
- camera{X} is an integer camera index (e.g., 1, 2, 3)
- {axis} is a label for the camera’s mounting axis (e.g., X, Y, Z)
- angle{XXX} is a zero-padded integer angle in degrees (e.g., 000, 015, 045, 120)

The pipeline:
1. Parses images and validates the naming scheme.
2. Runs COLMAP Feature Extraction + Matching on GPU.
3. Runs COLMAP Mapper (Structure-from-Motion) to estimate cameras and sparse points.
4. Runs COLMAP Dense Reconstruction (PatchMatch Stereo + Stereo Fusion) to generate a dense point cloud on GPU.
5. Optionally runs Poisson Mesher and Delaunay Mesher for surface meshes.

Outputs:
- work/sparse/0: Sparse reconstruction
- work/outputs/fused.ply: Dense fused point cloud
- work/outputs/poisson_mesh.ply: Poisson mesh (optional)
- work/outputs/delaunay_mesh.ply: Delaunay mesh (optional)

## Requirements (Windows + NVIDIA GPU)

- Windows 10/11
- Python 3.9+
- NVIDIA GPU with recent driver
- COLMAP (GPU-enabled build)
  - Install: https://colmap.github.io/install.html
  - Ensure `colmap` is available on PATH
  - Verify: `colmap -h`
- CUDA runtime (COLMAP GPU build requires a compatible CUDA runtime/driver)
- Optional: `nvidia-smi` available (comes with NVIDIA driver)

Python dependencies:
```
pip install -r requirements.txt
```

## Image directory layout

Place all images under a single directory, e.g.:

```
your_images/
├── camera1 - X - angle000.jpeg
├── camera1 - X - angle015.jpeg
├── camera1 - X - angle030.jpeg
├── camera2 - Y - angle000.jpeg
├── camera2 - Y - angle015.jpeg
├── camera3 - Z - angle000.jpeg
└── ...
```

Notes:
- You can use `.jpg`, `.jpeg`, or case variants. The regex is case-insensitive.
- Angles should be zero-padded to 3 digits.

## Quick start (GPU)

From the repo root:

```
python src/photogrammetry_pipeline.py ^
  --images .\your_images ^
  --workdir .\work ^
  --matcher exhaustive ^
  --gpu-index 0 ^
  --run-meshing
```

Or use the convenience script (edit paths in the file first):

```
scripts\run_windows_gpu.bat
```

Important flags:
- `--images`: Folder with your input images
- `--workdir`: Working directory (created if not exists)
- `--matcher`: One of `exhaustive` (default) or `sequential`
  - `exhaustive` is more robust, but slower
  - `sequential` can be faster for turntable data with ordered filenames
- `--gpu-index`: GPU index or comma-separated indices (e.g., `0`, `0,1`). Default: `0`
- `--use-pairs`: Restrict matching to algorithmically generated turntable pairs (faster, robust for your rig)
- `--pairs-window`: Neighbor window for intra-camera angle pairing when `--use-pairs` is set (default: 2)
- `--run-meshing`: Also produce Poisson and Delaunay meshes
- `--cpu`: Force CPU-only mode (if you need to debug without GPU)
- `--single-camera`: Treat all images as sharing a single camera model (only if cameras are truly identical)

Suggested run for a multi-camera turntable:
```
python src/photogrammetry_pipeline.py ^
  --images .\your_images ^
  --workdir .\work ^
  --matcher sequential ^
  --use-pairs ^
  --pairs-window 2 ^
  --gpu-index 0 ^
  --run-meshing
```

## Advanced notes

- This script does not require EXIF data; COLMAP estimates intrinsics per image. Use `--single-camera` only if all cameras genuinely share intrinsics.
- The `--use-pairs` mode constructs logical pairs for a turntable:
  - Within each (axis, camera) across neighboring angles
  - Across different cameras at the same angle on the same axis
- GPU use:
  - SIFT extraction, matching, and PatchMatch stereo run on the GPU.
  - Stereo fusion and meshing are CPU-bound in COLMAP.
- You can tune SIFT and PatchMatch parameters in the script if needed.

## Troubleshooting (Windows)

- COLMAP not found: ensure it’s installed and on PATH.
- GPU not used:
  - Make sure your COLMAP build has CUDA support.
  - Verify `nvidia-smi` works and that `--gpu-index` points to a valid device.
- Reconstruction fails:
  - Try `--matcher exhaustive`
  - Increase angle coverage or texture
  - Ensure steady lighting and no motion blur
- If GPU errors occur, try lowering memory use:
  - Reduce image size before processing
  - Set `--pms-max-image-size` (see script param) to limit PatchMatch stereo memory

## License

Apache-2.0 (or your preferred license)
