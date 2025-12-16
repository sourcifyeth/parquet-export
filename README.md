# VerA DB Parquet Export Script

Python scripts and Docker container to export the Verifier Alliance PostgreSQL database in Parquet format and upload it to Google Cloud Storage.

The latest export is publicly available at [https://export.verifieralliance.org](https://export.verifieralliance.org).

## Requirements

- Python 3

## Installation

Create a virtual environment:

```
python -m venv venv
```

Activate the virtual environment:

```
source venv/bin/activate
```

Install dependencies:

```
pip install -r requirements.txt
```

## Usage

Run the script with:

```
python main.py
```

The script takes some additional env vars for debugging purposes:

- `DEBUG`: Enables debug logging, reduces chunk sizes by 100x, processes only 1 file per table, and skips GCS upload
- `DEBUG_TABLE`: The name of the table to dump solely. Skips other tables.
- `DEBUG_OFFSET`: Will add an offset to the `SELECT` queries `"SELECT * FROM {table_name} OFFSET {os.getenv('DEBUG_OFFSET')}"`

The rest of the env vars can be found in the `.env-template` file. Copy the .env-template file to `.env` and fill in the values.

### Authentication

For local development, set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to point to your service account key JSON file:

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json
```

In Cloud Run, authentication is automatic via Workload Identity.

The [config.py](./config.py) file contains the configuration for each database table about the chunk sizes and number of chunks per file, and the datatypes for each column in the table.

Example:

```js
  {
    'name': 'verified_contracts',
    'datatypes': {
        'id': 'Int64',
        'created_at': 'datetime64[ns]',
        'updated_at': 'datetime64[ns]',
        'created_by': 'string',
        'updated_by': 'string',
        'deployment_id': 'string',
        'compilation_id': 'string',
        'creation_match': 'bool',
        'creation_values': 'string',
        'creation_transformations': 'string',
        'runtime_match': 'bool',
        'runtime_values': 'string',
        'runtime_transformations': 'string'
    },
    'chunk_size': 10000,
    'num_chunks_per_file': 10
  }
```

This config gives `10,000 * 10 = 100,000` rows per file.

The files will be named `verified_contracts_0_100000_zstd.parquet` and `verified_contracts_100000_200000_zstd.parquet` etc. (`zstd` is the compression algorithm).

## Docker

Build the image:

```
docker build --tag=kuzdogan/test-parquet-linux --platform=linux/amd64 .
```

Publish:

```
docker push kuzdogan/test-parquet-linux
```
