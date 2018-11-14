
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Jumptable. is a data structure of [256]operation. Each subscript corresponds to an instruction, using operation to store the processing logic corresponding to the instruction, gas consumption, stack verification method, memory usage and other functions.
## jumptable

The data structure operation stores the required functions of an instruction.

Type operation struct {
// op is the operation function
Execute executionFunc
// gasCost is the gas function and returns the gas required for execution gas consumption function
gasCost gasFunc
// validateStack validates the stack (size) for the operation stack size verification function
validateStack stackValidationFunc
// memorySize returns the memory size required for the operation
memorySize memorySizeFunc

Halts bool // indicates whether the operation shoult halt further execution indicates whether the operation stops further execution
Jumps bool // indicates whether the program counter should not increment indicates whether the program counter is not incremented
Writes bool // determines whether this a state modified operation determines if this is a state modification operation
Valid bool // indication whether the retrieved operation is valid and known indicates whether the retrieved operation is valid and known
Reverts bool // determines whether the operation reverts state (implicitly halts) determines whether the operation is restored (implicitly stopped)
Returns bool // determines whether the opertions sets the return data content
}

The instruction set, which defines three instruction sets for three different Ethereum versions,

Var (
frontierInstructionSet = NewFrontierInstructionSet()
homesteadInstructionSet = NewHomesteadInstructionSet()
byzantiumInstructionSet = NewByzantiumInstructionSet()
)
NewByzantiumInstructionSet Byzantine version first calls NewHomesteadInstructionSet to create the previous version of the instruction, and then add its own unique instructions. STATICCALL, RETURNDATASIZE, RETURNDATACOPY, REVERT

// NewByzantiumInstructionSet returns the frontier, homestead and
// byzantium instructions.
Func NewByzantiumInstructionSet() [256]operation {
// instructions that can be executed during the homestead phase.
instructionSet := NewHomesteadInstructionSet()
instructionSet[STATICCALL] = operation{
Execute: opStaticCall,
gasCost: gasStaticCall,
validateStack: makeStackFunc(6, 1),
memorySize: memoryStaticCall,
Valid: true,
Returns: true,
}
instructionSet[RETURNDATASIZE] = operation{
Execute: opReturnDataSize,
gasCost: constGasFunc(GasQuickStep),
validateStack: makeStackFunc(0, 1),
Valid: true,
}
instructionSet[RETURNDATACOPY] = operation{
Execute: opReturnDataCopy,
gasCost: gasReturnDataCopy,
validateStack: makeStackFunc(3, 0),
memorySize: memoryReturnDataCopy,
Valid: true,
}
instructionSet[REVERT] = operation{
Execute: opRevert,
gasCost: gasRevert,
validateStack: makeStackFunc(2, 0),
memorySize: memoryRevert,
Valid: true,
Reverts: true,
Returns: true,
}
Return instructionSet
}

NewHomesteadInstructionSet

// NewHomesteadInstructionSet returns the frontier and homestead
// instructions that can be executed during the homestead phase.
Func NewHomesteadInstructionSet() [256]operation {
instructionSet := NewFrontierInstructionSet()
instructionSet[DELEGATECALL] = operation{
Execute: opDelegateCall,
gasCost: gasDelegateCall,
validateStack: makeStackFunc(6, 1),
memorySize: memoryDelegateCall,
Valid: true,
Returns: true,
}
Return instructionSet
}



## instruction.go
Because there are a lot of instructions, so I won't list them one by one. Just a few examples. Although the combined functions can be complicated, it is quite intuitive for a single instruction.

Func opPc(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
Stack.push(evm.interpreter.intPool.get().SetUint64(*pc))
Return nil, nil
}

Func opMsize(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
Stack.push(evm.interpreter.intPool.get().SetInt64(int64(memory.Len())))
Return nil, nil
}



## gas_table.go
Gas_table returns the function of gas consumed by various instructions
The return value of this function is basically only the error of errGasUintOverflow integer overflow.

Func gasBalance(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
Return gt.Balance, nil
}

Func gasExtCodeSize(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
Return gt.ExtcodeSize, nil
}

