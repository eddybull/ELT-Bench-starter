# Environment Setup

This guide sets up everything the ELT-Bench environment needs to run locally. These instructions are specific to MacOS; please adapt according to your OS.

## Step 1: Install Requirements
- [Python](https://www.python.org/) 3.11+

- Conda (Recommended to install through [Miniforge](https://conda-forge.org/download/))
  - To install on MacOS:
    - Step 1: run `curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"`
    - Step 2: run `bash Miniforge3-$(uname)-$(uname -m).sh`
    - Step 3: run `source ~/.zshrc` to reload Terminal

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or [OrbStack](https://orbstack.dev/) on MacOS for a lightweight option)
   - If you use OrbStack and `docker` is not found:

      ```bash
      export PATH="$HOME/.orbstack/bin:$PATH"   # add to ~/.zshrc to persist
      ```

- Airbyte (***must be version 1.5.1!***)
   - > **Why Airbyte 1.5.1?** The ELT-Bench setup scripts use Airbyte 1.5.1 (the Snowflake connector manifest uses CDK 4.3.0). On Airbyte 2.x, `setup/setup_airbyte.py` fails with HTTP 401 (Basic auth is rejected) and publishing the connector from the UI fails with `No declarative manifest image version found in database`.

   - Follow these instructions to install Airbyte 1.5.1
      - 1. Download the macOS/Linux binary for **`v0.25.0`** from <https://github.com/airbytehq/abctl/releases/tag/v0.25.0> (for Apple Silicon macOS, run `curl -L -O https://github.com/airbytehq/abctl/releases/download/v0.25.0/abctl-v0.25.0-darwin-arm64.tar.gz`).
      - 2. Run the following commands: 
         ```bash
         mkdir -p ~/abctl-old && cd ~/abctl-old
         tar -xzf abctl-v0.25.0-darwin-arm64.tar.gz
         chmod +x abctl
         ./abctl version
         ```
      - 3. Create a new file named `values.yaml` and add the following into it:
         ```yaml
         minio:
            image:
               repository: airbyte/minio
         ```
      - 4. Run this (this can take up to 30 minutes; if it fails on a `minio` step, run it again):
         ```bash
         ./abctl local install --chart-version 1.5.1 --values ./values.yaml
         ```

      - 5. Verify and set your login:
         ```bash
         ./abctl local status        # App Version should be 1.5.1
         ./abctl local credentials
         ```

      - 6. Open <http://localhost:8000>, log in with the credentials from the previous command, and set your email and password when prompted.

- psql (run `brew install libpq && brew link --force libpq`)

## Step 2: Clone and create the Python environment
Clone this repo and create the Python env.

```bash
git clone --recurse-submodules https://github.com/eddybull/ELT-Bench-starter.git
cd ELT-Bench-starter

conda create -y -n elt python=3.11
conda activate elt
pip install -r setup/requirements.txt
```

## 3. Follow all instructions after and including "Setup data destination" section in <https://github.com/uiuc-kang-lab/ELT-Bench/blob/elt_bench%2B%2B/README.md>


### Verify

```bash
docker ps                                                  # elt-postgres, elt-mongodb, elt-localstack, elt-api, airbyte-abctl-control-plane
docker exec airbyte-abctl-control-plane getent hosts elt-postgres elt-mongodb elt-localstack elt-api   # should print an IP
curl -s "localhost:5005/address/alias?limit=2" | head -c 300   # should return data
```
