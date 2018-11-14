
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## contract.go
Contract represents a contract in the Ethereum state database. Contains the contract code and calls the parameters.


structure

// ContractRef is a reference to the contract's backing object
Type ContractRef interface {
Address() common.Address
}

// AccountRef implements ContractRef.
//
// Account references are used during EVM initialisation and
// it's primary use is to fetch addresses. Removing this object
// proves difficult because of the cached jump destinations which
// are fetched from the parent contract (i.e. the caller), which
// is a ContractRef.
Type AccountRef common.Address

// Address casts AccountRef to a Address
Func (ar AccountRef) Address() common.Address { return (common.Address)(ar) }

// Contract represents an ethereum contract in the state database. It contains
// the the contract code, calling arguments. Contract implements ContractRef
Type Contract struct {
// CallerAddress is the result of the caller which initialised this
// contract. However when the "call method" is delegated this value
// needs to be initialised to that of the caller's caller.
// CallerAddress is the person who initialized the contract. If it is a delegate, this value is set to the caller of the caller.
CallerAddress common.Address
Caller ContractRef
Self ContractRef

Jumpdests destinations // result of JUMPDEST analysis. Analysis of JUMPDEST instructions

Code [] byte / / code
CodeHash common.Hash // code HASH
CodeAddr *common.Address // code address
Input []byte // Enter the parameter

Gas uint64 // How many Gases are there in the contract?
Value *big.Int

Args []byte // does not seem to be used

DelegateCall bool
}

structure

// NewContract returns a new contract environment for the execution of EVM.
Func NewContract(caller ContractRef, object ContractRef, value *big.Int, gas uint64) *Contract {
c := &amp;Contract{CallerAddress: caller.Address(), caller: caller, self: object, Args: nil}

If parent, ok := caller.(*Contract); ok {
// Reuse JUMPDEST analysis from parent context if available.
// If the caller is a contract, it means that the contract called us. Jumpdests set to jumper's jumpdests
C.jumpdests = parent.jumpdests
} else {
C.jumpdests = make(destinations)
}

// Gas should be a pointer so it can safely be reduced through the run
// This pointer will be off the state transition
c.Gas = gas
// guarantee a value is set
C.value = value

Return c
}

AsDelegate sets the contract as a delegate call and returns the current contract (for chained calls)

// AsDelegate sets the contract to be a delegate call and returns the current
// contract (for chaining calls)
Func (c *Contract) AsDelegate() *Contract {
c.DelegateCall = true
// NOTE: caller must, at all times be a contract. It should never happen
// that caller is something other than a Contract.
Parent := c.caller.(*Contract)
c.CallerAddress = parent.CallerAddress
C.value = parent.value

Return c
}

GetOp is used to get the next hop instruction

// GetOp returns the n'th element in the contract's byte array
Func (c *Contract) GetOp(n uint64) OpCode {
Return OpCode(c.GetByte(n))
}

// GetByte returns the n'th byte in the contract's byte array
Func (c *Contract) GetByte(n uint64) byte {
If n &lt; uint64(len(c.Code)) {
Return c.Code[n]
}

Return 0
}

// Caller returns the caller of the contract.
//
// Caller will recursively call caller when the contract is a delegate
// call, including that of caller's caller.
Func (c *Contract) Caller() common.Address {
Return c.CallerAddress
}
UseGas uses Gas.

// UseGas attempts the use gas and subtracts it and returns true on success
Func (c *Contract) UseGas(gas uint64) (ok bool) {
If c.Gas &lt; gas {
Return false
}
c.Gas -= gas
Return true
}

// Address returns the contracts address
Func (c *Contract) Address() common.Address {
Return c.self.Address()
}

// Value returns the contracts value (sent to it from it's caller)
Func (c *Contract) Value() *big.Int {
Return c.value
}
SetCode, SetCallCode set the code.

// SetCode sets the code to the contract
Func (self *Contract) SetCode(hash common.Hash, code []byte) {
self.Code = code
self.CodeHash = hash
}

// SetCallCode sets the code of the contract and address of the backing data
// object
Func (self *Contract) SetCallCode(addr *common.Address, hash common.Hash, code []byte) {
self.Code = code
self.CodeHash = hash
self.CodeAddr = addr
}


## evm.go

structure


