# Prime Authorizer - an Example of Building an Envoy WASM Filter with Rust

Based on the [proxy-wasm crate](https://crates.io/crates/proxy-wasm/0.1.0)

The filter is ready to be built and verified with the enclosed docker-compose environment.

## Building and running:

1. clone this repo
2. `rustup target add wasm32-unknown-unknown`, if needed for cross compiling
3. `cargo build --target=wasm32-unknown-unknown --release`
4. `docker-compose up --build -d`

## What the Filter Does
Each request directed to our service needs to be authorized by sending a token which is then checked for validity by the filter. If the token is validated - the request is passed on to the service. Otherwise - 403 response is returned to the caller. 
The validity check for the token is quite dumb - we check if the token is a prime number.

## Testing it Works
```bash
curl  -H "token":"323232" 0.0.0.0:18000
Access forbidden.

curl  -H "token":"32323" 0.0.0.0:18000
"Welcome to WASM land"
```
## Cleanup
```bash
docker-compose down
```

Read this [blog post](https://antweiss.com/blog/extending-envoy-with-wasm-and-rust/) for more details on Envoy, WASM and Rust.

## Remote fetch WASM binary from Azure storage account
To prepare the proxy to run in AKS cluster using Istio, configure Envoy to fetch WASM binary from remote storage account.

Prereq:
1. Create Azure Storage Account.
1. Upload the WASM binary to blob storage.
   ```bash
   storage_account=<storage account name>
   projectRoot=$(git rev-parse --show-toplevel)
   wasm_file=myenvoyfilter.wasm
   wasm_filepath=${projectRoot}/target/wasm32-unknown-unknown/release/${wasm_file}
   az storage blob upload --file ${wasm_filepath} --account-name ${storage_account} --container-name wasm --name  ${wasm_file} --auth-mode login --overwrite
   ```
1. Prepare envoy config.
   ```bash
   storage_account=<storage account name>
   projectRoot=$(git rev-parse --show-toplevel)
   wasm_file=myenvoyfilter.wasm
   wasm_filepath=${projectRoot}/target/wasm32-unknown-unknown/release/${wasm_file}
   wasm_file_sha256=$(sha256sum ${wasm_filepath}| cut -d ' ' -f 1)
   expiry=$(date -d "+7 days" +"%Y-%m-%dT%H:%M:%SZ")
   wasm_file_uri_sas=$(az storage blob generate-sas --account-name ${storage_account} --container-name wasm  --name ${wasm_file} --permissions r --expiry $expiry --auth-mode login --as-user --full-uri --output tsv)
   (export wasm_file_uri_sas; export wasm_file_sha256; export storage_account;  envsubst < envoy/envoy-remote.yaml.template > envoy/envoy-remote.yaml)
   ```
1. Run
   ```bash
   docker compose -f docker-compose-remote.yaml up --build -d
   ```
## Testing it Works
```bash
curl  -H "token":"323232" 0.0.0.0:18000
Access forbidden.

curl  -H "token":"32323" 0.0.0.0:18000
"Welcome to WASM land"
```
## Cleanup
```bash
docker-compose -f docker-compose-remote.yaml up  down
```
