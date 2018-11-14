
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The accounts package implements the Ethereum client's wallet and account management. Ethereum's wallet offers both keyStore mode and usb wallet. At the same time, the ABI code of the Ethereum contract is also placed in the account/abi directory. The abi project seems to have nothing to do with account management. For the time being, only the interface of account management is analyzed. The implementation code for the specific keystore and usb will not be given for the time being.



Accounts are defined by data structures and interfaces.

## data structure
account number

// Account represents an Ethereum account located at a specific location defined
// by the optional URL field.
// An account is 20 bytes of data. The URL is an optional field.
Type Account struct {
Address common.Address `json:"address"` // Ethereum account address derived from the key
URL URL `json:"url"` // Optional resource locator within a backend
}

Const (
HashLength = 32
AddressLength = 20
)
// Address represents the 20 byte address of an Ethereum account.
Type Address [AddressLength]byte


wallet. The wallet should be the most important interface in it. The specific wallet also implements this interface.
The wallet has a so-called layered deterministic wallet and a regular wallet.

// Wallet represents a software or hardware wallet that might contain one or more
// accounts (derived from the same seed).
// Wallet is a software wallet or hardware wallet that contains one or more accounts.
Type Wallet interface {
// URL retrieves the canonical path under which this wallet is reachable. It is
// user by upper layers to define a sorting order over all wallets from multiple
// backends.
// The URL is used to get the canonical path that this wallet can access. It will be used by the upper layer to sort from all backend wallets.
URL() URL

// Status returns a textual status to aid the user in the current state of the
// wallet. It also returns an error indicating any failure the wallet might have
//Having encountered.
// Used to return a text value that identifies the current state of the wallet. It also returns an error to identify any errors encountered by the wallet.
Status() (string, error)

// Open initializes access to a wallet instance. It is not meant to unlock or
// decrypt account keys, rather simply to establish a connection to hardware
// wallets and/or to access derivation seeds.
// Open initializes access to the wallet instance. This approach does not mean unlocking or decrypting the account, but simply establishing a connection to the hardware wallet and/or accessing the derived seed.
// The passphrase parameter may or may not be used by the implementation of a
// particular wallet instance. The reason there is no passwordless open method
// is to strive towards a uniform wallet handling, oblivious to the different
// backend providers.
// The passphrase parameter may not be needed in some implementations. The reason for not providing an Open method without a passphrase parameter is to provide a uniform interface.
// Please note, if you open a wallet, you must close it to release any allocated
// resources (especially important when working with hardware wallets).
// Please note that if you open a wallet, you must close it. Otherwise some resources may not be released. Especially when using a hardware wallet, you need to pay special attention.
Open(passphrase string) error

// Close releases any resources held by an open wallet instance.
// Close Frees any resources occupied by the Open method.
Close() error

// Accounts retrieves the list of signing accounts the wallet is currently aware
// of. For hierarchical deterministic wallets, the list will not be exhaustive,
// rather only the accounts during pinned during account derivation.
// Accounts are used to get the wallet and found a list of accounts. For a hierarchical wallet, this list does not list all of the accounts in detail, but only the accounts that were explicitly fixed during the account's derivation.
Accounts() []Account

// Contains returns whether an account is part of this particular wallet or not.
// Contains Returns whether an account belongs to this wallet.
Contains(account Account) bool

// Derive attempts to explicitly derive a hierarchical deterministic account at
// the specified derivation path. If requested, the derived account will be added
// to the wallet's tracked account list.
// Derive attempts to explicitly derive a hierarchical deterministic account on the specified derived path. If the pin is true, the derived account will be added to the tracking account list in the wallet.
Derive(path DerivationPath, pin bool) (Account, error)

// SelfDerive sets a base account derivation path from which the wallet attempts
// to discover non zero accounts and automatically add them to list of tracked
// accounts.
// SelfDerive sets a basic account export path from which the wallet attempts to discover non-zero accounts and automatically adds them to the tracking account list.
// Note, self derivaton will increment the last component of the specified path
//mentioned to decending into a child path to allow discovering accounts starting
// from non zero components.
// Note that SelfDerive will increment the last component of the specified path instead of dropping it to a subpath to allow discovery of accounts from non-zero components.
// You can disable automatic account discovery by calling SelfDerive with a nil
// chain state reader.
// You can disable automatic account discovery by passing a nil's ChainStateReader.
SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)

