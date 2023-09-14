# Casper Wallet SDK

SDK to simplify integration of [Casper Wallet](https://github.com/make-software/casper-wallet) with your amazing web app!

## Installation

> In the near future we're planning to publish an SDK library as a npm package, so you can use it as a dependency in your JS/TS project, offering all the supplementary TypeScript types and other helpful utilities to further streamline maintenance of the wallet integration when we add new features.

For now, the SDK is injected into the global scope of your website window by the Casper Wallet extension content script, and you can access provider class and event types as below:

```ts
const CasperWalletProvider = window.CasperWalletProvider;
const CasperWalletEventTypes = window.CasperWalletEventTypes;
```

## CasperWalletProvider

The `CasperWalletProvider` class serves as the main interface for interacting with the Casper Wallet extension. It provides a collection of methods that enable developers to easily request connections, switch accounts, sign deploys, sign messages, and manage wallet events. By using this class, developers can seamlessly integrate the Casper Wallet into their web applications, ensuring a smooth user experience.

### Usage

NOTE: be aware that `window.CasperWalletProvider` will be available asynchronously, because extension has to load in the browser.

```ts
const CasperWalletProvider = window.CasperWalletProvider;

const provider = CasperWalletProvider();
```

### Constructor

```ts
constructor(options?: CasperWalletProviderOptions);
```

#### Constructor Options

```ts
type CasperWalletProviderOptions = {
  // timeout (in ms) for requests to the extension [DEFAULT: 30 min]
  timeout: number;
};
```

### Methods

#### Request the connect interface with the Casper Wallet extension. Will not show UI for already connected accounts and return true immediately

```ts
requestConnection(): Promise<boolean>
```

- returns `true` value when connection request is accepted by the user or when account is already connected, `false` otherwise.

- emits event of type `Connected` when successfully connected.

#### Request the switch account interface with the Casper Wallet extension

```ts
requestSwitchAccount(): Promise<boolean>
```

- returns `true` value when successfully switched account, `false` otherwise.

- emits event of type `ActiveKeyChanged` when successfully switched account.

#### Request the sign deploy interface with the Casper Wallet extension

```ts
sign(deployJson: string, signingPublicKeyHex: string): Promise<SignatureResponse>
```

- `deployJson` - stringified json of a deploy (use `DeployUtil.deployToJson` from `casper-js-sdk` and `JSON.stringify`)

- `signingPublicKeyHex` - public key hash (in hex format)

- returns a payload response when user responded to transaction request, it will contain `signature` if approved, or `cancelled === true` flag when rejected.

Example:

```ts
const deployJson = DeployUtil.deployToJson(deploy);

provider
  .sign(JSON.stringify(deployJson), accountPublicKey)
  .then(res => {
    if (res.cancelled) {
      alert('Sign cancelled');
    } else {
      const signedDeploy = DeployUtil.setSignature(
        deploy,
        res.signature,
        CLPublicKey.fromHex(accountPublicKey)
      );
      alert('Sign successful: ' + JSON.stringify(signedDeploy, null, 2));
    }
  })
  .catch(err => {
    alert('Error: ' + err);
  });
```

#### Request the interface to validate the transaction with the Casper Wallet extension

```ts
putDeploy(signedDeploy: Deploy): Promise<string>
```

- returns a Deploy's transaction hash, as a hexadecimal string

Example:

```ts
import { CLPublicKey, DeployUtil, CasperClient } from 'casper-js-sdk';
const deployJson = DeployUtil.deployToJson(deploy);

provider
  .sign(JSON.stringify(deployJson), accountPublicKey)
  .then(res => {
    if (res.cancelled) {
      alert('Sign cancelled');
    } else {
      const signedDeploy = DeployUtil.setSignature(
        deploy,
        res.signature,
        CLPublicKey.fromHex(accountPublicKey)
      );
      if(signedDeploy != null){
        const casperClient = new CasperClient(YOUR_NODE_URL);
        try {
          const res = await casperClient.putDeploy(signedDeploy);
          console.log(res);
        } catch (err) {
          console.error('Error:', err);
        }
      }
    }
  })
  .catch(err => {
    alert('Error: ' + err);
  });
```

#### Request the sign message interface with the Casper Wallet extension

```ts
signMessage(message: string, signingPublicKeyHex: string): Promise<SignatureResponse>
```

- `message` - message to sign as string

- `signingPublicKeyHex` - public key hash (in hex format)

- returns a payload response when user responded to transaction request, it will contain `signature` if approved, or `cancelled === true` flag when rejected.

Example:

```ts
provider
  .signMessage(message, accountPublicKey)
  .then(res => {
    if (res.cancelled) {
      alert('Sign cancelled');
    } else {
      alert('Sign successful: ' + JSON.stringify(res.signature, null, 2));
    }
  })
  .catch(err => {
    alert('Error: ' + err);
  });
```

#### Disconnect the Casper Wallet extension

```ts
disconnectFromSite(): Promise<boolean>
```

- returns `true` value when successfully disconnected, `false` otherwise.

- emits event of type `Disconnected` when successfully disconnected.

#### Get the connection status of the Casper Wallet extension

```ts
isConnected(): Promise<boolean>
```

- returns `true` value when currently connected at least one account, `false` otherwise.
- throws when wallet is locked (err.code: 1)

#### Get the active public key of the Casper Wallet extension

```ts
getActivePublicKey(): Promise<string>
```

- returns hex hash of the active public key.
- throws when wallet is locked (err.code: 1)
- throws when active account not approved to connect with the site (err.code: 2)

#### Get version of the Casper Wallet extension

```ts
getVersion(): Promise<string>;
```

- returns version of the installed wallet extension.

## Events

Casper Wallet extension is emitting events in the browser window of a connected site when the user interacts with the wallet extension.
You should set listeners to those events to keep the UI of your application in sync with the wallet extension state.

### CasperWalletState

Each event will contain a `json string` payload in the `event.detail` property. This payload contains the Casper Wallet extension internal state so you can keep you application UI in sync.

```ts
export type CasperWalletState = {
  /** contain wallet is locked flag */
  isLocked: boolean;
  /** if unlocked contain connected status flag of active key otherwise undefined */
  isConnected: boolean | undefined;
  /** if unlocked and connected contain active key otherwise undefined */
  activeKey: string | undefined;
};

const handleEvent = (event: { detail: string }) => {
  try {
    const state: CasperWalletState = JSON.parse(event.detail);
    console.log(state.activeKey);
  }
};
```

### CasperWalletEventType

Event types emitted by the Casper Wallet extension.
Events like (connected & disconnected & tabChanged) are relevant only to the currently active connected site that triggered the call so events will emit only to these tabs containing the same origin to not impact different opened sites.
Other events are emitted to all active tabs in all opened browser windows because they are triggered by the wallet state change and should be received by all opened sites to keep in sync.

Emitted when account was successfully connected (only to active site):

- Connected: "casper-wallet:connected"

Emitted when account was successfully disconnected (only to active site):

- Disconnected: "casper-wallet:disconnected"

Emitted when browser tab was changed to one containing a connected site (only to active site)::

- TabChanged: "casper-wallet:tabChanged"

Emitted when active key was changed in the wallet extension:

- ActiveKeyChanged: "casper-wallet:activeKeyChanged"

Emitted when the wallet extension was locked:

- Locked: "casper-wallet:locked"

Emitted when the wallet extension was unlocked:

- Unlocked: "casper-wallet:unlocked"

### Events Usage

```ts
useEffect(() => {
  const handleConnected = (event: any) => {
    try {
      const state: CasperWalletState = JSON.parse(event.detail);
      if (state.activeKey) {
        setActivePublicKey(state.activeKey);
      }
    } catch (err) {
      handleError(err);
    }
  };

  window.addEventListener(CasperWalletEventTypes.Connected, handleConnected);

  return () => {
    window.removeEventListener(
      CasperWalletEventTypes.Connected,
      handleConnected
    );
  };
}, [setActivePublicKey]);
```

## Types

Helper types for type safety and awesome developer experience.

### SignatureResponse

```ts
type SignatureResponse =
  | {
      cancelled: true; // if sign was cancelled
    }
  | {
      cancelled: false; // if sign was successfull
      signatureHex: string; // signature as hex hash
      signature: Uint8Array; // signature as byte array
    };
```

Usage:

```ts
function handleResponse(res: SignatureResponse) {
  if (res.cancelled) {
    alert('Cancelled');
  } else {
    console.log(res.signatureHex);
  }
}
```

## Error Handling

Each promise will return an error if something unexpected happens so you should always catch errors from each SDK call and handle them accordingly.

```ts
signMessage(message, accountPublicKey)
  .then((res) => { ... })
  .catch((err) => {
    handleError(err);
  });
```

## Contributing

We welcome contributions from the community to help improve the Casper Wallet SDK. If you're interested in contributing, please read our [contribution guidelines](CONTRIBUTING.md) to get started.
