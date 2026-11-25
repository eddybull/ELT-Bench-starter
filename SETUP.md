# Environment Setup

This guide sets up everything the ELT-Bench environment needs to run locally. These instructions are specific to MacOS; please adapt according to your OS.

## Step 1: Install Requirements
- [Python](https://www.python.org/) 3.11+

- Conda (Recommended to install through [Miniforge](https://conda-forge.org/download/))
  - To install on MacOS:
    - Run `curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"`
    - Run `bash Miniforge3-$(uname)-$(uname -m).sh`
    - Run `source ~/.zshrc` to reload Terminal

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or [OrbStack](https://orbstack.dev/) on MacOS for a lightweight option)
   - If you use OrbStack and `docker` is not found:

      ```bash
      export PATH="$HOME/.orbstack/bin:$PATH"   # add to ~/.zshrc to persist
      ```

- Airbyte (***must be version 1.5.1!***)
   - > **Why Airbyte 1.5.1?** The ELT-Bench setup scripts use Airbyte 1.5.1 (the Snowflake connector manifest uses CDK 4.3.0). On Airbyte 2.x, `setup/setup_airbyte.py` fails with HTTP 401 (Basic auth is rejected) and publishing the connector from the UI fails with `No declarative manifest image version found in database`.

   - Follow these instructions to install Airbyte 1.5.1
      - Download the binary for **`v0.25.0`** from <https://github.com/airbytehq/abctl/releases/tag/v0.25.0> by running
         ```bash
         mkdir -p ~/abctl-old && cd ~/abctl-old
         # Only for Apple Silicon
         curl -L -O https://github.com/airbytehq/abctl/releases/download/v0.25.0/abctl-v0.25.0-darwin-arm64.tar.gz
         tar -xzf abctl-v0.25.0-darwin-arm64.tar.gz --strip-components=1
         chmod +x abctl
         ./abctl version # make sure this is v0.25.0
         ```
      - Create a new file named `values.yaml` and add contents by running this:
         ```bash
         cat <<'EOF' > values.yaml
         minio:
           image:
             repository: airbyte/minio
         EOF
         ```
      - Install Airbyte 1.5.1 by running this (ensure OrbStack / Docker Desktop is running; this can take up to 30 minutes — if it fails on a minio step, run the exact same command again):
         ```bash
         ./abctl local install --chart-version 1.5.1 --values ./values.yaml
         ```
      - Verify the version and set your login:
         ```bash
         ./abctl local status        # App Version should be 1.5.1
         ./abctl local credentials
         ```
      - <http://localhost:8000> should open by itself (if not, open it manually). Type in your email and organization, and then use the password from the previous command to login.

- psql (run `brew install libpq && brew link --force libpq`)

## Step 2: Clone this repo and create the Python environment
Cd to wherever you want to clone this repo and create the Python env.

```bash
git clone --recurse-submodules https://github.com/eddybull/ELT-Bench-starter.git
cd ELT-Bench-starter

conda create -y -n elt python=3.11
conda activate elt
pip install -r ELT-Bench/setup/requirements.txt
```

## Step 3: Setup Snowflake.
  - Go to <https://signup.snowflake.com/> and sign up for AI Data Cloud for Enterprise account. Choose Standard edition and AWS if prompted.
  - Follow instructions here: <https://docs.snowflake.com/en/user-guide/ui-snowsight-worksheets-gs> to create a worksheet in Snowsight.
  - Copy all code from [ELT-Bench/setup/destination/setup.sql](ELT-Bench/setup/destination/setup.sql) into the Snowflake worksheet and click "run all".
  - Go to [ELT-Bench/setup/destination/snowflake_credential.json](ELT-Bench/setup/destination/snowflake_credential.json) and fill in your credentials.
    - For "account" field, go to the Snowsight worksheet, click on the circle with a letter inside on the bottom left corner, hover over the AWS line under Account, and click on "View Account Details". Copy the account identifier and paste into the json file (should be a dash).
    - For "user", type in your Snowflake account username
    - For "password", type in Snowflake account password
    - Leave the rest of the fields as is

## Step 4: Run elt_setup.sh
  - Replace `<your-email>` and `<password-from-abctl>` with the email and password you use to login to Airbyte and run this command from `ELT-Bench/setup/`
  
  ```bash
  AIRBYTE_USERNAME="<your-email>" AIRBYTE_PASSWORD="<password-from-abctl>" bash elt_setup.sh
  ```

## Step 5: (Optional) Add other destinations
  - To add Databricks or Redshift, see [ELT-Bench/README.md#set-up-data-destination](ELT-Bench/README.md#set-up-data-destination).

## Step 6: Download the ground-truth data from huggingface
  - First run `cd ELT-Bench` to move to the `ELT-Bench` directory
  - Follow instructions in [ELT-Bench/README.md#download-the-ground-truth](ELT-Bench/README.md#download-the-ground-truth) to download ground-truth for destinations.

## Step 7: Verify the environment is set up correctly

Run these read-only checks from `ELT-Bench-starter/`.

### Containers & Docker network
```bash
# Should show all 5 running: elt-postgres, elt-mongodb, elt-localstack, elt-api, airbyte-abctl-control-plane
docker ps --format "table {{.Names}}\t{{.Status}}"

# Airbyte must be able to resolve all 4 source containers by hostname (each should print an IP)
docker exec airbyte-abctl-control-plane getent hosts elt-postgres elt-mongodb elt-localstack elt-api
```

### Data sources loaded
```bash
# REST API (should return JSON rows)
curl -s "http://localhost:5005/address/alias?limit=2" | head -c 300; echo

# PostgreSQL (should return a positive row count)
PGPASSWORD=testelt psql -h localhost -p 5433 -U postgres -d books -c "SELECT count(*) AS cust_order_rows FROM cust_order;"

# MongoDB (should show rs state: 1, books collections, and a positive document count)
docker exec elt-mongodb mongosh --quiet --eval "
  print('rs state:', rs.status().myState);
  print('books collections:', db.getSiblingDB('books').getCollectionNames());
  print('books.book count:', db.getSiblingDB('books').book.countDocuments());
"

# LocalStack S3 (should list country.csv and shipping_method.csv)
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=us-west-2 \
  aws --endpoint-url=http://localhost:4566 s3 ls s3://books-bucket/
```

### Airbyte 1.5.1, credentials, and published `Elt Snowflake` connector
```bash
~/abctl-old/abctl local status | grep -E "Chart Version|App Version|Status"

python - <<'EOF'
import base64, json, urllib.request

creds = json.load(open("ELT-Bench/setup/airbyte/airbyte_credential.json"))
missing = [k for k in ("username", "password", "workspace_id", "api_definition_id") if not creds.get(k)]
assert not missing, f"Missing keys in airbyte_credential.json: {missing}"

auth = base64.b64encode(f"{creds['username']}:{creds['password']}".encode()).decode()
req = urllib.request.Request(
    "http://localhost:8000/api/v1/sources/list",
    json.dumps({"workspaceId": creds["workspace_id"]}).encode(),
    {"Content-Type": "application/json", "Authorization": f"Basic {auth}"},
)
sources = json.loads(urllib.request.urlopen(req, timeout=15).read())["sources"]
names = [s["name"] for s in sources]
print("Airbyte auth OK! Sources:", names)
assert "Elt Snowflake" in names, "Elt Snowflake source not found!"
EOF
```

### Generated agent inputs & Hugging Face ground truth
```bash
# Should be 100 task input bundles and 100 Snowflake ground-truth folders
echo "inputs count:        $(ls -1 ELT-Bench/inputs | wc -l)"
echo "gt_snowflake count:  $(ls -1 ELT-Bench/ground_truth/gt_snowflake | wc -l)"
ls ELT-Bench/ground_truth/gt_snowflake/books   # should list .csv files

# Confirm write_config.py filled in non-empty Snowflake & Airbyte fields
python - <<'EOF'
import yaml
cfg = yaml.safe_load(open("ELT-Bench/inputs/books/config.yaml"))
print("Snowflake account set:", bool(cfg["snowflake"]["config"].get("account")))
print("Airbyte workspace set:", bool(cfg["Airbyte"]["config"].get("workspace_id")))
print("Airbyte custom_api_definition_id set:", bool(cfg["Airbyte"]["config"].get("custom_api_definition_id")))
EOF
```

### Snowflake connection
```bash
python - <<'EOF'
import json
import snowflake.connector

with open("ELT-Bench/setup/destination/snowflake_credential.json") as f:
    creds = json.load(f)

# 1. Verify snowflake_credential.json (admin/evaluator connection)
conn = snowflake.connector.connect(login_timeout=30, **creds)
cur = conn.cursor()
cur.execute("SELECT CURRENT_ACCOUNT(), CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE()")
print("snowflake_credential.json OK:", cur.fetchone())
cur.execute("SHOW WAREHOUSES LIKE 'AIRBYTE_WAREHOUSE'")
print("AIRBYTE_WAREHOUSE exists:", len(cur.fetchall()) > 0)
cur.close()
conn.close()

# 2. Verify AIRBYTE_USER / AIRBYTE_ROLE created by setup.sql (used by Airbyte & dbt)
conn2 = snowflake.connector.connect(
    login_timeout=30,
    account=creds["account"],
    user="AIRBYTE_USER",
    password="Snowflake@123",
    role="AIRBYTE_ROLE",
    warehouse="AIRBYTE_WAREHOUSE",
)
cur2 = conn2.cursor()
cur2.execute("SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE()")
print("AIRBYTE_USER OK:", cur2.fetchone())
cur2.close()
conn2.close()
EOF
```