// Context provides the EVM with auxiliary information. Once provided
// it shouldn't be modified.
// The context provides auxiliary information for the EVM. Once provided, it should not be modified.
Type Context struct {
// CanTransfer returns whether the account contains
// sufficient ether to transfer the value
// The CanTransfer function returns if the account has enough ether to transfer
CanTransfer CanTransferFunc
// Transfer transfers ether from one account to the other
// Transfer is used to transfer money from one account to another
Transfer TransferFunc
// GetHash returns the hash corresponding to n
// GetHash is used to return the hash value corresponding to the input parameter n
GetHash GetHashFunc

// Message information
// used to provide information about Origin. address of sender
Origin common.Address // Provides information for ORIGIN
// used to provide GasPrice information
GasPrice *big.Int // Provides information for GASPRICE

// Block information
Coinbase common.Address // Provides information for COINBASE
GasLimit *big.Int // Provides information for GASLIMIT
BlockNumber *big.Int // Provides information for NUMBER
Time *big.Int // Provides information for TIME
Difficulty *big.Int // Provides information for DIFFICULTY
}

// EVM is the Ethereum Virtual Machine base object and provides
// the necessary tools to run a contract on the given state with
// the provided context. It should be noted that any error
// generated through any of the calls should be considered a
// revert-state-and-consume-all-gas operation, no checks on
// specific errors should ever be performed. The interpreter makes
// sure that any errors generated are to be considered faulty code.
// EVM Ethereum virtual machine base object and provides the necessary tools to run the contract for a given state using the provided context.
// It should be noted that any errors generated by any call should be considered a rollback modification state and consume all GAS operations.
// You should not perform a check for specific errors. The interpreter ensures that any errors generated are considered to be the wrong code.
// The EVM should never be reused and is not thread safe.
Type EVM struct {
// Context provides auxiliary blockchain related information
Context
// StateDB gives access to the underlying state
StateDB StateDB
// Depth is the current call stack
// current call stack
Depth int

// chainConfig contains information about the current chain
// contains information about the current blockchain
chainConfig *params.ChainConfig
// chain rules contains the chain rules for the current epoch
chainRules params.Rules
// virtual machine configuration options used to initialise the
// evm.
vmConfig Config
// global (to this context) ethereum virtual machine
// used throughout the execution of the tx.
Interpreter *Interpreter
// abort is used to abort the EVM calling
// NOTE: must be set atomically
Abort int32
}

Constructor

// NewEVM retutrns a new EVM . The returned EVM is not thread safe and should
// only ever be used *once*.
Func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {
Evm := &amp;EVM{
Context: ctx,
StateDB: statedb,
vmConfig: vmConfig,
chainConfig: chainConfig,
chainRules: chainConfig.Rules(ctx.BlockNumber),
}

Evm.interpreter = NewInterpreter(evm, vmConfig)
Return evm
}

// Cancel cancels any running EVM operation. This may be called concurrently and
// it's safe to be called multiple times.
Func (evm *EVM) Cancel() {
atomic.StoreInt32(&amp;evm.abort, 1)
}


Contract creation Create will create a new contract.


// Create creates a new contract using code as deployment code.
Func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {

// Depth check execution. Fail if we're trying to execute above the
// limit.
If evm.depth &gt; int(params.CallCreateDepth) {
Return nil, common.Address{}, gas, ErrDepth
}
If !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
Return nil, common.Address{}, gas, ErrInsufficientBalance
}
// Ensure there's no existing contract already at the designated address
// Make sure there is no contract exists for a specific address
Nonce := evm.StateDB.GetNonce(caller.Address())
evm.StateDB.SetNonce(caller.Address(), nonce+1)

contractAddr = crypto.CreateAddress(caller.Address(), nonce)
contractHash := evm.StateDB.GetCodeHash(contractAddr)
If evm.StateDB.GetNonce(contractAddr) != 0 || (contractHash != (common.Hash{}) &amp;&amp; contractHash != emptyCodeHash) { //If it already exists
Return nil, common.Address{}, 0, ErrContractAddressCollision
}
// Create a new account on the state
Snapshot := evm.StateDB.Snapshot() //Create a snapshot of StateDB for rollback
evm.StateDB.CreateAccount(contractAddr) //Create an account
If evm.ChainConfig().IsEIP158(evm.BlockNumber) {
evm.StateDB.SetNonce(contractAddr, 1) //Set nonnce
}
evm.Transfer(evm.StateDB, caller.Address(), contractAddr, value) //Transfer

