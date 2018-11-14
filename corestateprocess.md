
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## StateTransition
State transition model



/*
The State Transitioning Model
State transition model
A state transition is a change made when a transaction is applied to the current world state
State transition refers to executing the transaction with the current world state and changing the current world state
The state transitioning model does all all the necessary work to work out a valid new state root.
State transitions do all the work required to produce a new valid state root
1) Nonce handling Nonce processing
2) Pre pay gas Prepaid Gas
3) Create a new state object if the recipient is \0*32 If the recipient is empty, create a new state object
4) Value transfer transfer
== If contract creation ==
4a) Attempt to run transaction data Try to run the input data
4b) If valid, use result as code for the new state object If valid, then use the result of the run as the code of the new state object
== end ==
5) Run Script section Run the script section
6) Derive new state root Export new state root
*/
Type StateTransition struct {
Gp *GasPool / / used to track the use of Gas inside the block
Msg Message // Message Call
Gas uint64
gasPrice *big.Int // price of gas
initialGas *big.Int // The first gas
Value *big.Int // the value of the transfer
Data []byte // input data
State vm.StateDB // StateDB
Evm *vm.EVM // virtual machine
}

// Message represents a message sent to a contract.
Type Message interface {
From() common.Address
//FromFrontier() (common.Address, error)
To() *common.Address //

GasPrice() *big.Int // Message's GasPrice
Gas() *big.Int //message of GasLimit
Value() *big.Int

Nonce() uint64
CheckNonce() bool
Data() []byte
}


structure

// NewStateTransition initialises and returns a new state transition object.
Func NewStateTransition(evm *vm.EVM, msg Message, gp *GasPool) *StateTransition {
Return &amp;StateTransition{
Gp: gp,
Evm: evm,
Msg: msg,
gasPrice: msg.GasPrice(),
initialGas: new(big.Int),
Value: msg.Value(),
Data: msg.Data(),
State: evm.StateDB,
}
}


Execute Message

// ApplyMessage computes the new state by applying the given message
// against the old state within the environment.
// ApplyMessage generates a new state by applying the given Message and state
// ApplyMessage returns the bytes returned by any EVM execution (if it took place),
// the gas used (which includes gas refunds) and an error if it failed. An error always
// indicates a core error meaning that the message would always fail for that particular
// state and would never be accepted within a block.
// ApplyMessage returns the bytes returned by any EVM execution (if it happens).
// Use Gas (including Gas refund) and return an error if it fails. An error always indicates a core error,
// means that this message will always fail for this particular state and will never be accepted in a block.
Func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, *big.Int, bool, error) {
St := NewStateTransition(evm, msg, gp)

Ret, _, gasUsed, failed, err := st.TransitionDb()
Return ret, gasUsed, failed, err
}

TransitionDb

// TransitionDb will transition the state by applying the current message and returning the result
// including the required gas for the operation as well as the used gas. It returns an error if it
// failed. An error indicates a consensus issue.
// TransitionDb
Func (st *StateTransition) TransitionDb() (ret []byte, requiredGas, usedGas *big.Int, failed bool, err error) {
If err = st.preCheck(); err != nil {
Return
}
Msg := st.msg
Sender := st.from() // err checked in preCheck

Homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
contractCreation := msg.To() == nil // If msg.To is nil then it is considered a contract creation

// Pay intrinsic gas
// TODO convert to uint64
// Calculate the first Gas g0
intrinsicGas := IntrinsicGas(st.data, contractCreation, homestead)
If intrinsicGas.BitLen() &gt; 64 {
Return nil, nil, nil, false, vm.ErrOutOfGas
}
If err = st.useGas(intrinsicGas.Uint64()); err != nil {
Return nil, nil, nil, false, err
}

Var (
Evm = st.evm
// vm errors do not effect consensus and are therefor
// not assigned to err, except for insufficient balance
// error.
Vmerr error
)
If contractCreation { //If it is a contract creation, then call the create method of evm
Ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
} else {
// Increment the nonce for the next transaction
// If it is a method call. Then first set the nonce of the sender.
st.state.SetNonce(sender.Address(), st.state.GetNonce(sender.Address())+1)
Ret, st.gas, vmerr = evm.Call(sender, st.to().Address(), st.data, st.gas, st.value)
}
If vmerr != nil {
log.Debug("VM returned with error", "err", vmerr)
// The only possible consensus-error would be if there wasn't
// sufficient balance to make the transfer happen. The first
// balance transfer may never fail.
If vmerr == vm.ErrInsufficientBalance {
Return nil, nil, nil, false, vmerr
}
}
requiredGas = new(big.Int).Set(st.gasUsed()) // Calculate the amount of Gas used

st.refundGas() // Calculate the refund for Gas and increase it to st.gas. So the miners got the tax rebate
st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(st.gasUsed(), st.gasPrice)) // Increase revenue for miners.
// The difference between requiredGas and gasUsed is that there is no tax refund, and one is tax refund.
// Look at the above call ApplyMessage directly discards the requiredGas, indicating that the return is tax refund.
Return ret, requiredGas, st.gasUsed(), vmerr != nil, err
}

