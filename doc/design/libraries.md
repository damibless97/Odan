# Libraries

| Name                     | Description |
|--------------------------|-------------|
| *libodan_cli*         | RPC client functionality used by *odan-cli* executable |
| *libodan_common*      | Home for common functionality shared by different executables and libraries. Similar to *libodan_util*, but higher-level (see [Dependencies](#dependencies)). |
| *libodan_consensus*   | Stable, backwards-compatible consensus functionality used by *libodan_node* and *libodan_wallet*. |
| *libodan_crypto*      | Hardware-optimized functions for data encryption, hashing, message authentication, and key derivation. |
| *libodan_kernel*      | Consensus engine and support library used for validation by *libodan_node*. |
| *libodanqt*           | GUI functionality used by *odan-qt* and *odan-gui* executables. |
| *libodan_ipc*         | IPC functionality used by *odan-node*, *odan-wallet*, *odan-gui* executables to communicate when [`--enable-multiprocess`](multiprocess.md) is used. |
| *libodan_node*        | P2P and RPC server functionality used by *odand* and *odan-qt* executables. |
| *libodan_util*        | Home for common functionality shared by different executables and libraries. Similar to *libodan_common*, but lower-level (see [Dependencies](#dependencies)). |
| *libodan_wallet*      | Wallet functionality used by *odand* and *odan-wallet* executables. |
| *libodan_wallet_tool* | Lower-level wallet functionality used by *odan-wallet* executable. |
| *libodan_zmq*         | [ZeroMQ](../zmq.md) functionality used by *odand* and *odan-qt* executables. |

## Conventions

- Most libraries are internal libraries and have APIs which are completely unstable! There are few or no restrictions on backwards compatibility or rules about external dependencies. An exception is *libodan_kernel*, which, at some future point, will have a documented external interface.

- Generally each library should have a corresponding source directory and namespace. Source code organization is a work in progress, so it is true that some namespaces are applied inconsistently, and if you look at [`libodan_*_SOURCES`](../../src/Makefile.am) lists you can see that many libraries pull in files from outside their source directory. But when working with libraries, it is good to follow a consistent pattern like:

  - *libodan_node* code lives in `src/node/` in the `node::` namespace
  - *libodan_wallet* code lives in `src/wallet/` in the `wallet::` namespace
  - *libodan_ipc* code lives in `src/ipc/` in the `ipc::` namespace
  - *libodan_util* code lives in `src/util/` in the `util::` namespace
  - *libodan_consensus* code lives in `src/consensus/` in the `Consensus::` namespace

## Dependencies

- Libraries should minimize what other libraries they depend on, and only reference symbols following the arrows shown in the dependency graph below:

<table><tr><td>

```mermaid

%%{ init : { "flowchart" : { "curve" : "basis" }}}%%

graph TD;

odan-cli[odan-cli]-->libodan_cli;

odand[odand]-->libodan_node;
odand[odand]-->libodan_wallet;

odan-qt[odan-qt]-->libodan_node;
odan-qt[odan-qt]-->libodanqt;
odan-qt[odan-qt]-->libodan_wallet;

odan-wallet[odan-wallet]-->libodan_wallet;
odan-wallet[odan-wallet]-->libodan_wallet_tool;

libodan_cli-->libodan_util;
libodan_cli-->libodan_common;

libodan_consensus-->libodan_crypto;

libodan_common-->libodan_consensus;
libodan_common-->libodan_crypto;
libodan_common-->libodan_util;

libodan_kernel-->libodan_consensus;
libodan_kernel-->libodan_crypto;
libodan_kernel-->libodan_util;

libodan_node-->libodan_consensus;
libodan_node-->libodan_crypto;
libodan_node-->libodan_kernel;
libodan_node-->libodan_common;
libodan_node-->libodan_util;

libodanqt-->libodan_common;
libodanqt-->libodan_util;

libodan_util-->libodan_crypto;

libodan_wallet-->libodan_common;
libodan_wallet-->libodan_crypto;
libodan_wallet-->libodan_util;

libodan_wallet_tool-->libodan_wallet;
libodan_wallet_tool-->libodan_util;

classDef bold stroke-width:2px, font-weight:bold, font-size: smaller;
class odan-qt,odand,odan-cli,odan-wallet bold
```
</td></tr><tr><td>

**Dependency graph**. Arrows show linker symbol dependencies. *Crypto* lib depends on nothing. *Util* lib is depended on by everything. *Kernel* lib depends only on consensus, crypto, and util.

</td></tr></table>

- The graph shows what _linker symbols_ (functions and variables) from each library other libraries can call and reference directly, but it is not a call graph. For example, there is no arrow connecting *libodan_wallet* and *libodan_node* libraries, because these libraries are intended to be modular and not depend on each other's internal implementation details. But wallet code is still able to call node code indirectly through the `interfaces::Chain` abstract class in [`interfaces/chain.h`](../../src/interfaces/chain.h) and node code calls wallet code through the `interfaces::ChainClient` and `interfaces::Chain::Notifications` abstract classes in the same file. In general, defining abstract classes in [`src/interfaces/`](../../src/interfaces/) can be a convenient way of avoiding unwanted direct dependencies or circular dependencies between libraries.

- *libodan_crypto* should be a standalone dependency that any library can depend on, and it should not depend on any other libraries itself.

- *libodan_consensus* should only depend on *libodan_crypto*, and all other libraries besides *libodan_crypto* should be allowed to depend on it.

- *libodan_util* should be a standalone dependency that any library can depend on, and it should not depend on other libraries except *libodan_crypto*. It provides basic utilities that fill in gaps in the C++ standard library and provide lightweight abstractions over platform-specific features. Since the util library is distributed with the kernel and is usable by kernel applications, it shouldn't contain functions that external code shouldn't call, like higher level code targetted at the node or wallet. (*libodan_common* is a better place for higher level code, or code that is meant to be used by internal applications only.)

- *libodan_common* is a home for miscellaneous shared code used by different Odan Core applications. It should not depend on anything other than *libodan_util*, *libodan_consensus*, and *libodan_crypto*.

- *libodan_kernel* should only depend on *libodan_util*, *libodan_consensus*, and *libodan_crypto*.

- The only thing that should depend on *libodan_kernel* internally should be *libodan_node*. GUI and wallet libraries *libodanqt* and *libodan_wallet* in particular should not depend on *libodan_kernel* and the unneeded functionality it would pull in, like block validation. To the extent that GUI and wallet code need scripting and signing functionality, they should be get able it from *libodan_consensus*, *libodan_common*, *libodan_crypto*, and *libodan_util*, instead of *libodan_kernel*.

- GUI, node, and wallet code internal implementations should all be independent of each other, and the *libodanqt*, *libodan_node*, *libodan_wallet* libraries should never reference each other's symbols. They should only call each other through [`src/interfaces/`](../../src/interfaces/) abstract interfaces.

## Work in progress

- Validation code is moving from *libodan_node* to *libodan_kernel* as part of [The libodankernel Project #24303](https://github.com/odan/odan/issues/24303)
- Source code organization is discussed in general in [Library source code organization #15732](https://github.com/odan/odan/issues/15732)
