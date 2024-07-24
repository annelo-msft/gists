# System.ClientModel Long-Running Operations

## Requirements

### Azure requirements

- Start operation
- Get updates regarding operation progress by polling service
- Set polling interval per update via (1) retry-after header (2) custom interval (3) exponential backoff
- Get current status of operation
- Get outcome of completed operation
- Communicate to user when operation is completed
- Communicate to user when operation state is changed
- Rehydrate an operation from a separate process
- Get HTTP details from service responses

### New third-party service requirements

- Communicate to user when operation is suspended and requires action to continue
- Get updates regarding operation progress via service response stream
- Expose linked operations (e.g. Cancel) on operation type
- Protocol method return type enables adding convenience method overload without breaking changes

## System.ClientModel types

[System.ClientModel APIView](https://spa.apiview.dev/review/1b123e7a51d44ebe945f0212ee039c65?activeApiRevisionId=52c33ec0ef944f2984f69c9fa0f5af5c&diffApiRevisionId=3063cc5747204d499f9e8c212b84c0b3&diffStyle=trees)

## Third-party generated client types

[OpenAI APIView]()

## Usage samples

## Client evolution