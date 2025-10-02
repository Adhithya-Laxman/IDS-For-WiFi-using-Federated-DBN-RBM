## IDS for WiFi — Federated Deep Belief Network (DBN)

An intrusion detection system (IDS) for WiFi networks that combines federated learning with a Deep Belief Network (DBN). Multiple clients train locally and share model updates with a central server, which aggregates them using Federated Averaging (FedAvg).

### Key features
- **Federated learning**: clients train locally; only model updates are shared
- **DBN-based model**: RBM-stacked pretraining with FNN classifier
- **RPC server/client**: lightweight Go services to exchange model parameters
- **Visualization & metrics**: accuracy, precision/recall, F1, ROC, confusion matrix

---

## Repository layout
```
config/                      # YAML configs (placeholders)
datasets/                    # Data and processed CSVs
logs/                        # Server and client logs
output_images/               # Plots generated during training
scripts/                     # Data helpers (placeholders)
src/
  clients/
    client1/                 # Go RPC client + Python runner
    client2/, client3/       # Python clients (placeholders)
  FEDavg/                    # FedAvg aggregator + client config JSON
  models/                    # DBN/RBM implementations and training scripts
  server/                    # Go RPC server + helper scripts
  trained models/            # Local/global model checkpoints (.pth)
```

Notable files:
- `src/server/server.go`: RPC server (listens on `0.0.0.0:8080`) serving and accepting model files
- `src/clients/client1/client.go`: RPC client to download global model and run `DBN_client.py`, then upload updates
- `src/FEDavg/fedavg.py`: FedAvg aggregator working from `client_configuration.json`
- `src/models/{DBN.py,RBM.py}`: model code; `DBN_client*.py` training entry points
- `src/FEDavg/client_configuration.json`: source of truth for clients and global model path

---

## Requirements
- Python 3.10+ (3.12 tested by bytecode present)
- Go 1.20+
- Recommended: Linux environment with virtualenv/conda

Python packages commonly used by the codebase:
- `torch`, `torchvision` (optional), `tqdm`, `pandas`, `scikit-learn`, `matplotlib`, `seaborn`, `networkx`, `torchviz`

If `requirements.txt` is empty, install manually:
```bash
python -m venv .venv && source .venv/bin/activate
pip install torch tqdm pandas scikit-learn matplotlib seaborn networkx torchviz
```

---

## Data
- Place your processed CSVs under `datasets/processed/` (e.g., `Filtered_dataset.csv`, `Preprocessed_set*.csv`).
- Some scripts contain absolute paths. Update them to point to files under `datasets/processed/` on your machine.
- Example quick check helper: `datasets/processed/preprocess.py` prints unique label values from the last column.

---

## Environment variables
Set these before running server/clients to avoid path issues (note: folder name has a space):
```bash
# Run from repository root
export PARAMS_PATH="$PWD/src/trained models/GLOBAL_dbn_rbm_model.pth"
export JSON_PATH="$PWD/src/FEDavg/client_configuration.json"
export UNIQUE_ID="client_001"
export NUMBER_ITR="10000"      # number of instances trained last iteration
export NAME="$(hostname)"      # client display name
```
Helper scripts:
- `src/server/server_setup.sh` exports `PARAMS_PATH` and `JSON_PATH` to `~/.bashrc`
- `src/clients/client_setup.sh` shows an example client-side setup and cron scheduling

---

## Running the RPC server (Go)
The server expects to find `auth.json` and writes into `../FEDavg/` relative to its working directory.
```bash
# From repo root
cd src/server
go run server.go
# The server listens on 0.0.0.0:8080 and serves model at $PARAMS_PATH
```
Authentication list: `src/server/auth.json` (valid client IDs). Update it to match your clients.

---

## Running a client (Go + Python)
The Go client performs two operations:
- `dwn`: download the global model (via RPC) into `PARAMS_PATH` and run `../../models/DBN_client.py`
- `updt`: upload the locally updated model in `PARAMS_PATH` back to the server

```bash
# From repo root
cd src/clients/client1
# Set env vars as shown above, ensure dataset paths in Python scripts are valid
# 1) Download global params and trigger Python training
go run client.go -opn dwn
# 2) After training finishes, upload updated params
go run client.go -opn updt
```
RPC address: The client currently dials a placeholder host. Update both occurrences in `src/clients/client1/client.go` to your server:
- `rpc.Dial("tcp", "NotAvailable-37552.portmap.host:37552")` → e.g. `rpc.Dial("tcp", "<server-ip>:8080")`

---

## Federated averaging (FedAvg)
Aggregation is driven by `src/FEDavg/client_configuration.json`:
- `network_summary.last_iteration_summary.global_model_path`: path to the global model checkpoint
- `clients[]`: list of clients with `instances_trained_last_iteration` and their model checkpoints

You can trigger FedAvg manually:
```bash
# Ensure JSON_PATH is exported to src/FEDavg/client_configuration.json
python src/FEDavg/fedavg.py
```
Note: The server also invokes FedAvg after receiving a client update.

---

## Centralized or standalone training
Several training entry points are provided under `src/models/`. Make sure to change any absolute dataset paths to your local paths under `datasets/processed/`.
- `DBN_client.py`, `DBN_client_with_visualization.py`, `DBN_client_two_attacks.py`
- `DBN_server.py` (server-side training)

These scripts typically:
- Pretrain a DBN via stacked RBMs (see `RBM.py`)
- Build an FNN from the DBN layers (`initialize_model`)
- Train and report metrics, optionally saving a `.pth` checkpoint to `src/trained models/`

---

## Outputs
- Checkpoints: `src/trained models/*.pth`
- Metrics/plots: `output_images/*.png`
- Logs: `logs/server.log`, `logs/client*.log`

---

## Troubleshooting
- **RPC cannot connect**: Update the RPC host/port in `src/clients/client1/client.go` to your server’s IP and open port `8080`.
- **Relative paths fail**: Run the server from `src/server/`; it expects `auth.json` in CWD and writes to `../FEDavg/`.
- **Spaces in path**: The `trained models` folder contains a space. Always quote `PARAMS_PATH` as shown.
- **FedAvg script error**: If `src/FEDavg/fedavg.py` complains about function arguments, ensure `_load_json` signature matches how it is called and that `JSON_PATH` points to an existing JSON.
- **Hard-coded absolute paths**: Some model scripts include absolute paths. Replace them with repo-relative paths.

---

## Contributing
- Open an issue describing the change
- Create a feature branch and submit a PR
- Please avoid committing large binaries; link to dataset sources instead

## License
No license file detected. Add one (e.g., MIT/Apache-2.0) if you plan to share or distribute.
