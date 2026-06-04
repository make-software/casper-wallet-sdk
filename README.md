# Casper Wallet SDK

SDK to simplify integration of [Casper Wallet](https://github.com/make-software/casper-wallet) with your amazing web app!

**Please note that the recommended way of integrating Casper Wallet into your app is now through [CSPR.click](https://CSPR.click), which provides a combined integration of major wallets in the Casper ecosystem, all at once, without the burden of maintaining multiple integrations at the same time. Please head over to [the CSPR.click documentation](https://docs.cspr.click) to start.**

## Installation

> In the near future we're planning to publish an SDK library as a npm package, so you can use it as a dependency in your JS/TS project, offering all the supplementary TypeScript types and other helpful utilities to further streamline maintenance of the wallet integration when we add new features.

For now, the SDK is injected into the global scope of your website window by the Casper Wallet extension content script, and you can access provider class and event types as below:

```ts
const CasperWalletProvider = window.CasperWalletProvider;
const CasperWalletEventTypes = window.CasperWalletEventTypes;
```

## CasperWalletProvider

The `CasperWalletProvider` class serves as the main interface for interacting with the Casper Wallet extension. It provides a collection of methods that enable developers to easily request connections, switch accounts, sign transactions, sign messages, and manage wallet events. By using this class, developers can seamlessly integrate the Casper Wallet into their web applications, ensuring a smooth user experience.

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

#### Request the sign transaction interface with the Casper Wallet extension

```ts
sign(transactionJson: string, signingPublicKeyHex: string): Promise<SignatureResponse>
```

- `transactionJson` - stringified json of a transaction (use `toJSON` method of `Transaction` instance from `casper-js-sdk` and `JSON.stringify`)

- `signingPublicKeyHex` - public key hash (in hex format)

- returns a payload response when user responded to transaction request, it will contain `signature` if approved, or `cancelled === true` flag when rejected.

Example:

```ts
const transaction = Transaction.fromTransactionV1(...);
const transactionJson = transaction.toJSON();

provider
  .sign(JSON.stringify(transactionJson), accountPublicKey)
  .then(res => {
    if (res.cancelled) {
      alert('Sign cancelled');
    } else {
      const signedTransaction = transaction.setSignature(
        res.signature,
        CLPublicKey.fromHex(accountPublicKey)
      );
      alert('Sign successful: ' + JSON.stringify(signedTransaction, null, 2));
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

#### Request the EIP-712 typed data signing interface with the Casper Wallet extension

```ts
signTypedData(params: SignTypedDataParams, signingPublicKey: string): Promise<SignTypedDataResult | undefined>
```

- `params` - the EIP-712 typed data to sign (`domain`, `types`, `primaryType`, `message`) plus optional `options` (see [`SignTypedDataParams`](#signtypeddataparams))

- `signingPublicKey` - public key to sign with (in hex format)

- returns a [`SignTypedDataResult`](#signtypeddataresult). On success it contains the `signature`, the signed `digest` and the `publicKey`; if the user rejects, `cancelled === true`; on failure, `error` and `errorCode` are set. May resolve to `undefined` if the request could not be delivered.

- requires the active account to support `sign-typed-data-eip712` (check via [`getActivePublicKeySupports`](#get-a-list-of-features-that-the-active-public-key-supports) / [`CasperWalletSupports`](#types)).

Example:

```ts
const typedData = {
  domain: {
    name: 'MyDapp',
    version: '1',
    chain_name: 'casper',
    contract_package_hash: '0x...'
  },
  types: {
    Permit: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' }
    ]
  },
  primaryType: 'Permit',
  message: {
    owner: '0x...',
    spender: '0x...',
    value: '1000'
  }
};

provider
  .signTypedData({ typedData }, accountPublicKey)
  .then(res => {
    if (!res || res.cancelled) {
      alert('Sign cancelled');
    } else if (res.error) {
      alert(`Sign failed: ${res.error} (${res.errorCode})`);
    } else {
      alert('Sign successful: ' + res.signature);
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

#### Get a list of features that the active public key supports.

It can be `CasperWalletSupports` (`sign-deploy`, `sign-transactionv1`, `signMessage` and `signTypedDataEIP712`)

```ts
getActivePublicKeySupports(): Promise<string[]>
```

- returns array of features that supports the active public key.
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
  /** if unlocked and connected, contain a list of supported features for the current active key otherwise `undefined` */
  activeKeySupports: CasperWalletSupports[] | undefined;
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

The active key was changed using the Wallet interface:

- ActiveKeySupportsChanged: casper-wallet:activeKeySupportsChanged

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

```ts
enum CasperWalletSupports {
  signDeploy = 'sign-deploy',
  signTransactionV1 = 'sign-transactionv1',
  signMessage = 'sign-message',
  signTypedDataEIP712 = 'sign-typed-data-eip712'
}
```

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

### SignTypedDataParams

Parameters for EIP-712 typed-data signing.

```ts
type SignTypedDataParams = {
  typedData: {
    domain: Record<string, unknown>;
    types: Record<string, Array<{ name: string; type: string }>>;
    primaryType: string;
    message: Record<string, unknown>;
  };
  options?: {
    /**
     * Domain schema for hashing.
     * Optional. The wallet uses typedData.types.EIP712Domain when present.
     */
    domainTypes?: Array<{ name: string; type: string }>;
    /** If true, include digest/domainSeparator/structHash in response */
    returnHashArtifacts?: boolean;
    /** If true, wallet MUST reject if typedData contains unknown/extra message fields */
    rejectUnknownFields?: boolean;
  };
};
```

### SignTypedDataResult

Result of an EIP-712 typed-data signing operation.

```ts
type EIP712HashArtifacts = {
  /** Domain type string (only if returnHashArtifacts was true) */
  domainTypeString?: string;
  /** Domain object (only if returnHashArtifacts was true) */
  domain?: Record<string, unknown>;
  /** 0x-prefixed domain separator hash (only if returnHashArtifacts was true) */
  domainSeparator?: string;
  /** Canonical type string (only if returnHashArtifacts was true) */
  canonicalTypeString?: string;
  /** 0x prefixed type hash (only if returnHashArtifacts was true) */
  typeHash?: string;
  /** 0x-prefixed struct hash (only if returnHashArtifacts was true) */
  structHash?: string;
};

type SignTypedDataResult = {
  /** Whether the signing operation was cancelled by the user */
  cancelled: boolean;
  /** Prefixed signature hex (01 for ed25519, 02 for secp256k1) + signature bytes, or null if cancelled/failed */
  signature: string | null;
  /** 0x-prefixed 32-byte hex digest that was signed, or null if cancelled/failed */
  digest: string | null;
  /** Prefixed public key used for signing, or null if cancelled/failed */
  publicKey: string | null;
  /** Error message if the operation failed, or null if successful */
  error: string | null;
  /** Machine-readable error code from SignTypedDataErrorCodes */
  errorCode?: SignTypedDataErrorCode;
  /** intermediate hash artifacts for debugging (only if returnHashArtifacts was true) */
  hashArtifacts?: EIP712HashArtifacts;
};
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

### signTypedData error codes

Unlike the other methods, `signTypedData` does not throw on a failed signing request — instead it resolves with a [`SignTypedDataResult`](#signtypeddataresult) whose `error` (human-readable message) and `errorCode` (machine-readable) fields are set. Possible codes:

```ts
const SignTypedDataErrorCodes = {
  USER_REJECTED: 'USER_REJECTED',
  INVALID_PARAMS: 'INVALID_PARAMS',
  UNSUPPORTED_TYPE: 'UNSUPPORTED_TYPE',
  DOMAIN_TYPES_REQUIRED: 'DOMAIN_TYPES_REQUIRED',
  SIGNATURE_SCHEME_NOT_SUPPORTED: 'SIGNATURE_SCHEME_NOT_SUPPORTED',
  ACCOUNT_NOT_FOUND: 'ACCOUNT_NOT_FOUND',
  NOT_AUTHORIZED: 'NOT_AUTHORIZED'
} as const;

type SignTypedDataErrorCode =
  (typeof SignTypedDataErrorCodes)[keyof typeof SignTypedDataErrorCodes];
```

```ts
provider.signTypedData({ typedData }, accountPublicKey).then(res => {
  if (res && res.error) {
    handleError(res.errorCode, res.error);
  }
});
```

## Contributing

We welcome contributions from the community to help improve the Casper Wallet SDK. If you're interested in contributing, please read our [contribution guidelines](CONTRIBUTING.md) to get started.