The calculation of g0 is detailed in the yellow book.
The part that has a certain discrepancy with the Yellow Book is if contractCreation &amp;&amp; homestead {igas.SetUint64(params.TxGasContractCreation) This is because Gtxcreate+Gtransaction = TxGasContractCreation

Func IntrinsicGas(data []byte, contractCreation, homestead bool) *big.Int {
Igas := new(big.Int)
If contractCreation &amp;&amp; homestead {
igas.SetUint64(params.TxGasContractCreation)
} else {
igas.SetUint64(params.TxGas)
}
If len(data) &gt; 0 {
Var nz int64
For _, byt := range data {
If byt != 0 {
Nz++
}
}
m := big.NewInt(nz)
m.Mul(m, new(big.Int).SetUint64(params.TxDataNonZeroGas))
igas.Add(igas, m)
m.SetInt64(int64(len(data)) - nz)
m.Mul(m, new(big.Int).SetUint64(params.TxDataZeroGas))
igas.Add(igas, m)
}
Return igas
}


Pre-execution check

Func (st *StateTransition) preCheck() error {
Msg := st.msg
Sender := st.from()

// Make sure this transaction's nonce is correct
If msg.CheckNonce() {
Nonce := st.state.GetNonce(sender.Address())
// The current local nonce needs to be the same as msnce's Nonce. Otherwise the state is out of sync.
If nonce &lt; msg.Nonce() {
Return ErrNonceTooHigh
} else if nonce &gt; msg.Nonce() {
Return ErrNonceTooLow
}
}
Return st.buyGas()
}

buyGas, to achieve Gas's withholding fee, first deduct your GasLimit * GasPrice money. Then return a part based on the calculated status.

Func (st *StateTransition) buyGas() error {
Mgas := st.msg.Gas()
If mgas.BitLen() &gt; 64 {
Return vm.ErrOutOfGas
}

Mgval := new(big.Int).Mul(mgas, st.gasPrice)

Var (
State = st.state
Sender = st.from()
)
If state.GetBalance(sender.Address()).Cmp(mgval) &lt; 0 {
Return errInsufficientBalanceForGas
}
If err := st.gp.SubGas(mgas); err != nil { // Subtract from the gaspool of the block, because the block is used by GasLimit to limit the entire block of Gas.
Return err
}
St.gas += mgas.Uint64()

st.initialGas.Set(mgas)
state.SubBalance(sender.Address(), mgval)
// Subtract GasLimit * GasPrice from the account number
Return nil
}


Tax refunds, tax rebates are meant to reward you with running instructions that can alleviate the burden of the blockchain, such as emptying the account's storage. Or running the suicide command to clear the account.

Func (st *StateTransition) refundGas() {
// Return eth for remaining gas to the sender account,
// exchanged at the original rate.
Sender := st.from() // err already checked
Remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
// First, return the remaining Gas from the user.
st.state.AddBalance(sender.Address(), remaining)

// Apply refund counter, capped to half of the used gas.
// Then the total amount of tax refund will not exceed 1/2 of the total used by the user Gas.
Uhalf := remaining.Div(st.gasUsed(), common.Big2)
Refund := math.BigMin(uhalf, st.state.GetRefund())
St.gas +=payment.Uint64()
// Add the amount of the tax refund to the user's account.
st.state.AddBalance(sender.Address(), refund.Mul(refund, st.gasPrice))

// Also return remaining gas to the block gas counter so it is
// available for the next transaction.
// Also return the tax refund money to gaspool to make a Gas space for the next transaction.
st.gp.AddGas(new(big.Int).SetUint64(st.gas))
}


## StateProcessor
StateTransition is used to process one transaction. Then StateProcessor is used to handle block-level transactions.

Structure and construction

// StateProcessor is a basic Processor, which takes care of transitioning
// state from one point to another.
//
// StateProcessor implements Processor.
Type StateProcessor struct {
Config *params.ChainConfig // Chain configuration options
Bc *BlockChain // Canonical block chain
Engine consensus.Engine // Consensus engine used for block rewards
}

// NewStateProcessor initialises a new StateProcessor.
Func NewStateProcessor(config *params.ChainConfig, bc *BlockChain, engine consensus.Engine) *StateProcessor {
Return &amp;StateProcessor{
Config: config,
Bc: bc,
Engine: engine,
}
}


Process, this method will be called by the blockchain.

// Process processes the state changes according to the Ethereum rules by running
// the transaction messages using the statedb and applying any rewards to both
// the processor (coinbase) and any included uncles.
// Process performs state changes to statedb based on Ethereum rules, and rewards miners or other uncles.
// Process returns the receipts and logs accumulated during the process and
// returns the amount of gas that was used in the process. If any of the
// transactions failed to execute due to insufficient gas it will return an error.
// Process returns the receipts and logs accumulated during the execution and returns the Gas used in the process. If any transaction execution fails due to insufficient Gas, an error will be returned.
Func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, *big.Int, error) {
Var (
Receipts types.Receipts
totalUsedGas = big.NewInt(0)
Header = block.Header()
allLogs []*types.Log
Gp = new(GasPool).AddGas(block.GasLimit())
)
// Mutate the the block and state according to any hard-fork specs
// Hard fork processing of DAO events
If p.config.DAOForkSupport &amp;&amp; p.config.DAOForkBlock != nil &amp;&amp; p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
misc.ApplyDAOHardFork(statedb)
}
// Iterate over and process the individual transactions
For i, tx := range block.Transactions() {
statedb.Prepare(tx.Hash(), block.Hash(), i)
Receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, totalUsedGas, cfg)
If err != nil {
Return nil, nil, nil, err
}
Receipts = append(receipts, receipt)
allLogs = append(allLogs, receipt.Logs...)
}
// Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)
// return receipt log total gas usage and nil
Return receipts, allLogs, totalUsedGas, nil
}

