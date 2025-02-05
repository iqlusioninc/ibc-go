# Migrating from v4 to v5

This document is intended to highlight significant changes which may require more information than presented in the CHANGELOG.
Any changes that must be done by a user of ibc-go should be documented here.

There are four sections based on the four potential user groups of this document:
- [Chains](#chains)
- [IBC Apps](#ibc-apps)
- [Relayers](#relayers)
- [IBC Light Clients](#relayers)

**Note:** ibc-go supports golang semantic versioning and therefore all imports must be updated to bump the version number on major releases.
```go
github.com/cosmos/ibc-go/v4 -> github.com/cosmos/ibc-go/v5
```

## Chains

### Ante decorator

The `AnteDecorator` type in `core/ante` has been renamed to `RedundantRelayDecorator` (and the corresponding constructor function to `NewRedundantRelayDecorator`). Therefore in the function that creates the instance of the `sdk.AnteHandler` type (e.g. `NewAnteHandler`) the change would be like this:

```diff
func NewAnteHandler(options HandlerOptions) (sdk.AnteHandler, error) {
   // parameter validation

   anteDecorators := []sdk.AnteDecorator{
      // other ante decorators
-     ibcante.NewAnteDecorator(opts.IBCkeeper),
+     ibcante.NewRedundantRelayDecorator(options.IBCKeeper),
   }

   return sdk.ChainAnteDecorators(anteDecorators...), nil
}
```

The `AnteDecorator` was actually renamed twice, but in [this PR](https://github.com/cosmos/ibc-go/pull/1820) you can see the changes made for the final rename.

## IBC Apps

### ICS27 - Interchain Accounts

An additional parameter, `ics4Wrapper` has been added to the `host` submodule `NewKeeper` function in `modules/apps/27-interchain-accounts/host/keeper`.
This allows the `host` submodule to correctly unwrap the channel version for channel reopening handshakes in the `OnChanOpenTry` callback.

```diff
func NewKeeper(
   cdc codec.BinaryCodec, 
   key storetypes.StoreKey, 
   paramSpace paramtypes.Subspace,
+  ics4Wrapper icatypes.ICS4Wrapper,
   channelKeeper icatypes.ChannelKeeper, 
   portKeeper icatypes.PortKeeper,
   accountKeeper icatypes.AccountKeeper, 
   scopedKeeper icatypes.ScopedKeeper, 
   msgRouter icatypes.MessageRouter,
) Keeper
```

### Core

The `key` parameter of the `NewKeeper` function in `modules/core/keeper` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace,
   stakingKeeper clienttypes.StakingKeeper, 
   upgradeKeeper clienttypes.UpgradeKeeper,
   scopedKeeper capabilitykeeper.ScopedKeeper,
) *Keeper
```

The `RegisterRESTRoutes` function in `modules/core` has been removed.

### ICS03 - Connection

The `key` parameter of the `NewKeeper` function in `modules/core/03-connection/keeper` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace, 
   ck types.ClientKeeper
) Keeper
```

### ICS04 - Channel 

The function `NewPacketId` in `modules/core/04-channel/types` has been renamed to `NewPacketID`:

```diff
-  func NewPacketId(
+  func NewPacketID(
  portID, 
  channelID string, 
  seq uint64
) PacketId 
```

The `key` parameter of the `NewKeeper` function in `modules/core/04-channel/keeper` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   clientKeeper types.ClientKeeper,
   connectionKeeper types.ConnectionKeeper,
   portKeeper types.PortKeeper, 
   scopedKeeper capabilitykeeper.ScopedKeeper,
) Keeper 
```

### ICS20 - Transfer

The `key` parameter of the `NewKeeper` function in `modules/apps/transfer/keeper` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey, 
   paramSpace paramtypes.Subspace,
   ics4Wrapper types.ICS4Wrapper, 
   channelKeeper types.ChannelKeeper,
   portKeeper types.PortKeeper,
   authKeeper types.AccountKeeper,
   bankKeeper types.BankKeeper,
   scopedKeeper capabilitykeeper.ScopedKeeper,
) Keeper
```

The `amount` parameter of function `GetTransferCoin` in `modules/apps/transfer/types` is now of type `math.Int` (`"cosmossdk.io/math"`):

```diff
func GetTransferCoin(
   portID, channelID, baseDenom string,
-  amount sdk.Int
+  amount math.Int
) sdk.Coin
```

The `RegisterRESTRoutes` function in `modules/apps/transfer` has been removed.

### ICS27 - Interchain Accounts

The `key` and `msgRouter` parameters of the `NewKeeper` functions in 

- `modules/apps/27-interchain-accounts/controller/keeper` 
- and `modules/apps/27-interchain-accounts/host/keeper` 

have changed type. The `key` parameter is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`), and the `msgRouter` parameter is now of type `*icatypes.MessageRouter` (where `icatypes` is an import alias for `"github.com/cosmos/ibc-go/v5/modules/apps/27-interchain-accounts/types"`):

```diff
// NewKeeper creates a new interchain accounts controller Keeper instance
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace,
   ics4Wrapper icatypes.ICS4Wrapper,
   channelKeeper icatypes.ChannelKeeper,
   portKeeper icatypes.PortKeeper,
   scopedKeeper capabilitykeeper.ScopedKeeper,
-  msgRouter *baseapp.MsgServiceRouter,
+  msgRouter *icatypes.MessageRouter,
) Keeper  
```

```diff
// NewKeeper creates a new interchain accounts host Keeper instance
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace,
   channelKeeper icatypes.ChannelKeeper,
   portKeeper icatypes.PortKeeper,
   accountKeeper icatypes.AccountKeeper,
   scopedKeeper capabilitykeeper.ScopedKeeper,
-  msgRouter *baseapp.MsgServiceRouter,
+  msgRouter *icatypes.MessageRouter,
) Keeper 
```

The new `MessageRouter` interface is defined as:

```go
type MessageRouter interface {
 	Handler(msg sdk.Msg) baseapp.MsgServiceHandler
}
```

The `RegisterRESTRoutes` function in `modules/apps/27-interchain-accounts` has been removed.

#### Cosmos SDK message handler responses in packet acknowledgement

The construction of the transaction response of a message execution on the host chain has changed. The `Data` field in the `sdk.TxMsgData` has been deprecated and since Cosmos SDK 0.46 the `MsgResponses` field contains the message handler responses packed into `Any`s.

For chains on Cosmos SDK 0.45 and below, the message response was constructed like this:

```go
txMsgData := &sdk.TxMsgData{
   Data: make([]*sdk.MsgData, len(msgs)),
}

for i, msg := range msgs {
   // message validation

   msgResponse, err := k.executeMsg(cacheCtx, msg)
   // return if err != nil

   txMsgData.Data[i] = &sdk.MsgData{
      MsgType: sdk.MsgTypeURL(msg),
      Data:    msgResponse,
   }
}

// emit events

txResponse, err := proto.Marshal(txMsgData)
// return if err != nil

return txResponse, nil
```

And for chains on Cosmos SDK 0.46 and above, it is now done like this:

```go
txMsgData := &sdk.TxMsgData{
   MsgResponses: make([]*codectypes.Any, len(msgs)),
}

for i, msg := range msgs {
   // message validation

   any, err := k.executeMsg(cacheCtx, msg)
   // return if err != nil

   txMsgData.MsgResponses[i] = any
}

// emit events

txResponse, err := proto.Marshal(txMsgData)
// return if err != nil

return txResponse, nil
```

When handling the acknowledgement in the `OnAcknowledgementPacket` callback of a custom ICA controller module, then depending on whether `txMsgData.Data` is empty or not, the logic to handle the message handler response will be different. **Only controller chains on Cosmos SDK 0.46 or above will be able to write the logic needed to handle the response from a host chain on Cosmos SDK 0.46 or above.**

```go
var ack channeltypes.Acknowledgement
if err := channeltypes.SubModuleCdc.UnmarshalJSON(acknowledgement, &ack); err != nil {
   return err
}

var txMsgData sdk.TxMsgData
if err := proto.Unmarshal(ack.GetResult(), txMsgData); err != nil {
   return err
}

switch len(txMsgData.Data) {
case 0: // for SDK 0.46 and above
   for _, msgResponse := range txMsgData.MsgResponses {
      // unmarshall msgResponse and execute logic based on the response 
   }
   return nil
default: // for SDK 0.45 and below
   for _, msgData := range txMsgData.Data {
      // unmarshall msgData and execute logic based on the response 
   }
}
```

See [ADR-03](../architecture/adr-003-ics27-acknowledgement.md/#next-major-version-format) for more information or the [corrresponding documentation about authentication modules](../apps/interchain-accounts/auth-modules.md#onacknowledgementpacket).

### ICS29 - Fee Middleware

The `key` parameter of the `NewKeeper` function in `modules/apps/29-fee` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace,
   ics4Wrapper types.ICS4Wrapper,
   channelKeeper types.ChannelKeeper,
   portKeeper types.PortKeeper,
   authKeeper types.AccountKeeper,
   bankKeeper types.BankKeeper,
) Keeper 
```

The `RegisterRESTRoutes` function in `modules/apps/29-fee` has been removed.

### IBC testing package

The `MockIBCApp` type has been renamed to `IBCApp` (and the corresponding constructor function to `NewIBCApp`). This has resulted therefore in:

- The `IBCApp` field of the `*IBCModule` in `testing/mock` to change its type as well to `*IBCApp`:

```diff
type IBCModule struct {
   appModule *AppModule
-  IBCApp    *MockIBCApp // base application of an IBC middleware stack
+  IBCApp    *IBCApp // base application of an IBC middleware stack
}
```

- The `app` parameter to `*NewIBCModule` in `testing/mock` to change its type as well to `*IBCApp`:

```diff
func NewIBCModule(
   appModule *AppModule,
-  app *MockIBCApp
+  app *IBCApp
) IBCModule
```

The `MockEmptyAcknowledgement` type has been renamed to `EmptyAcknowledgement` (and the corresponding constructor function to `NewEmptyAcknowledgement`).

The `TestingApp` interface in `testing` has gone through some modifications:

- The return type of the function `GetStakingKeeper` is not the concrete type `stakingkeeper.Keeper` anymore (where `stakingkeeper` is an import alias for `"github.com/cosmos/cosmos-sdk/x/staking/keeper"`), but it has been changed to the interface `ibctestingtypes.StakingKeeper` (where `ibctestingtypes` is an import alias for `""github.com/cosmos/ibc-go/v5/testing/types"`). See this [PR](https://github.com/cosmos/ibc-go/pull/2028) for more details. The `StakingKeeper` interface is defined as:

```go
type StakingKeeper interface {
 	GetHistoricalInfo(ctx sdk.Context, height int64) (stakingtypes.HistoricalInfo, bool)
}
```

- The return type of the function `LastCommitID` has changed to `storetypes.CommitID` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`).

See the following `git diff` for more details:

```diff
type TestingApp interface {
   abci.Application
   
   // ibc-go additions
   GetBaseApp() *baseapp.BaseApp
-  GetStakingKeeper() stakingkeeper.Keeper
+  GetStakingKeeper() ibctestingtypes.StakingKeeper
   GetIBCKeeper() *keeper.Keeper
   GetScopedIBCKeeper() capabilitykeeper.ScopedKeeper
   GetTxConfig() client.TxConfig

   // Implemented by SimApp
   AppCodec() codec.Codec
  
   // Implemented by BaseApp
-  LastCommitID() sdk.CommitID
+  LastCommitID() storetypes.CommitID
   LastBlockHeight() int64
}
```

The `powerReduction` parameter of the function `SetupWithGenesisValSet` in `testing` is now of type `math.Int` (`"cosmossdk.io/math"`):

```diff
func SetupWithGenesisValSet(
   t *testing.T,
   valSet *tmtypes.ValidatorSet,
   genAccs []authtypes.GenesisAccount,
   chainID string,
-  powerReduction sdk.Int,
+  powerReduction math.Int,
   balances ...banktypes.Balance
) TestingApp
```

The `accAmt` parameter of the functions

- `AddTestAddrsFromPubKeys` ,
- `AddTestAddrs`
- and `AddTestAddrsIncremental`

in `testing/simapp` are now of type `math.Int` (`"cosmossdk.io/math"`):

```diff
func AddTestAddrsFromPubKeys(
   app *SimApp,
   ctx sdk.Context,
   pubKeys []cryptotypes.PubKey,
-  accAmt sdk.Int,
+  accAmt math.Int
) 
func addTestAddrs(
   app *SimApp,
   ctx sdk.Context,
   accNum int,
-  accAmt sdk.Int,
+  accAmt math.Int,
   strategy GenerateAccountStrategy
) []sdk.AccAddress
func AddTestAddrsIncremental(
   app *SimApp,
   ctx sdk.Context,
   accNum int,
-  accAmt sdk.Int,
+  accAmt math.Int
) []sdk.AccAddress
```

The `RegisterRESTRoutes` function in `testing/mock` has been removed.

## Relayers

- No relevant changes were made in this release.

## IBC Light Clients

### ICS02 - Client

The `key` parameter of the `NewKeeper` function in `modules/core/02-client/keeper` is now of type `storetypes.StoreKey` (where `storetypes` is an import alias for `"github.com/cosmos/cosmos-sdk/store/types"`):

```diff
func NewKeeper(
   cdc codec.BinaryCodec,
-  key sdk.StoreKey,
+  key storetypes.StoreKey,
   paramSpace paramtypes.Subspace,
   sk types.StakingKeeper,
   uk types.UpgradeKeeper
) Keeper
```