// initialise a new contract and set the code that is to be used by the
// E The contract is a scoped evmironment for this execution context
// only.
Contract := NewContract(caller, AccountRef(contractAddr), value, gas)
contract.SetCallCode(&amp;contractAddr, crypto.Keccak256Hash(code), code)

If evm.vmConfig.NoRecursion &amp;&amp; evm.depth &gt; 0 {
Return nil, contractAddr, gas, nil
}
Ret, err = run(evm, snapshot, contract, nil) //The initialization code of the execution contract
// check whether the max code size has been exceeded
/ / Check the length of the code generated by the initialization does not exceed the limit
maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) &amp;&amp; len(ret) &gt; params.MaxCodeSize
// if the contract creation ran successfully and no errors were returned
// calculate the gas required to store the code. If the code could not
// be stored due to not enough gas set an error and let it be handled
// by the error checking condition below.
// If the contract was created successfully and returned without errors, calculate the GAS required to store the code. If the code cannot be stored incorrectly due to insufficient GAS, it is handled by the following error checking conditions.
If err == nil &amp;&amp; !maxCodeSizeExceeded {
createDataGas := uint64(len(ret)) * params.CreateDataGas
If contract.UseGas(createDataGas) {
evm.StateDB.SetCode(contractAddr, ret)
} else {
Err = ErrCodeStoreOutOfGas
}
}

// When an error was returned by the EVM or when setting the creation code
// above we revert to the snapshot and consume any gas remaining.
// when we're in homestead this also counts for code storage gas errors.
// When the error returns we roll back the changes,
If maxCodeSizeExceeded || (err != nil &amp;&amp; (evm.ChainConfig().IsHomestead(evm.BlockNumber) || err != ErrCodeStoreOutOfGas)) {
evm.StateDB.RevertToSnapshot(snapshot)
If err != errExecutionReverted {
contract.UseGas(contract.Gas)
}
}
// Assign err if contract code size exceeds the max while the err is still empty.
If maxCodeSizeExceeded &amp;&amp; err == nil {
Err = errMaxCodeSizeExceeded
}
Return ret, contractAddr, contract.Gas, err
}


The Call method, whether we transfer the money or execute the contract code, will be called here, and the call instruction in the contract will also be executed here.


// Call executing the contract associated with the addr with the given input as
// parameters. It also handles any necessary value transfer required and takes
// the necessary steps to create accounts and reverses the state in case of an
// execution error or failed value transfer.

// Call executes the contract associated with addr with the given input as a parameter.
// It also handles any necessary transfer operations required and takes the necessary steps to create an account
// and roll back the operation in case of any errors.

Func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
If evm.vmConfig.NoRecursion &amp;&amp; evm.depth &gt; 0 {
Return nil, gas, nil
}

// Fail if we're trying to execute above the call depth limit
/ / Call depth up to 1024
If evm.depth &gt; int(params.CallCreateDepth) {
Return nil, gas, ErrDepth
}
// Fail if we're trying to transfer more than the available balance
// Check if our account has enough money.
If !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
Return nil, gas, ErrInsufficientBalance
}

Var (
To = AccountRef(addr)
Snapshot = evm.StateDB.Snapshot()
)
If !evm.StateDB.Exist(addr) { // Check if the specified address exists
// If the address does not exist, check if it is a native go contract, the native go contract is
// inside the contracts.go file
Precompiles := PrecompiledContractsHomestead
If evm.ChainConfig().IsByzantium(evm.BlockNumber) {
Precompiles = PrecompiledContractsByzantium
}
If precompiles[addr] == nil &amp;&amp; evm.ChainConfig().IsEIP158(evm.BlockNumber) &amp;&amp; value.Sign() == 0 {
// If it is not the specified contract address, and the value of value is 0 then returns to normal, and this call does not consume Gas
Return nil, gas, nil
}
// Responsible for creating addr in local state
evm.StateDB.CreateAccount(addr)
}
// Perform a transfer
evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)

// initialise a new contract and set the code that is to be used by the
// E The contract is a scoped environment for this execution context
// only.
Contract := NewContract(caller, to, value, gas)
contract.SetCallCode(&amp;addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

Ret, err = run(evm, snapshot, contract, input)
// When an error was returned by the EVM or when setting the creation code
// above we revert to the snapshot and consume any gas remaining.
// when we're in homestead this also counts for code storage gas errors.
If err != nil {
evm.StateDB.RevertToSnapshot(snapshot)
If err != errExecutionReverted {
// If it is an error triggered by the revert command, because ICO generally sets a limit on the number of people or funds
// These restrictions are likely to be triggered when you snap up, causing a lot of money to be drawn. at this time
// You can't set the lower GasPrice and GasLimit. Because it is fast.
// Then don't use all the remaining Gas, but only the Gas that is executed using the code
// Otherwise it will be taken away by GasLimit *GasPrice, which is quite a lot.
contract.UseGas(contract.Gas)
}
}
Return ret, contract.Gas, err
}


