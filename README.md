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

The script automatically detects existing files in GCS and performs **append-only exports**:

- **First run**: Exports all data from the database
- **Subsequent runs**:
  - Finds the newest file in GCS for each table
  - Downloads it and reads the first row to determine the checkpoint
  - Regenerates the last file completely (in case it was incomplete)
  - Exports only new data that arrived since that checkpoint

### Debugging

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

### Configuration

The [config.py](./config.py) file contains the configuration for each database table including:

- `primary_key`: The primary key column name (used for append-only ordering)
- `datatypes`: Column type mappings for proper Parquet schema generation
- `chunk_size`: Number of rows to fetch per database query
- `num_chunks_per_file`: Number of chunks to write per Parquet file

Example:

```python
{
    'name': 'verified_contracts',
    'primary_key': 'id',
    'datatypes': {
        'id': 'Int64',
        'created_at': 'datetime64[ns]',
        'updated_at': 'datetime64[ns]',
        'created_by': 'string',
        'updated_by': 'string',
        'deployment_id': 'string',
        'compilation_id': 'string',
        'creation_match': 'bool',
        'creation_values': 'json',
        'creation_transformations': 'json',
        'runtime_match': 'bool',
        'runtime_values': 'json',
        'runtime_transformations': 'json',
        'runtime_metadata_match': 'bool',
        'creation_metadata_match': 'bool'
    },
    'chunk_size': 100000,
    'num_chunks_per_file': 10
}
```

This config gives `100,000 * 10 = 1,000,000` rows per file.

The files will be named `verified_contracts_0_1000000.parquet`, `verified_contracts_1000000_2000000.parquet`, etc.

Files are stored in GCS under the `v2/{table_name}/` prefix and compressed using zstd.

## Docker

Build the image:

```
docker build --tag=kuzdogan/test-parquet-linux --platform=linux/amd64 .
```

Publish:

```
docker push kuzdogan/test-parquet-linux
```
