# Monero C++ Library

A C++ library for creating Monero applications using native bindings to [monero v0.18.3.1 'Fluorine Fermi'](https://github.com/monero-project/monero/tree/v0.18.3.1).

* Supports fully client-side wallets by wrapping [wallet2.h](https://github.com/monero-project/monero/blob/master/src/wallet/wallet2.h).
* Supports multisig, view-only, and offline wallets.
* Uses a clearly defined [data model and API specification](https://moneroecosystem.org/monero-java/monero-spec.pdf) intended to be intuitive and robust.
* Query wallet transactions, transfers, and outputs by their properties.
* Receive notifications when wallets sync, send, or receive.
* Tested by over 100 tests in [monero-java](https://github.com/monero-ecosystem/monero-java) and [monero-ts](https://github.com/monero-ecosystem/monero-ts) using JNI and WebAssembly bindings.

## Table of contents

* [Sample code](#sample-code)
* [Documentation](#documentation)
* [Using this library in your project](#using-this-library-in-your-project)
* [Related projects](#related-projects)
* [License](#license)
* [Donations](#donations)

## Sample code

```c++
// create a wallet from a seed phrase
monero_wallet_config wallet_config;
wallet_config.m_seed = "hefty value later extra artistic firm radar yodel talent future fungal nutshell because sanity awesome nail unjustly rage unafraid cedar delayed thumbs comb custom sanity";
wallet_config.m_path = "MyWalletRestored";
wallet_config.m_password = "supersecretpassword123";
wallet_config.m_network_type = monero_network_type::STAGENET;
wallet_config.m_server = monero_rpc_connection("http://localhost:38081", "superuser", "abctesting123");
wallet_config.m_restore_height = 380104;
wallet_config.m_seed_offset = "";
monero_wallet* wallet_restored = monero_wallet_full::create_wallet(wallet_config);

// synchronize the wallet and receive progress notifications
struct : monero_wallet_listener {
  void on_sync_progress(uint64_t height, uint64_t start_height, uint64_t end_height, double percent_done, const string& message) {
    // feed a progress bar?
  }
} my_sync_listener;
wallet_restored->sync(my_sync_listener);

// start syncing the wallet continuously in the background
wallet_restored->start_syncing();

// get balance, account, subaddresses
string restored_primary = wallet_restored->get_primary_address();
uint64_t balance = wallet_restored->get_balance(); // can specify account and subaddress indices
monero_account account = wallet_restored->get_account(1, true); // get account with subaddresses
uint64_t unlocked_account_balance = account.m_unlocked_balance.get(); // get boost::optional value

// query a transaction by hash
monero_tx_query tx_query;
tx_query.m_hash = "314a0f1375db31cea4dac4e0a51514a6282b43792269b3660166d4d2b46437ca";
vector<shared_ptr<monero_tx_wallet>> txs = wallet_restored->get_txs(tx_query);
shared_ptr<monero_tx_wallet> tx = txs[0];
for (const shared_ptr<monero_transfer> transfer : tx->get_transfers()) {
  bool is_incoming = transfer->is_incoming().get();
  uint64_t in_amount = transfer->m_amount.get();
  int account_index = transfer->m_account_index.get();
}
monero_utils::free(txs);

// query incoming transfers to account 1
monero_transfer_query transfer_query;
transfer_query.m_is_incoming = true;
transfer_query.m_account_index = 1;
vector<shared_ptr<monero_transfer>> transfers = wallet_restored->get_transfers(transfer_query);
monero_utils::free(transfers);

// query unspent outputs
monero_output_query output_query;
output_query.m_is_spent = false;
vector<shared_ptr<monero_output_wallet>> outputs = wallet_restored->get_outputs(output_query);
monero_utils::free(outputs);

// create and sync a new wallet with a random seed phrase
wallet_config = monero_wallet_config();
wallet_config.m_path = "MyWalletRandom";
wallet_config.m_password = "supersecretpassword123";
wallet_config.m_network_type = monero_network_type::STAGENET;
wallet_config.m_server = monero_rpc_connection("http://localhost:38081", "superuser", "abctesting123");
wallet_config.m_language = "English";
monero_wallet* wallet_random = monero_wallet_full::create_wallet(wallet_config);
wallet_random->sync();

// synchronize in the background every 5 seconds
wallet_random->start_syncing(5000);

// get wallet info
string random_seed = wallet_random->get_seed();
string random_primary = wallet_random->get_primary_address();
uint64_t random_height = wallet_random->get_height();
bool random_is_synced = wallet_random->is_synced();

// receive notifications when funds are received, confirmed, and unlocked
struct : monero_wallet_listener {
  void on_output_received(const monero_output_wallet& output) {
    cout << "Wallet received funds!" << endl;
    uint64_t amount = output.m_amount.get();
    string tx_hash = output.m_tx->m_hash.get();
    bool is_confirmed = output.m_tx->m_is_confirmed.get();
    bool is_locked = dynamic_pointer_cast<monero_tx_wallet>(output.m_tx)->m_is_locked.get();
    int account_index = output.m_account_index.get();
    int subaddress_index = output.m_subaddress_index.get();
    FUNDS_RECEIVED = true;
  }
} my_listener;
wallet_random->add_listener(my_listener);

// send funds from the restored wallet to the random wallet
monero_tx_config tx_config;
tx_config.m_account_index = 0;
tx_config.m_address = wallet_random->get_address(1, 0);
tx_config.m_amount = 50000;
tx_config.m_relay = true;
shared_ptr<monero_tx_wallet> sent_tx = wallet_restored->create_tx(tx_config);
bool in_pool = sent_tx->m_in_tx_pool.get();  // true
monero_utils::free(sent_tx);

// mine with 7 threads to push the network along
int num_threads = 7;
bool is_background = false;
bool ignore_battery = false;
wallet_restored->start_mining(num_threads, is_background, ignore_battery);

// wait for the next block to be added to the chain
uint64_t next_height = wallet_random->wait_for_next_block();

// stop mining
wallet_restored->stop_mining();

// create config to send funds to multiple destinations in the random wallet
tx_config = monero_tx_config();
tx_config.m_account_index = 1; // withdraw funds from this account
tx_config.m_subaddress_indices = vector<uint32_t>();
tx_config.m_subaddress_indices.push_back(0);
tx_config.m_subaddress_indices.push_back(1); // withdraw funds from these subaddresses within the account
tx_config.m_priority = monero_tx_priority::UNIMPORTANT; // no rush
tx_config.m_relay = false; // create transaction and relay to the network if true
vector<shared_ptr<monero_destination>> destinations; // specify the recipients and their amounts
destinations.push_back(make_shared<monero_destination>(wallet_random->get_address(1, 0), 50000));
destinations.push_back(make_shared<monero_destination>(wallet_random->get_address(2, 0), 50000));
tx_config.m_destinations = destinations;

// create the transaction, confirm with the user, and relay to the network
shared_ptr<monero_tx_wallet> created_tx = wallet_restored->create_tx(tx_config);
uint64_t fee = created_tx->m_fee.get(); // "Are you sure you want to send ...?"
wallet_restored->relay_tx(*created_tx); // recipient receives notification within 5 seconds
monero_utils::free(created_tx);

// save and close the wallets
wallet_restored->close(true);
wallet_random->close(true);
delete wallet_restored;
delete wallet_random;
```

## Documentation

* [API documentation](https://moneroecosystem.org/monero-cpp/annotated.html)
* [API and model overview with visual diagrams](https://moneroecosystem.org/monero-java/monero-spec.pdf)
* [monero-ts documentation](https://github.com/monero-ecosystem/monero-ts#documentation) provides additional documentation which translates to monero-cpp

## Using this library in your project

This project may be compiled as part of another application or built as a shared or static library.

For example, [monero-java](https://github.com/monero-ecosystem/monero-java) compiles this project to a shared library to support Java JNI bindings, while [monero-ts](https://github.com/monero-ecosystem/monero-ts) compiles this project to WebAssembly binaries.

### Building monero-cpp on Mac & Linux

1. If building this library standalone instead of as a submodule in another project, clone the project repository and update its submodules:
    1. `git clone --recurse-submodules https://github.com/monero-ecosystem/monero-cpp.git`
    2. `cd ./monero-cpp && ./bin/update_submodules.sh`
2. Install [monero-project dependencies](https://github.com/monero-project/monero#dependencies) for your system. For example, on Ubuntu:<br>
    1. `sudo apt update && sudo apt install build-essential cmake pkg-config libssl-dev libzmq3-dev libsodium-dev libunwind8-dev liblzma-dev libreadline6-dev libpgm-dev qttools5-dev-tools libhidapi-dev libusb-1.0-0-dev libprotobuf-dev protobuf-compiler libudev-dev libboost-chrono-dev libboost-date-time-dev libboost-filesystem-dev libboost-locale-dev libboost-program-options-dev libboost-regex-dev libboost-serialization-dev libboost-system-dev libboost-thread-dev python3 ccache`
    2. Install expat (dependency of unbound):

         ```
         cd ~
         wget https://github.com/libexpat/libexpat/releases/download/R_2_4_8/expat-2.4.8.tar.bz2
         tar -xf expat-2.4.8.tar.bz2
         sudo rm expat-2.4.8.tar.bz2
         cd expat-2.4.8
         ./configure --enable-static --disable-shared
         make
         sudo make install
         cd ../
         ```
     3. Install unbound:

         ```
         cd ~
         wget https://www.nlnetlabs.nl/downloads/unbound/unbound-1.17.0.tar.gz
         tar xzf unbound-1.17.0.tar.gz
         sudo apt update
         sudo apt install -y build-essential
         sudo apt install -y libssl-dev
         sudo apt install -y libexpat1-dev
         sudo apt-get install -y bison
         sudo apt-get install -y flex
         cd unbound-1.17.0
         ./configure --with-libexpat=/usr --with-ssl=/usr
         make
         sudo make install
         cd ../
         ```
3. `export MONERO_CPP=path/to/monero-cpp`
4. `cd $MONERO_CPP/external/monero-project`
5. Build monero-project to create .a libraries, e.g.: `make release-static -j8`
6. Link to this library's source files in your application or build as a shared library in ./build/: `cd $MONERO_CPP && ./bin/build_libmonero_cpp.sh`

### Building libmonero-cpp.dll on Windows

1. Download and install [MSYS2](https://www.msys2.org/).
2. Press the Windows button and launch `MSYS2 MINGW64` for 64 bit systems or `MSYS2 MINGW32` for 32 bit.
3. Update packages: `pacman -Syu` and confirm at prompts.
4. Relaunch MSYS2 (if necessary) and install dependencies:

    For 64 bit:

     ```
     pacman -S mingw-w64-x86_64-toolchain make mingw-w64-x86_64-cmake mingw-w64-x86_64-boost mingw-w64-x86_64-openssl mingw-w64-x86_64-zeromq mingw-w64-x86_64-libsodium mingw-w64-x86_64-hidapi mingw-w64-x86_64-unbound mingw-w64-x86_64-protobuf git mingw-w64-x86_64-libusb gettext base-devel
     ```

     For 32 bit:

     ```
     pacman -S  mingw-w64-i686-toolchain make mingw-w64-i686-cmake mingw-w64-i686-boost mingw-w64-i686-openssl mingw-w64-i686-zeromq mingw-w64-i686-libsodium mingw-w64-i686-hidapi mingw-w64-i686-unbound mingw-w64-i686-protobuf git mingw-w64-i686-libusb gettext base-devel
     ```
5. Clone repo if installing standalone (skip if building as part of another repo like monero-java or monero-ts): `git clone --recursive https://github.com/monero-ecosystem/monero-cpp.git`
6. Update submodules: `cd <path/to/monero-cpp> && ./bin/update_submodules.sh`
7. `cd ./external/monero-project`
9. Build monero-project:

    For 64 bit: `make release-static-win64`
    
    For 32 bit: `make release-static-win32`
10. Build as a shared library to ./build/: `cd ../../ && ./bin/build_libmonero_cpp.sh`

## Running sample code and tests

1. In CMakeLists.txt, set the flags to build:

         set(BUILD_LIBRARY ON)
         set(BUILD_SAMPLE ON)
         set(BUILD_SCRATCHPAD ON)
         set(BUILD_TESTS ON)
2. `./bin/build_libmonero_cpp.sh`
3. Run the app, for example: `./build/sample_code`

## Related projects

* [monero-java](https://github.com/monero-ecosystem/monero-java)
* [monero-ts](https://github.com/monero-ecosystem/monero-ts)

## License

This project is licensed under MIT.

## Donations

If this library brings you value, please consider donating.

<p align="center">
	<img src="donate.png" width="115" height="115"/><br>
	<code>46FR1GKVqFNQnDiFkH7AuzbUBrGQwz2VdaXTDD4jcjRE8YkkoTYTmZ2Vohsz9gLSqkj5EM6ai9Q7sBoX4FPPYJdGKQQXPVz</code>
</p>