The remaining three functions, CallCode, DelegateCall, and StaticCall, cannot be called externally and can only be triggered by Opcode.


CallCode

// CallCode differs from Call in the sense that it executes the given address'
// code with the caller as context.
// The difference between CallCode and Call is that it uses the caller's context to execute the code for the given address.

Func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
If evm.vmConfig.NoRecursion &amp;&amp; evm.depth &gt; 0 {
Return nil, gas, nil
}

// Fail if we're trying to execute above the call depth limit
If evm.depth &gt; int(params.CallCreateDepth) {
Return nil, gas, ErrDepth
}
// Fail if we're trying to transfer more than the available balance
If !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
Return nil, gas, ErrInsufficientBalance
}

Var (
Snapshot = evm.StateDB.Snapshot()
To = AccountRef(caller.Address()) //This is the most different place. The address of to is changed to the address of the caller and there is no transfer behavior.
)
// initialise a new contract and set the code that is to be used by the
// E The contract is a scoped evmironment for this execution context
// only.
Contract := NewContract(caller, to, value, gas)
contract.SetCallCode(&amp;addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

Ret, err = run(evm, snapshot, contract, input)
If err != nil {
evm.StateDB.RevertToSnapshot(snapshot)
If err != errExecutionReverted {
contract.UseGas(contract.Gas)
}
}
Return ret, contract.Gas, err
}

DelegateCall

// DelegateCall differs from CallCode in the sense that it executes the given address'
// code with the caller as context and the caller is set to the caller of the caller.
// The difference between DelegateCall and CallCode is that caller is set to caller's caller
Func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
If evm.vmConfig.NoRecursion &amp;&amp; evm.depth &gt; 0 {
Return nil, gas, nil
}
// Fail if we're trying to execute above the call depth limit
If evm.depth &gt; int(params.CallCreateDepth) {
Return nil, gas, ErrDepth
}

Var (
Snapshot = evm.StateDB.Snapshot()
To = AccountRef(caller.Address())
)

// Initialise a new contract and make initialise the delegate values
// identified as AsDelete()
Contract := NewContract(caller, to, nil, gas).AsDelegate()
contract.SetCallCode(&amp;addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

Ret, err = run(evm, snapshot, contract, input)
If err != nil {
evm.StateDB.RevertToSnapshot(snapshot)
If err != errExecutionReverted {
contract.UseGas(contract.Gas)
}
}
Return ret, contract.Gas, err
}

// StaticCall executes the contract associated with the addr with the given input
// as parameters while disallowing any modifications to the state during the call.
// Opcodes that attempt to perform such modifications will result in exceptions
// instead of performing the modifications.
// StaticCall does not allow any operations that modify state to be performed.

Func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
If evm.vmConfig.NoRecursion &amp;&amp; evm.depth &gt; 0 {
Return nil, gas, nil
}
// Fail if we're trying to execute above the call depth limit
If evm.depth &gt; int(params.CallCreateDepth) {
Return nil, gas, ErrDepth
}
// Make sure the readonly is only set if we aren't in readonly yet
// this makes also sure that the readonly flag isn't removed for
// child calls.
If !evm.interpreter.readOnly {
evm.interpreter.readOnly = true
Defer func() { evm.interpreter.readOnly = false }()
}

Var (
To = AccountRef(addr)
Snapshot = evm.StateDB.Snapshot()
)
// Initialise a new contract and set the code that is to be used by the
// EVM. The contract is a scoped environment for this execution context
// only.
Contract := NewContract(caller, to, new(big.Int), gas)
contract.SetCallCode(&amp;addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

// When an error was returned by the EVM or when setting the creation code
// above we revert to the snapshot and consume any gas remaining.
// when we're in Homestead this also counts for code storage gas errors.
Ret, err = run(evm, snapshot, contract, input)
If err != nil {
evm.StateDB.RevertToSnapshot(snapshot)
If err != errExecutionReverted {
contract.UseGas(contract.Gas)
}
}
Return ret, contract.Gas, err
}
</pre></body></html>