// SignHash requests the wallet to sign the given hash.
// SignHash requests the wallet to sign the incoming hash.
// It looks up the account specified either individually via its address contained within,
// or annular with the aid of any location metadata from the embedded URL field.
// It can look up the specified account by the address it contains (or optionally by any location metadata in the embedded URL field).
// If the wallet requires additional authentication to sign the request (e.g.
// a password to decrypt the account, or a PIN code o verify the transaction),
// an AuthNeededError instance will be returned, containing infos for the user
// about which fields or actions are needed. The user may retry by providing
// the needed details via SignHashWithPassphrase, or by other means (e.g. unlock
// the account in a keystore).
// If the wallet requires additional verification to sign (for example, a password is required to unlock the account, or a PIN code is required to verify the transaction.)
// will return an AuthNeededError with the user's information and which fields or operations need to be provided.
// Users can sign by SignHashWithPassphrase or by other means (unlock the account in the keystore)
SignHash(account Account, hash []byte) ([]byte, error)

// SignTx requests the wallet to sign the given transaction.
// SignTx requests the wallet to sign the specified transaction.
// It looks up the account specified either individually via its address contained within,
// or annular with the aid of any location metadata from the embedded URL field.
//
// If the wallet requires additional authentication to sign the request (e.g.
// a password to decrypt the account, or a PIN code o verify the transaction),
// an AuthNeededError instance will be returned, containing infos for the user
// about which fields or actions are needed. The user may retry by providing
// the needed details via SignTxWithPassphrase, or by other means (e.g. unlock
// the account in a keystore).
SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)

// SignHashWithPassphrase requests the wallet to sign the given hash with the
// given passphrase as extra authentication information.
// SignHashWithPassphrase requests the wallet to sign the given hash using the given passphrase
// It looks up the account specified either individually via its address contained within,
// or annular with the aid of any location metadata from the embedded URL field.
SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)

//SignTxWithPassphrase requests the wallet to sign the given transaction, with the
// given passphrase as extra authentication information.
// SignHashWithPassphrase requests the wallet to sign the given transaction using the given passphrase
// It looks up the account specified either individually via its address contained within,
// or annular with the aid of any location metadata from the embedded URL field.
SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
}


Backend Backend

// Backend is a "wallet provider" that may contain a batch of accounts they can
// sign transactions with and upon request, do so.
// Backend is a wallet provider. Can contain a batch of accounts. They can sign the transaction on request and do so.
Type Backend interface {
// Wallets retrieves the list of wallets the backend is currently aware of.
// Wallets get the wallet that can be found currently
// The returned wallets are not opened by default. For software HD wallets this
// means that no base seeds are decrypted, and for hardware wallets that no actual
// connection is established.
// The returned wallet is not open by default.
// The resulting wallet list will be sorted alphabetically based on its internal
// URL assigned by the backend. Since wallets (especially hardware) may come and
// go, the same wallet might appear at a different positions in the list during
//after retrievals.
//The resulting list of wallets will be sorted alphabetically according to the internal URL assigned by the backend. Since wallets (especially hardware wallets) may be turned on and off, the same wallet may appear in different locations in the list during subsequent retrievals.
Wallets() []Wallet

// Subscribe creates an async subscription to receive notifications when the
// backend detects the arrival or departure of a wallet.
// Subscribe to create an asynchronous subscription to receive notifications when the backend detects the arrival or departure of the wallet.
Subscribe(sink chan&lt;- WalletEvent) event.Subscription
}


## manager.go
Manager is an account management tool that contains everything. You can communicate with all Backends to sign a transaction.

data structure

// Manager is an overarching account manager that can communicate with various
// backends for signing transactions.
Type manager struct {
// All registered Backends
Backends map[reflect.Type][]Backend // Index of backends currently registered
// All Backend update subscribers
Updaters []event.Subscription // Wallet update subscriptions for all backends
// backend updated subscription slot
Updates chan WalletEvent // Subscription sink for backend wallet changes
// Cache of all registered Backends wallets
Wallets []Wallet // Cache of all wallets from all registered backends
// Notice of the arrival and departure of the wallet
Feed event.Feed // Wallet feed notifying of arrivals/departures
// exit the queue
Quit chan chan error
Lock sync.RWMutex
}


Create Manager