Func gasSLoad(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
Return gt.SLoad, nil
}

Func gasExp(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
expByteLen := uint64((stack.data[stack.len()-2].BitLen() + 7) / 8)

Var (
Gas = expByteLen * gt.ExpByte // no overflow check required. Max is 256 * ExpByte gas
Overflow bool
)
If gas, overflow = math.SafeAdd(gas, GasSlowStep); overflow {
Return 0, errGasUintOverflow
}
Return gas, nil
}

## interpreter.go Interpreter

data structure

// Config are the configuration options for the Interpreter
Type Config struct {
// Debug enabled debugging Interpreter options
Debug bool
// EnableJit enabled the JIT VM
EnableJit bool
// ForceJit forces the JIT VM
ForceJit bool
// Tracer is the op code logger
Tracer Tracer
// NoRecursion disabled Interpreter call, callcode,
// delegate call and create.
NoRecursion bool
// Disable gas metering
DisableGasMetering bool
// Enable recording of SHA3/keccak preimages
EnablePreimageRecording bool
// JumpTable contains the EVM instruction table. This
// may be left uninitialised and will be set to the default
// table.
JumpTable [256]operation
}

// Interpreter is used to run Ethereum based contracts and will utilise the
// passed evmironment to query external sources for state information.
// The Interpreter will run the byte code VM or JIT VM based on the passed
// configuration.
Type Interpreter struct {
Evm *EVM
Cfg Config
gasTable params.GasTable // identifies the price of Gas for many operations
intPool *intPool

readOnly bool // whether to throw on stateful modifications
returnData []byte // Last CALL's return data for next reuse The return value of the last function
}

Constructor

// NewInterpreter returns a new instance of the Interpreter.
Func NewInterpreter(evm *EVM, cfg Config) *Interpreter {
// We use the STOP instruction whether to see
// the jump table was initialised. If it was not
// we'll set the default jump table.
// Test whether the JumpTable has been initialized with a STOP command, set to default if not initialized
If !cfg.JumpTable[STOP].valid {
Switch {
Case evm.ChainConfig().IsByzantium(evm.BlockNumber):
cfg.JumpTable = byzantiumInstructionSet
Case evm.ChainConfig().IsHomestead(evm.BlockNumber):
cfg.JumpTable = homesteadInstructionSet
Default:
cfg.JumpTable = frontierInstructionSet
}
}

Return &amp;Interpreter{
Evm: evm,
Cfg: cfg,
gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
intPool: newIntPool(),
}
}


The interpreter has a total of two methods, the enforceRestrictions method and the Run method.



Func (in *Interpreter) enforceRestrictions(op OpCode, operation operation, stack *Stack) error {
If in.evm.chainRules.IsByzantium {
If in.readOnly {
// If the interpreter is operating in readonly mode, make sure no
// state-modifying operation is performed. The 3rd stack item
// for a call operation is the value. Transferring value from one
// account to the others means the state is modified and should also
// return with an error.
If operation.writes || (op == CALL &amp;&amp; stack.Back(2).BitLen() &gt; 0) {
Return errWriteProtection
}
}
}
Return nil
}

