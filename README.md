# Request Payments subgraph

This repo contains the code and configuration for Request Payment subgraphs:

- [Mainnet](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-mainnet)
- [Polygon (Matic)](https://thegraph.com/explorer/subgraph/requestnetwork/request-payments-matic)
- [Celo](https://thegraph.com/explorer/subgraph/requestnetwork/request-payments-celo)
- [BSC](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-bsc)
- [Gnosis Chain (xDai)](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-xdai)
- [Fuse](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-fuse)
- [Fantom](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-fantom)
- [Near](https://thegraph.com/hosted-service/subgraph/requestnetwork/request-payments-near)

It indexes Request's proxy smart-contracts for easy querying of payment data.

Smart-contract addresses can be found here:

- [ERC20 Proxy](https://github.com/RequestNetwork/requestNetwork/blob/master/packages/smart-contracts/src/lib/artifacts/ERC20Proxy/index.ts)
- [ERC20 Fee Proxy](https://github.com/RequestNetwork/requestNetwork/blob/master/packages/smart-contracts/src/lib/artifacts/ERC20FeeProxy/index.ts)
- [ERC20 Conversion Proxy](https://github.com/RequestNetwork/requestNetwork/blob/master/packages/smart-contracts/src/lib/artifacts/Erc20ConversionProxy/index.ts)
- [requestnetwork.near](https://github.com/RequestNetwork/requestNetwork/blob/master/packages/payment-detection/src/near-detector.ts)

[Learn more about TheGraph](https://thegraph.com/)

## Contributing

```
# setup variables
cp .env.sample .env # don't forget to edit the WEB3_URL and, if required, the NETWORK variable

# install dependencies
yarn
# generate ABIs & subgraph manifests, and generate types
yarn prepare

# Run a local Graph Node
docker-compose up -d
# Run
```

### Adding a new chain

> This requires the `@requestnetwork/smart-contracts` package to be deployed.

```
export NETWORK=my-network

# update to latest version
yarn add --exact @requestnetwork/smart-contracts@next

# add new network
cat <<< $(jq '. + [env.NETWORK] | unique' cli/networks.json) > cli/networks.json

# update CI (update deployment targets with cli/networks.json)
NETWORKS=$(cat ./cli/networks.json) yq e -i '.jobs.deploy.strategy.matrix.chain |= env(NETWORKS)' .github/workflows/deploy.yaml

# create Github Environment (for CI) based on mainnet
yarn subgraph configure-ci $NETWORK
```

## Manifests

The subgraphs manifests are automatically generated using the [prepare script](./scripts/prepare.ts), which uses `@requestnetwork/smart-contracts` NPM package to get the smart-contracts addresses.

One manifest can refer to many different versions of proxies dealing with the same payment network. The first version found is not explicitely mentionned in generated files and data sources naming. Example; `EthProxy` implicitely refers to the version `0.1.0`. Further versions are referenced in this format: `EthProxy_0_2_0` for the contract `EthProxy` of abi version `0.2.0`.

> Note: The `TransferWithReferenceAndFee` event is configured twice. That is because the Conversion proxy makes an internal call to the ERC20 Fee proxy. Both `TransferWithReferenceAndFee` and `TransferWithConversionAndReference` need to be parsed for the Conversion smart-contract.

### Build

```
export NETWORK=goerli
yarn build
```

## Deployment

### Local

```
export NETWORK=goerli
yarn create-local
yarn deploy-local
```

### Networks

The live deployment is automated for EVM chains on the hosted service.
For test chains (goerli), it will be automatically deployed when pushed to `main`

For production chains (all others), it is semi automatic, and requires a manual approval in [github actions](https://github.com/RequestNetwork/payments-subgraph/actions).

For non-EVM deployments, use:

```
yarn graph deploy --product hosted-service --deploy-key <GRAPH_KEY> requestnetwork/request-payments-<network> ./subgraph.<network>.yaml
```

For decentralized network, use:
```
yarn graph deploy --studio request-payments-<network> ./subgraph.<network>.yaml --version-label v1.<bumped-version>
```

### Check the deployed version

You can compare the code to the deployed version using one of these commands

```
# all
yarn subgraph compare
# one network
yarn subgraph compare NETWORK_NAME
# several networks
yarn subgraph compare NETWORK_NAME_1 NETWORK_NAME_2
```

## Example query

```graphql
{
  payments {
    txHash
    gasPrice
    contractAddress
    block
    amount
    amountInCrypto
    feeAmount
    feeAmountInCrypto
    feeAddress
    tokenAddress
    maxRateTimespan
    reference
    id
    to
    from
    currency
  }
}
```

## Troubleshooting

### Delays

Run of these commands to check for indexing delays.

```
# all
yarn subgraph monitor
# one network
yarn subgraph monitor NETWORK_NAME
# several networks
yarn subgraph monitor NETWORK_NAME_1 NETWORK_NAME_2
```

### Hosting service API

URL: https://api.thegraph.com/index-node/graphql
Schema: https://github.com/graphprotocol/graph-node/blob/master/server/index-node/src/schema.graphql

### Sync failed with no logs

```
http POST 'https://api.thegraph.com/index-node/graphql'  query="{ indexingStatusForPendingVersion(subgraphName: \"requestnetwork/request-payments-goerli\") { subgraph fatalError { message } nonFatalErrors {message } } }" | jq .data
```

### Build issue `TS6054: File '~lib/allocator/arena.ts' not found.`

You probably have an issue in the package resolution of `assemblyscript`.

See next issue for resolution.

### Install issue `Couldn't find match for ...`

This is related to the fact TheGraph uses a very old version of assemblyscript (see [This PR](https://github.com/graphprotocol/graph-ts/pull/185/files) for migration to the latest version).

In the meantime, `yarn cache clean` should resolve it.