// NewManager creates a generic account manager to sign transaction via various
// supported backends.
Func NewManager(backends ...Backend) *Manager {
// Subscribe to wallet notifications from all backends
Updates := make(chan WalletEvent, 4*len(backends))

Subs := make([]event.Subscription, len(backends))
For i, backend := range backends {
Subs[i] = backend.Subscribe(updates)
}
// Retrieve the initial list of wallets from the backends and sort by URL
Var wallets []Wallet
For _, backend := range backends {
Wallets = merge(wallets, backend.Wallets()...)
}
// Assemble the account manager and return
Am := &amp;Manager{
Backends: make(map[reflect.Type][]Backend),
Updaters: subs,
Updates: updates,
Wallets: wallets,
Quit: make(chan chan error),
}
For _, backend := range backends {
Kind := reflect.TypeOf(backend)
Am.backends[kind] = append(am.backends[kind], backend)
}
Go am.update()

Return am
}

Update method. Is a goroutine. Will monitor all update information triggered by backend. Then forward it to the feed.

// update is the wallet event loop listening for notifications from the backends
// and updating the cache of wallets.
Func (am *Manager) update() {
// Close all subscriptions when the manager terminates
Defer func() {
am.lock.Lock()
For _, sub := range am.updaters {
sub.Unsubscribe()
}
Am.updaters = nil
am.lock.Unlock()
}()

// Loop until termination
For {
Select {
Case event := &lt;-am.updates:
// Wallet event arrived, update local cache
am.lock.Lock()
Switch event.Kind {
Case WalletArrived:
Am.wallets = merge(am.wallets, event.Wallet)
Case WalletDropped:
Am.wallets = drop(am.wallets, event.Wallet)
}
am.lock.Unlock()

// Notify any listeners of the event
am.feed.Send(event)

Case errc := &lt;-am.quit:
// Manager terminating, return
Errc &lt;- nil
Return
}
}
}

Return backend

// Backends retrieves the backend(s) with the given type from the account manager.
Func (am *Manager) Backends(kind reflect.Type) []Backend {
Return am.backends[kind]
}


Subscription message

// Subscribe creates an async subscription to receive notifications when the
// manager detects the arrival or departure of a wallet from any of its backends.
Func (am *Manager) Subscribe(sink chan&lt;- WalletEvent) event.Subscription {
Return am.feed.Subscribe(sink)
}


For node. When was the account manager created?

// New creates a new P2P node, ready for protocol registration.
Func New(conf *Config) (*Node, error) {
...
Am, ephemeralKeystore, err := makeAccountManager(conf)




Func makeAccountManager(conf *Config) (*accounts.Manager, string, error) {
scryptN := keystore.StandardScryptN
scryptP := keystore.StandardScryptP
If conf.UseLightweightKDF {
scryptN = keystore.LightScryptN
scryptP = keystore.LightScryptP
}

Var (
Keydir string
Ephemeral string
Err error
)
Switch {
Case filepath.IsAbs(conf.KeyStoreDir):
Keydir = conf.KeyStoreDir
Case conf.DataDir != "":
If conf.KeyStoreDir == "" {
Keydir = filepath.Join(conf.DataDir, datadirDefaultKeyStore)
} else {
Keydir, err = filepath.Abs(conf.KeyStoreDir)
}
Case conf.KeyStoreDir != "":
Keydir, err = filepath.Abs(conf.KeyStoreDir)
Default:
// There is no datadir.
Keydir, err = ioutil.TempDir("", "go-ethereum-keystore")
Ephemeral = keydir
}
If err != nil {
Return nil, "", err
}
If err := os.MkdirAll(keydir, 0700); err != nil {
Return nil, "", err
}
// Assemble the account manager and supported backends
// Created a backend of the KeyStore
Backends := []accounts.Backend{
keystore.NewKeyStore(keydir, scryptN, scryptP),
}
// If it is a USB wallet. Need to do some extra work.
If !conf.NoUSB {
// Start a USB hub for Ledger hardware wallets
If ledgerhub, err := usbwallet.NewLedgerHub(); err != nil {
log.Warn(fmt.Sprintf("Failed to start Ledger hub, disabling: %v", err))
} else {
Backends = append(backends, ledgerhub)
}
// Start a USB hub for Trezor hardware wallets
If trezorhub, err := usbwallet.NewTrezorHub(); err != nil {
log.Warn(fmt.Sprintf("Failed to start Trezor hub, disabling: %v", err)
} else {
Backends = append(backends, trezorhub)
}
}
Return accounts.NewManager(backends...), ephemeral, nil
}
</pre></body></html>