# Migrating from ibc-go v6 to v7

This document is intended to highlight significant changes which may require more information than presented in the CHANGELOG.
Any changes that must be done by a user of ibc-go should be documented here.

There are four sections based on the four potential user groups of this document:
- Chains
- IBC Apps
- Relayers
- IBC Light Clients

**Note:** ibc-go supports golang semantic versioning and therefore all imports must be updated to bump the version number on major releases.

## Chains

Chains will perform automatic migrations to remove existing localhost clients and to migrate the solomachine to v3 of the protobuf definition. 

An optional upgrade handler has been added to prune expired tendermint consensus states. It may be used during any upgrade (from v7 onwards).
Add the following to the function call to the upgrade handler in `app/app.go`, to perform the optional state pruning.

```go
import (
    // ...
    ibctmmigrations "github.com/cosmos/ibc-go/v6/modules/light-clients/07-tendermint/migrations"
)

// ...

app.UpgradeKeeper.SetUpgradeHandler(
    upgradeName,
    func(ctx sdk.Context, _ upgradetypes.Plan, _ module.VersionMap) (module.VersionMap, error) {
        // prune expired tendermint consensus states to save storage space
        _, err := ibctmmigrations.PruneExpiredConsensusStates(ctx, app.Codec, app.IBCKeeper.ClientKeeper)
        if err != nil {
            return nil, err
        }

        return app.mm.RunMigrations(ctx, app.configurator, fromVM)
    },
)
```

Checkout the logs to see how many consensus states are pruned.

### Light client registration

Chains must explicitly register the types of any light client modules it wishes to integrate. 

#### Tendermint registration

To register the tendermint client, modify the `app.go` file to include the tendermint `AppModuleBasic`:

```diff
import (
    // ...
+   ibctm "github.com/cosmos/ibc-go/v6/modules/light-clients/07-tendermint"
)

// ...

ModuleBasics = module.NewBasicManager(
    ...
    ibc.AppModuleBasic{},
+   ibctm.AppModuleBasic{},
    ...
)
```

It may be useful to reference the [PR](https://github.com/cosmos/ibc-go/pull/2825) which added the `AppModuleBasic` for the tendermint client.

#### Solo machine registration

To register the solo machine client, modify the `app.go` file to include the solo machine `AppModuleBasic`:

```diff
import (
    // ...
+   solomachine "github.com/cosmos/ibc-go/v6/modules/light-clients/06-solomachine"
)

// ...

ModuleBasics = module.NewBasicManager(
    ...
    ibc.AppModuleBasic{},
+   solomachine.AppModuleBasic{},
    ...
)
```

It may be useful to reference the [PR](https://github.com/cosmos/ibc-go/pull/2826) which added the `AppModuleBasic` for the solo machine client.

## IBC Apps

- No relevant changes were made in this release.

## Relayers

- No relevant changes were made in this release.

## IBC Light Clients

### `ClientState` interface changes

The `VerifyUpgradeAndUpdateState` function has been modified. The client state and consensus state return values have been removed.

Light clients **must** handle all management of client and consensus states including the setting of updated client state and consensus state in the client store.

The `Initialize` method is now expected to set the initial client state, consensus state and any client-specific metadata in the provided store upon client creation. 

The `CheckHeaderAndUpdateState` method has been split into 4 new methods:

- `VerifyClientMessage` verifies a `ClientMessage`. A `ClientMessage` could be a `Header`, `Misbehaviour`, or batch update. Calls to `CheckForMisbehaviour`, `UpdateState`, and `UpdateStateOnMisbehaviour` will assume that the content of the `ClientMessage` has been verified and can be trusted. An error should be returned if the `ClientMessage` fails to verify.

- `CheckForMisbehaviour` checks for evidence of a misbehaviour in `Header` or `Misbehaviour` types.

- `UpdateStateOnMisbehaviour` performs appropriate state changes on a `ClientState` given that misbehaviour has been detected and verified.

- `UpdateState` updates and stores as necessary any associated information for an IBC client, such as the `ClientState` and corresponding `ConsensusState`. An error is returned if `ClientMessage` is of type `Misbehaviour`. Upon successful update, a list containing the updated consensus state height is returned.

The `CheckMisbehaviourAndUpdateState` function has been removed from `ClientState` interface. This functionality is now encapsulated by the usage of `VerifyClientMessage`, `CheckForMisbehaviour`, `UpdateStateOnMisbehaviour`.

The function `GetTimestampAtHeight` has been added to the `ClientState` interface. It should return the timestamp for a consensus state associated with the provided height.

Prior to ibc-go/v7 the `ClientState` interface defined a method for each data type which was being verified in the counterparty state store.
The state verification functions for all IBC data types have been consolidated into two generic methods, `VerifyMembership` and `VerifyNonMembership`.
Both are expected to be provided with a standardised key path, `exported.Path`, as defined in [ICS 24 host requirements](https://github.com/cosmos/ibc/tree/main/spec/core/ics-024-host-requirements). Membership verification requires callers to provide the marshalled value `[]byte`. Delay period values should be zero for non-packet processing verification. A zero proof height is now allowed by core IBC and may be passed into `VerifyMembership` and `VerifyNonMembership`. Light clients are responsible for returning an error if a zero proof height is invalid behaviour. 

See below for an example of how ibc-go now performs channel state verification.

```go
merklePath := commitmenttypes.NewMerklePath(host.ChannelPath(portID, channelID))
merklePath, err := commitmenttypes.ApplyPrefix(connection.GetCounterparty().GetPrefix(), merklePath)
if err != nil {
    return err
}

channelEnd, ok := channel.(channeltypes.Channel)
if !ok {
    return sdkerrors.Wrapf(sdkerrors.ErrInvalidType, "invalid channel type %T", channel)
}

bz, err := k.cdc.Marshal(&channelEnd)
if err != nil {
    return err
}

if err := clientState.VerifyMembership(
    ctx, clientStore, k.cdc, height,
    0, 0, // skip delay period checks for non-packet processing verification
    proof, merklePath, bz,
); err != nil {
    return sdkerrors.Wrapf(err, "failed channel state verification for client (%s)", clientID)
}
```

### `Header` and `Misbehaviour`

`exported.Header` and `exported.Misbehaviour` interface types have been merged and renamed to `ClientMessage` interface.

`GetHeight` function has been removed from `exported.Header` and thus is not included in the `ClientMessage` interface

### `ConsensusState`

The `GetRoot` function has been removed from consensus state interface since it was not used by core IBC.

### Client Keeper

Keeper function `CheckMisbehaviourAndUpdateState` has been removed since function `UpdateClient` can now handle updating `ClientState` on `ClientMessage` type which can be any `Misbehaviour` implementations.  

### SDK Message

`MsgSubmitMisbehaviour` is deprecated since `MsgUpdateClient` can now submit a `ClientMessage` type which can be any `Misbehaviour` implementations.

The field `header` in `MsgUpdateClient` has been renamed to `client_message`.
