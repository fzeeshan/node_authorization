# node_authorization
At the time of this code update, substrate tutorial on addition node Authorization pallet does not work, this repo shows how to get that code to to work on the latest version of Substrate (at this time the tag is polkadot-v1.9.0)

# original tutorial
https://docs.substrate.io/tutorials/build-a-blockchain/authorize-specific-nodes/

Compiled Successfully on the followin:
rustup 1.27.1 (54dd3d00f 2024-04-24)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.77.0-nightly (ef71f1047 2024-01-21)`

# This repo contains files that should be modified.
Here is the break down of all the code

open runtime/Cargo.toml
add in [dependencies]
```rust
pallet-node-authorization = { git = "https://github.com/paritytech/polkadot-sdk.git", tag = "polkadot-v1.9.0", default-features = false } 
```

in the following section
[features]
default = ["std"]
std = [
add the line on node-authorization
```rust
"pallet-node-authorization/std",    # added FZ
```
/runtime/src/lib.rs
add the following:

```rust
// added below for pallet_node_authorization
use frame_system::EnsureRoot;

//configure pallet_node_authorization starts

parameter_types! {
 pub const MaxWellKnownNodes: u32 = 8;
 pub const MaxPeerIdLength: u32 = 128;
}
impl pallet_node_authorization::Config for Runtime {
 type RuntimeEvent = RuntimeEvent;
 type MaxWellKnownNodes = MaxWellKnownNodes;
 type MaxPeerIdLength = MaxPeerIdLength;
 type AddOrigin = EnsureRoot<AccountId>;
 type RemoveOrigin = EnsureRoot<AccountId>;
 type SwapOrigin = EnsureRoot<AccountId>;
 type ResetOrigin = EnsureRoot<AccountId>;
 type WeightInfo = ();
}
//configure pallet_node_authorization ends
```
```rust
//update the runtime section add according to your runtime, here is an example:
#[frame_support::runtime]
mod runtime {
	#[runtime::runtime]
	#[runtime::derive(
		RuntimeCall,
		RuntimeEvent,
		RuntimeError,
		RuntimeOrigin,
		RuntimeFreezeReason,
		RuntimeHoldReason,
		RuntimeSlashReason,
		RuntimeLockId,
		RuntimeTask
	)]
	pub struct Runtime;

	#[runtime::pallet_index(0)]
	pub type System = frame_system;

	#[runtime::pallet_index(1)]
	pub type Timestamp = pallet_timestamp;

	#[runtime::pallet_index(2)]
	pub type Aura = pallet_aura;

	#[runtime::pallet_index(3)]
	pub type Grandpa = pallet_grandpa;

	#[runtime::pallet_index(4)]
	pub type Balances = pallet_balances;

	#[runtime::pallet_index(5)]
	pub type TransactionPayment = pallet_transaction_payment;

	#[runtime::pallet_index(6)]
	pub type Sudo = pallet_sudo;

	// Include the custom logic from the pallet-template in the runtime.
	#[runtime::pallet_index(7)]
	pub type TemplateModule = pallet_template;

	// pallet_node_authorization runtime
	#[runtime::pallet_index(8)]
	pub type NodeAuthorization = pallet_node_authorization;
}
```

Open
node/Cargo.toml
```rust
[dependencies]
# addfor node authorization
bs58 = { version = "0.4.0" }
```
open
node/src/node/src/chain_spec.rs

update fn testnet_genesis 
add node ids for ones owned by known accounts.
Alice and Bob
``` rust
//New code
use sp_core::OpaquePeerId;
use node_template_runtime::NodeAuthorizationConfig;

fn testnet_genesis(
    initial_authorities: Vec<(AuraId, GrandpaId)>,
    root_key: AccountId,
    endowed_accounts: Vec<AccountId>,
    _enable_println: bool,
) -> serde_json::Value {
    use serde_json::json;

    // Define the NodeAuthorizationConfig
    //trying new suggestion
 let node_authorization_config = NodeAuthorizationConfig {
    nodes: vec![
        (
            OpaquePeerId(bs58::decode("12D3KooWBmAwcd4PJNJvfV89HwE48nwkRmAgo8Vy3uQEyNNHBox2").into_vec().unwrap()),
            endowed_accounts[0].clone(),
        ),
        (
            OpaquePeerId(bs58::decode("12D3KooWQYV9dGMFoRzNStwpXztXaBUjtPqi6aU76ZgUriHhKust").into_vec().unwrap()),
            endowed_accounts[1].clone(),
        ),
    ],
};
    // Serialize NodeAuthorizationConfig into JSON
    let node_authorization_json = serde_json::to_value(&node_authorization_config).unwrap();

    // Construct the overall genesis configuration JSON
    let genesis_config = json!({
        "balances": {
            // Configure endowed accounts with initial balance of 1 << 60.
            "balances": endowed_accounts.iter().cloned().map(|k| (k, 1u64 << 60)).collect::<Vec<_>>(),
        },
        "aura": {
            "authorities": initial_authorities.iter().map(|x| (x.0.clone())).collect::<Vec<_>>(),
        },
        "grandpa": {
            "authorities": initial_authorities.iter().map(|x| (x.1.clone(), 1)).collect::<Vec<_>>(),
        },
        "sudo": {
            // Assign network admin rights.
            "key": Some(root_key),
        },
        "nodeAuthorization": node_authorization_json, // new code try
    });

    genesis_config
}
```