// Run loops and evaluates the contract's code with the given input data and returns
// the return byte-slice and an error if one occurred.
// Execute the contract's code with the given argument and return the returned byte fragment, returning an error if an error occurs.
// It's important to note that any errors returned by the interpreter should be
// consider a revert-and-consume-all-gas operation. No error specific checks
// should be handled to reduce complexity and errors further down the in.
// It's important to note that any errors returned by the interpreter will consume all of the gas. In order to reduce complexity, there is no special error handling process.
Func (in *Interpreter) Run(snapshot int, contract *Contract, input []byte) (ret []byte, err error) {
// Increment the call depth which is restricted to 1024
In.evm.depth++
Defer func() { in.evm.depth-- }()

// Reset the previous call's return data. It's unimportant to preserve the old buffer
// as every returning call will return new data anyway.
in.returnData = nil

// Don't bother with the execution if there's no code.
If len(contract.Code) == 0 {
Return nil, nil
}

Codehash := contract.CodeHash // codehash is used when doing jump dest caching
If codehash == (common.Hash{}) {
Codehash = crypto.Keccak256Hash(contract.Code)
}

Var (
Op OpCode // current opcode
Mem = NewMemory() // bound memory
Stack = newstack() // local stack
// For optimisation reason we're using uint64 as the program counter.
// It's theoretically possible to go above 2^64. The YP defines the PC
// to be uint256. Practically much less so feasible.
Pc = uint64(0) // program counter
Cost uint64
// copies used by tracer
stackCopy = newstack() // stackCopy needed for Tracer since stack is mutated by 63/64 gas rule
PcCopy uint64 // needed for the deferred Tracer
GasCopy uint64 // for Tracer to log gas remaining before execution
Logged bool // deferred Tracer should ignore already logged steps
)
contract.Input = input

Defer func() {
If err != nil &amp;&amp; !logged &amp;&amp; in.cfg.Debug {
in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
}
}()

// The Interpreter main run loop (contextual). This loop runs until either an
// explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
// the execution of one of the operations or until the done flag is set by the
// parent context.
// The main loop of the interpreter until the STOP, RETURN, SELFDESTRUCT instruction is executed, or any error is encountered, or the done flag is set by the parent context.
For atomic.LoadInt32(&amp;in.evm.abort) == 0 {
// Get the memory location of pc
// Is it the next instruction that needs to be executed?
Op = contract.GetOp(pc)

If in.cfg.Debug {
Logged = false
pcCopy = uint64(pc)
gasCopy = uint64(contract.Gas)
stackCopy = newstack()
For _, val := range stack.data {
stackCopy.push(val)
}
}

// get the operation from the jump table matching the opcode
/ / Get the corresponding operation through JumpTable
Operation := in.cfg.JumpTable[op]
/ / This check the read-only mode can not execute the writes command
/ / staticCall will be set to readonly mode
If err := in.enforceRestrictions(op, operation, stack); err != nil {
Return nil, err
}

// if the op is invalid abort the process and return an error
If !operation.valid { //Check if the instruction is illegal
Return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
}

// validate the stack and make sure both enough
// to perform the operation
// Check if there is enough stack space. Including stacking and popping
If err := operation.validateStack(stack); err != nil {
Return nil, err
}

Var memorySize uint64
// calculate the new memory size and expand the memory to fit
// the operation
If operation.memorySize != nil { // Calculate memory usage, charge
memSize, overflow := bigUint64(operation.memorySize(stack))
If overflow {
Return nil, errGasUintOverflow
}
// memory is expanded in words of 32 bytes. Gas
// is also calculated in words.
If memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
Return nil, errGasUintOverflow
}
}

If !in.cfg.DisableGasMetering { //This parameter is useful when the local simulation is executed, you can not consume or check the GAS execution transaction and get the return result
// consume the gas and return an error if not enough gas is available.
// cost is explicitly set so that the capture state defer method cas get the proper cost
// Calculate the cost of gas and use it. If it is not enough, it will return an OutOfGas error.
Cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
If err != nil || !contract.UseGas(cost) {
Return nil, ErrOutOfGas
}
}
If memorySize &gt; 0 { // expand the memory range
mem.Resize(memorySize)
}

If in.cfg.Debug {
in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
Logged = true
}

// execute the operation
			// Excuting an order
Res, err := operation.execute(&amp;pc, in.evm, contract, mem, stack)
// verifyPool is a build flag. Pool verification makes sure the integrity
// of the integer pool by comparing values ​​to a default value.
If verifyPool {
verifyIntegerPool(in.intPool)
}
// if the operation clears the return data (e.g. it has returning data)
// set the last return to the result of the operation.
If operation.returns { //If there is a return value, then set the return value. Note that only the last one returns an effect.
in.returnData = res
}

Switch {
Case err != nil:
Return nil, err
Case operation.reverts:
Return res, errExecutionReverted
Case operation.halts:
Return res, nil
Case !operation.jumps:
Pc++
}
}
Return nil, nil
}
</pre></body></html>