ApplyTransaction

// ApplyTransaction attempts to apply a transaction to the given state database
// and uses the input parameters for its environment. It returns the receipt
// for the transaction, gas used and an error if the transaction failed,
// indicates the block was invalid.
ApplyTransaction attempts to apply a transaction to a given state database and uses the input parameters of its environment.
// It returns the receipt of the transaction, the used Gas and the error, and if the transaction fails, the block is invalid.

Func ApplyTransaction(config *params.ChainConfig, bc *BlockChain, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *big.Int, cfg vm. Config) (*types.Receipt, *big.Int, error) {
// Convert the transaction to a Message
// Here's how to verify that the message was indeed sent by Sender. TODO
Msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
If err != nil {
Return nil, nil, err
}
// Create a new context to be used in the EVM environment
// A new virtual machine environment is created for each transaction.
Context := NewEVMContext(msg, header, bc, author)
// Create a new environment which holds all relevant information
// about the transaction and calling mechanisms.
Vmenv := vm.NewEVM(context, statedb, config, cfg)
// Apply the transaction to the current state (included in the env)
_, gas, failed, err := ApplyMessage(vmenv, msg, gp)
If err != nil {
Return nil, nil, err
}

// Update the state with pending changes
// Get the intermediate state
Var root []byte
If config.IsByzantium(header.Number) {
statedb.Finalise(true)
} else {
Root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
}
usedGas.Add(usedGas, gas)

// Create a new receipt for the transaction, storing the intermediate root and gas used by the tx
// based on the eip phase, we're passing wether the root touch-delete accounts.
// Create a receipt, which is used to store the root of the intermediate state, and the gas used by the transaction.
Receipt := types.NewReceipt(root, failed, usedGas)
receipt.TxHash = tx.Hash()
receipt.GasUsed = new(big.Int).Set(gas)
// if the transaction created a contract, store the creation address in the receipt.
// If it's a contract creation transaction, then we store the creation address in the receipt.
If msg.To() == nil {
receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
}

// Set the receipt logs and create a bloom for filtering
receipt.Logs = statedb.GetLogs(tx.Hash())
receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
// Get all the logs and create a Bloom filter for the log.
Return receipt, gas, err
}
</pre></body></html>