# Getting Started with Cosmos Evaluator

## Prerequisites

- **Docker** — Required for the Dazel (Bazel-in-Docker) build system
  - Linux: [Docker Engine](https://docs.docker.com/engine/install/)
  - macOS: [Colima](https://github.com/abiosoft/colima) with QEMU (see [macOS Setup](customization.md#macos-setup) for more information)
- **NGC API Key** — Required to pull the prebuilt docker images for the checkers
  - Click links for [create or login to an NGC account](https://ngc.nvidia.com/signin), and [create an API key](https://org.ngc.nvidia.com/setup/api-keys).
  - Docker login to nvcr.io by following the steps found in [create an API key](https://org.ngc.nvidia.com/setup/api-keys).
- **NVIDIA GPU with CUDA** (obstacle check only) — Required for the Obstacle check's segmentation models. CPU mode is available but significantly slower.
- **Git** — To clone the repository
- **Storage Provider Credentials** — Checkers require a connection to a storage provider (S3 is recommended) to store video and data inputs. Ensure you have an S3 bucket set up and have credentials with read access to S3.
- **API Endpoints for LLM and VLM** — The VLM checker and attribute verification checker require connections to LLM and VLM endpoints. See the [Setting up your LLM and VLM API key](#setting-up-your-llm-and-vlm-api-key) section to get free, limited access to these endpoints for testing through [build.nvidia.com](https://build.nvidia.com/).

## Clone the repo
```bash
git clone https://github.com/nvidia-cosmos/cosmos-evaluator.git
cd cosmos-evaluator
```

## Repo Setup

### Minimum Requirements

| Requirement | Minimum Version | Notes |
|-------------|----------------|-------|
| **Python 3** | 3.5+ | Must be available as `python3` |
| **Docker** | 28.1.1+ | [Docker Engine](https://docs.docker.com/engine/install/) on Linux, [Colima](https://github.com/abiosoft/colima) on macOS |
| **Git** | 2.49.0+ | For cloning and LFS support |
| **NVIDIA GPU Driver** | 570.124.06+ | Required for GPU-accelerated checks (Obstacle) |
| **NVIDIA Container Toolkit** | 1.16.2+ | Required for `--gpus` flag in Docker |

GPU-related requirements are only needed for checks that use GPU inference (e.g., Obstacle). The VLM, Hallucination, and Attribute Verification checks can run without a GPU.

### Build Environment

Cosmos Evaluator uses **Dazel** (Bazel-in-Docker) for hermetic, reproducible builds. All build commands (`dazel build`, `dazel test`, `dazel run`) execute Bazel inside a Docker container, so you don't need to install Bazel locally.

To set up the build environment, source the setup script from the repo root:

```bash
. build/env_setup.sh
```

This script:
1. Validates that all required tools are installed and meet minimum version requirements
2. Registers the `dazel` command in your shell (with tab completion for bash/zsh)
3. Sets the `COSMOS_EVALUATOR_ROOT` environment variable

You need to re-source this script in each new terminal session. Verify the setup by running:

```bash
dazel info
```

For macOS-specific setup (Colima, `user.dazelrc`), see the [Customization Guide](customization.md#macos-setup).

### SegFormer Model Download

The Obstacle check requires the CitySemsegFormer ONNX model for semantic segmentation. Download it from the [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/tao/models/citysemsegformer?version=deployable_onnx_v1.0) and place it in the repo:

```bash
unzip deployable_onnx_v1.0.zip -d /tmp/citysemsegformer
cp /tmp/citysemsegformer/*.onnx checks/utils/citysemsegformer.onnx
```

Without this file, the Obstacle check's dynamic processor will raise a `FileNotFoundError` at runtime. All other checks and services are unaffected.

### Building and Testing

Build all targets in the repo:

```bash
dazel build //...
```

Run the full test suite:

```bash
dazel test //...
```

`dazel` is a drop-in replacement for `bazel` — any `bazel` command you see in documentation or tutorials works the same way by substituting `dazel`. For more information on available commands and target syntax, see the [Bazel documentation](https://bazel.build/run/build).

## Test the Checkers

This section contains the instructions to configure and run the prebuilt docker images for each of the checkers in the project.

All services provide a Swagger UI at `/docs` (e.g., `http://localhost:8000/docs`) for browsing the API documentation and testing endpoints from a browser.

### Hallucination Check

#### Setup Environment Variables

The Hallucination check requires the NVIDIA Multistorage configuration to be set. Multistorage is used to connect the checker to various different storage backends through a single, unified configuration.

To set up the environment variable, first copy the `.env.example` template from the hallucination check:

```bash
cp checks/hallucination/.env.example checks/hallucination/.env
```

then update the file with your Multistorage configuration. See the [Multistorage Configuration Example](#multistorage-configuration-example) section for an example of how this can be done for an S3 bucket, or the [Official Multistorage Documentation](https://nvidia.github.io/multi-storage-client/references/configuration.html) for the full configuration reference.

When setting the Multistorage environment variable in the ENV file, compact the JSON onto the same line to avoid parsing issues. This ENV should also be wrapped in single quotes (e.g. `'{/* your config json here */}'`).

#### Start the Service:

```bash
docker run --env-file=checks/hallucination/.env -p 8080:8080 nvcr.io/nvidia/cosmos/hallucination-checker:1.0.0
```

Verify that the service is running:

```bash
curl http://localhost:8080/health
```

#### Test the service

> Note: Before running the hallucination checker, one must upload their input to a storage provider, and add Multistorage configuration

```bash
curl -X POST "http://localhost:8080/process" \
  -H "Content-Type: application/json" \
  -d '{
    "clip_id": "my_clip",
    "original_video_path": "s3://<bucket-name>/path/to/original.mp4",
    "augmented_video_path": "s3://<bucket-name>/path/to/augmented.mp4"
  }'

```

### Obstacle Check

#### Download the SegFormer Model

> **Note:** The Obstacle check's dynamic processor requires the CitySemsegFormer ONNX model so it is not available as a prebuilt container. Download the `deployable_onnx_v1.0` zip file from the [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/tao/models/citysemsegformer?version=deployable_onnx_v1.0), extract the `.onnx` file, and save it as `checks/utils/citysemsegformer.onnx`.

See the [Obstacle Check docs](checks/obstacle.md) for more details of building and running this container.

### VLM Preset Check

#### Setup Environment Variables

The VLM check also uses the [StorageProvider framework](../services/framework/storage_providers/factory.py) to download the input video from the URL provided in the `/process/preset` request. Storage is configured through the following environment variables:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `COSMOS_EVALUATOR_STORAGE_TYPE` | No | `s3` | Storage backend: `s3` or `local` |
| `COSMOS_EVALUATOR_STORAGE_BUCKET` | Yes (S3) | — | S3 bucket name |
| `COSMOS_EVALUATOR_STORAGE_REGION` | Yes (S3) | — | AWS region (also reads `AWS_DEFAULT_REGION`) |
| `COSMOS_EVALUATOR_STORAGE_ACCESS_KEY` | No | — | AWS access key (also reads `AWS_ACCESS_KEY_ID`) |
| `COSMOS_EVALUATOR_STORAGE_SECRET_KEY` | No | — | AWS secret key (also reads `AWS_SECRET_ACCESS_KEY`) |

The storage type determines how the `augmented_video_url` in the request is resolved. With `s3`, it can be an `s3://` URI, a presigned URL, or any HTTPS URL. With `local`, it is treated as a local filesystem path.

If you already have AWS credentials in a `~/.aws/.env` file, the storage settings will also read from `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` as fallbacks if passed via `--env-file` below.

The VLM check requires access to a Vision-Language Model endpoint. To get a free API key for testing, visit any model on [build.nvidia.com](https://build.nvidia.com/), click **View Code**, then **Generate API Key**. The key will start with `nvapi-`.

Add `~/.cosmos_evaluator/.env` with your API key environment variable:

```bash
mkdir -p ~/.cosmos_evaluator
cat > ~/.cosmos_evaluator/.env << 'EOF'
BUILD_NVIDIA_API_KEY=your_api_key_here
EOF
```

See the [VLM Preset Check guide](checks/vlm-preset.md#endpoint-configuration) for details on configuring endpoints and setting up custom VLM providers.

#### Start the Service

```bash
docker run --env-file ~/.cosmos_evaluator/.env -p 8000:8000 nvcr.io/nvidia/cosmos/vlm:1.15.0
```

Verify that the service is running:

```bash
curl http://localhost:8000/health
```

#### Test the service

```bash
curl -X POST http://localhost:8000/process/preset \
  -H "Content-Type: application/json" \
  -d '{
    "video_path": "/path/to/generated_video.mp4",
    "preset_conditions": {
      "name": "environment",
      "weather": "Clear Sky",
      "time_of_day_illumination": "Morning",
      "region_geography": "Dense City Center",
      "road_surface_conditions": "Dry"
    }
  }'
```

### Attribute Verification Check

#### Setup Environment Variables

The attribute verification check NVIDIA Multistorage configuration to be set, and also requires an API key to be set for the VLM and LLM connections.

To set up the environment variable, first copy the `.env.example` template from the attribute verification check:

```bash
cp checks/attribute_verification/.env.example checks/attribute_verification/.env
```

then update the file with your Multistorage configuration and API key.

To set the Multistorage configuration, see the [Multistorage Configuration Example](#multistorage-configuration-example) section for an example of how this can be done for an S3 bucket, or the [Official Multistorage Documentation](https://nvidia.github.io/multi-storage-client/references/configuration.html) for the full configuration reference.

When setting the Multistorage environment variable in the ENV file, compact the JSON onto the same line to avoid parsing issues. This ENV should also be wrapped in single quotes (e.g. `'{/* your config json here */}'`).

To set the API key, get the API key from your VLM and LLM services. This can be an API key from build.nvidia.com models, or an API key for any models with an OpenAI API interface. The same API key will be shared between the LLM and VLM calls. If an API key is not needed, this variable can be left empty or unset.

The LLM and VLM endpoint and model can be configured in the default config for the services, or overridden in the checker /process request. See [LLM and VLM model setup](checks/attribute-verification.md#model-setup) for more information on setting the LLM and VLM endpoints and model.

#### Start the Service:

```bash
docker run --env-file=checks/attribute_verification/.env -p 8080:8080 nvcr.io/nvidia/cosmos/attribute-verification-checker:1.0.0
```

Verify that the service is running:

```bash
curl http://localhost:8080/health
```

#### Test the service

> Note: Before running the attribute verification checker, one must upload their input to a storage provider, and add Multistorage configuration.

```bash
# Verify whether the augmented video shows a scene with a sunny morning, by checking attributes for weather and time of day
curl -X POST "http://localhost:8080/process" \
  -H "Content-Type: application/json" \
  -d '{
    "clip_id": "my_clip",
    "augmented_video_path": "s3://<bucket-name>/path/to/augmented.mp4",
    "config": {
      "selected_variables": {
        "weather": "sunny",
        "time_of_day": "morning"
      },
      "variable_options": {
        "weather": [
          "sunny",
          "cloudy",
          "rainy",
          "snowy"
        ],
        "time_of_day": [
          "morning",
          "night"
        ]
      },
      "question_generation": {
          "llm": {
              "endpoint": "https://integrate.api.nvidia.com/v1",
              "model": "qwen/qwen3.5-397b-a17b"
          }
      },
      "vlm_verification": {
          "vlm": {
              "endpoint": "https://integrate.api.nvidia.com/v1",
              "model": "qwen/qwen3.5-397b-a17b"
          }
      }
    }
  }'


```

## Next Steps

- Visit the [Deployment Guide](deployment.md) for instructions on building, tagging, pushing, and running containers in production.
- Visit the [Customization Guide](customization.md) for instructions on getting started with local development.

## Appendix

### Multistorage Configuration Example

To create a multistorage configuration for an S3 bucket, you will need:
* Bucket name
* AWS access key ID
* AWS secret access key

The Multistorage configuration should be constructed like this:
```json
{
  "profiles":{
    "<bucket-name>":{
      "storage_provider":{
        "type":"s3",
        "options":{
          "base_path":"<bucket-name>",
        }
      },
      "credentials_provider":{
        "type":"S3Credentials",
        "options":{
          "access_key":"<access-key>",
          "secret_key":"<secret-key>"
        }
      }
    }
  },
  "path_mapping":{
    "s3://<bucket-name>/":"msc://<bucket-name>/"
  }
}
```

When setting the Multistorage environment variable in the ENV file, compact the JSON onto the same line to avoid parsing issues. This ENV should also be wrapped in single quotes (e.g. `'{/* your config json here */}'`).

### Setting up your LLM and VLM API key

All checkers use OpenAI-compatible endpoints for the LLM and VLM endpoints. By default, the checkers will point to [build.nvidia.com](https://build.nvidia.com/) for these endpoints. While this is a great way to get started with the checkers, it is not recommended to use the [build.nvidia.com](https://build.nvidia.com/) endpoints for production use. For production, please either download the LLM and VLM NIMs and run them locally, or connect to a stable external endpoint of your choice.

To get an API key that can connect to [build.nvidia.com](https://build.nvidia.com/), you may visit any model on the [Build Website](https://build.nvidia.com/qwen/qwen3.5-397b-a17b), then click `View Code`, then click `Generate API Key`. Take the API key (commonly starts with `nvapi-`), and use it as your environment variable for API key (e.g. `BUILD_NVIDIA_API_KEY` for the attribute verification checker).
