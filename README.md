# Fortress Event Listener
Event listener that listens for security related predefined set of events from the Ethereum network, and broadcasts these events for use by Fortress Live Security Audit Engine and Fortress Threat Detection and Response Engine. Built on [Eventeum](https://github.com/eventeum/eventeum). Fortress creates smart contract data cache for security related events and transactions using events collected to [MongoDB](https://www.mongodb.com/) through [Kafka](https://kafka.apache.org/).

Fortress Event Listener has dependencies such as Kafka, MongoDB and Parity Dev Node. The easiest way to running locally is to use docker-compose script from the server folder. For production plase update the properties file as we write in Configuration section.
## Installation

### Prerequisites
* Java 8
* Maven
* Docker

### Install

```sh
$ cd /path/to/eventlistener/
$ mvn clean package
$ cd server
$ docker-compose -f docker-compose.yml build
$ docker-compose -f docker-compose.yml up
```
### Development

```sh
$ make changes in the repo and - mvn clean package -
$ docker build -t fortressfoundation/eventlistener:latest . (or your hub)
$ logout docker /login docker
$ docker push fortressfoundation/eventlistener:latest
```

## How To Use
Once docker container is running, eventlistener will start listening for predefined events and transactions to write them to mongodb database. Audit and tdr engines will use this data. If you want to change configuration or manually register events/transactions read next sections.

## Configuration
Nodes can be configured in the properties file.

```yaml
ethereum:
  nodes:
    - name: default
      url: http://mainnet:8545
    - name: sidechain
      url: wss://sidechain/ws
```

If an event does not specify a node, then it will be registered against the 'default' node. Other custom flags you can activate per node are:

- `maxIdleConnections`: Maximum number of connections to the node. (default: 5)
- `keepAliveDuration`: Duration of the keep alive http in milliseconds (default: 10000)
- `connectionTimeout`: Http connection timeout to the node in milliseconds (default: 5000)
- `readTimeout`: Http read timeout to the node in milliseconds (default: 60000)
- `addTransactionRevertReason`: Enables receiving the revert reason when a transaction fails.  (default: false)
- `pollInterval`: Polling interval of the rpc request to the node (default: 10000)
- `healthcheckInterval`: Polling interval of that evenreum will use to check if the node is active (default: 10000)
- `numBlocksToWait`: Blocks to wait until we decide event is confirmed (default: 1). Overrides broadcaster config
- `numBlocksToWaitBeforeInvalidating`:  Blocks to wait until we decide event is invalidated (default: 1).  Overrides broadcaster config
- `numBlocksToWaitForMissingTx`: Blocks to wait until we decide tx is missing (default: 1)  Overrides broadcaster config

This will be an example with a complex configuration:

```yaml
ethereum:
  nodes:
  - name: default
    url: http://mainnet:8545
    pollInterval: 1000
    maxIdleConnections: 10
    keepAliveDuration: 15000
    connectionTimeout: 7000
    readTimeout: 35000
    healthcheckInterval: 3000
    addTransactionRevertReason: true
    numBlocksToWait: 1
    numBlocksToWaitBeforeInvalidating: 1
    numBlocksToWaitForMissingTx: 1
  blockStrategy: POLL

```

## Registering Events

REST api can be used to register events that should be subscribed to / broadcast.

### REST


-   **URL:** `/api/rest/v1/event-filter`
-   **Method:** `POST`
-   **Headers:**

| Key | Value |
| -------- | -------- |
| content-type | application/json |

-   **URL Params:** `N/A`
-   **Body:**

```json
{
	"id": "event-identifier",
	"contractAddress": "0x1fbBeeE6eC2B7B095fE3c5A572551b1e260Af4d2",
	"eventSpecification": {
		"eventName": "TestEvent",
		"indexedParameterDefinitions": [
		  {"position": 0, "type": "UINT256"},
		  {"position": 1, "type": "ADDRESS"}],
		"nonIndexedParameterDefinitions": [
		  {"position": 2, "type": "BYTES32"},
		  {"position": 3, "type": "STRING"}] },
	"correlationIdStrategy": {
		"type": "NON_INDEXED_PARAMETER",
		"parameterIndex": 0 }
}
```
| Name | Type | Mandatory | Default | Description |
| -------- | -------- | -------- | -------- | -------- |
| id | String | no | Autogenerated | A unique identifier for the event. |
| contractAddress | String | yes |  | The address of the smart contract that the address will be emitted from. |
| eventSpecification | json | yes |  | The event specification |
| correlationIdStrategy | json | no | null | Define a correlation id for the event (only used with the Kafka broadcaster).  See the advanced section for details. |

**eventSpecification**:

| Name | Type | Mandatory | Default | Description |
| -------- | -------- | -------- | -------- | -------- |
| eventName | String | yes | | The event name within the smart contract |
| indexedParameterTypes | String array | no | null | The array of indexed parameter types for the event. |
| nonIndexedParameterTypes | String array | no | null | The array of non-indexed parameter types for the event. |

**parameterDefinition**:

| Name | Type | Mandatory | Default | Description |
| -------- | -------- | -------- | -------- | -------- |
| position | Number | yes | | The zero indexed position of the parameter within the event specification |
| type | String | yes | | The type of the event parameter. |

Currently supported parameter types: `UINT8-256`, `INT8-256`, `ADDRESS`, `BYTES1-32`, `STRING`, `BOOL`.

Dynamically sized arrays are also supported by suffixing the type with `[]`, e.g. `UINT256[]`.

**correlationIdStrategy**:

| Name | Type | Mandatory | Default | Description |
| -------- | -------- | -------- | -------- | -------- |
| type | String | yes | | The correlation id strategy type. |
| parameterIndex | Number | yes | | The parameter index to use within the correlation strategy. |

-   **Success Response:**
    -   **Code:** 200
        **Content:**

```json
{
    "id": "event-identifier"
}
```

### Hard Coded Configuration
Static events can be configured within the application.yml file of Eventeum.

```yaml
eventFilters:
  - id: RequestCreated
    contractAddress: ${CONTRACT_ADDRESS:0x4aecf261541f168bb3ca65fa8ff5012498aac3b8}
    eventSpecification:
      eventName: RequestCreated
      indexedParameterDefinitions:
        - position: 0
          type: BYTES32
        - position: 1
          type: ADDRESS
      nonIndexedParameterDefinitions:
        - position: 2
          type: BYTES32
    correlationId:
      type: NON_INDEXED_PARAMETER
      index: 0
```

## Un-Registering Events

### REST

-   **URL:** `/api/rest/v1/event-filter/{event-id}`
-   **Method:** `DELETE`
-   **Headers:**  `N/A`
-   **URL Params:** `N/A`
-   **Body:** `N/A`

-   **Success Response:**
    -   **Code:** 200
        **Content:** `N/A`

## Listing Registered Events

### REST

-   **URL:** `/api/rest/v1/event-filter`
-   **Method:** `GET`
-   **Headers:**

| Key | Value |
| -------- | -------- |
| accept | application/json |

-   **URL Params:** `N/A`

-   **Response:** List of contract event filters:
```json
[{
	"id": "event-identifier-1",
	"contractAddress": "0x1fbBeeE6eC2B7B095fE3c5A572551b1e260Af4d2",
	"eventSpecification": {
		"eventName": "TestEvent",
		"indexedParameterDefinitions": [
		  {"position": 0, "type": "UINT256"},
		  {"position": 1, "type": "ADDRESS"}],
		"nonIndexedParameterDefinitions": [
		  {"position": 2, "type": "BYTES32"},
		  {"position": 3, "type": "STRING"}] },
	"correlationIdStrategy": {
		"type": "NON_INDEXED_PARAMETER",
		"parameterIndex": 0 }
},
....
{
	"id": "event-identifier-N",
	"contractAddress": "0x1fbBeeE6eC2B7B095fE3c5A572551b1e260Af4d2",
	"eventSpecification": {
		"eventName": "TestEvent",
		"indexedParameterDefinitions": [
		  {"position": 0, "type": "UINT256"},
		  {"position": 1, "type": "ADDRESS"}],
		"nonIndexedParameterDefinitions": [
		  {"position": 2, "type": "BYTES32"},
		  {"position": 3, "type": "STRING"}] },
	"correlationIdStrategy": {
		"type": "NON_INDEXED_PARAMETER",
		"parameterIndex": 0 }
}
]
```


NOTE: This is Pre-Alpha Release. Development for stable alpha release is being done and this may work unstable at the moment.

