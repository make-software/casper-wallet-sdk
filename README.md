# casper-wallet-sdk

SDK to simplify integration of Casper Wallet with your amazing web app!

## Installation

In the future this library will be available as npm package, for now it's injected as an global object to the global scope of your webapp.

```ts
const CasperWalletProvider = window.CasperWalletProvider;
const CasperWalletEventTypes = window.CasperWalletEventTypes;
```

## CasperWalletProvider

### Usage

```ts
const CasperWalletProvider = window.CasperWalletProvider;

const provider = new CasperWalletProvider();
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

#### Request the connect account interface with the Casper Wallet extension

```ts
requestConnection(): Promise<boolean>
```

- returns `true` value when connection request is accepted by the user, `false` otherwise.

#### Disconnect the Casper Wallet extension

```ts
disconnectFromSite(): Promise<boolean>
```

- returns `true` value when successfully disconnected, `false` otherwise.

#### Request the switch account interface with the Casper Wallet extension

```ts
requestSwitchAccount(): Promise<boolean>
```

- returns `true` value when successfully switched account, `false` otherwise.

#### Get the connection status of the Casper Wallet extension

```ts
isConnected(): Promise<boolean>
```

- returns `true` value when curently connected at least one account, `false` otherwise.

#### Get the active public key of the Casper Wallet extension

```ts
getActivePublicKey(): Promise<string | undefined>
```

- returns hex hash of the active public key.

#### Get version of the Casper Wallet extension

```ts
getVersion(): Promise<string>;
```

- returns version of the installed wallet extension.

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

## Events

Casper Wallet extension is emitting events when the user interacts with the wallet interface.
You should listen to those events to keep the UI of your application in sync with the Wallet state.

### CasperWalletState

Each event will contain a `json string` payload in the `event.detail` property. This payload contains the Casper Wallet internal state so you can keep you application UI in sync.

```ts
export type CasperWalletState = {
  /** contain wallet is locked flag */
  isLocked: boolean;
  /** contain active key is connected flag */
  isConnected: boolean;
  /** if unlocked and connected contain active key otherwise null */
  activeKey: string | null;
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

Account was connected using the wallet:

- Connected: "casper-wallet:connected"

Account was disconnected using the wallet:

- Disconnected: "casper-wallet:disconnected"

Browser tab was changed to some connected site:

- TabChanged: "casper-wallet:tabChanged"

Active key was changed using the Wallet interface:

- ActiveKeyChanged: "casper-wallet:activeKeyChanged"

Wallet was locked:

- Locked: "casper-wallet:locked"

Wallet was unlocked:

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
