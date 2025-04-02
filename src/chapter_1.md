# 第一章 revm浅析

REVM核心是一个解释器，负责evm tx的解释执行，本次目标是弄清一笔tx是如何执行的

## 1. 本地运算与栈

EVM是一个stack machine，stack最多1024个item，每个item 是256 bit.

```rust
// crates/interpreter/src/interpreter/stack.rs

/// EVM interpreter stack limit.
pub const STACK_LIMIT: usize = 1024;

/// EVM stack with [STACK_LIMIT] capacity of words.
#[derive(Debug, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize))]
pub struct Stack {
    /// The underlying data of the stack.
    data: Vec<U256>,
}
```



执行 ADD 操作

```rust
// crates/interpreter/src/instructions/arithmetic.rs

pub fn add<WIRE: InterpreterTypes, H: Host + ?Sized>(
    interpreter: &mut Interpreter<WIRE>,
    _host: &mut H,
) {
    gas!(interpreter, gas::VERYLOW);    // 检查gas 是否足够
    popn_top!([op1], op2, interpreter); // 弹出op1, op2是top的可变引用，stack就保存在interpreter中
    *op2 = op1.wrapping_add(*op2);      // 相加
}
```

## 2. 访问链上storage

定义storage接口

```rust
// crates/database/interface/src/lib.rs

/// EVM database interface.
#[auto_impl(&mut, Box)]
pub trait Database {
    /// The database error type.
    type Error: DBErrorMarker + Error;

    /// Gets basic account information.
    fn basic(&mut self, address: Address) -> Result<Option<AccountInfo>, Self::Error>;

    /// Gets account code by its hash.
    fn code_by_hash(&mut self, code_hash: B256) -> Result<Bytecode, Self::Error>;

    /// Gets storage value of address at index.
    fn storage(&mut self, address: Address, index: U256) -> Result<U256, Self::Error>;

    /// Gets block hash by block number.
    fn block_hash(&mut self, number: u64) -> Result<B256, Self::Error>;
}
```



以执行 SLOAD为例

```rust
// crates/interpreter/src/instructions/host.rs

pub fn sload<WIRE: InterpreterTypes, H: Host + ?Sized>(
    interpreter: &mut Interpreter<WIRE>,
    host: &mut H, // host结构体包含一个database的实例
) {
    popn_top!([], index, interpreter);   // index就是要查询的key

    let Some(value) = host.sload(interpreter.input.target_address(), *index) else { // 调用host的 sload() 函数
        interpreter
            .control
            .set_instruction_result(InstructionResult::FatalExternalError); // 调用失败就报错
        return;
    };

    gas!(
        interpreter,
        gas::sload_cost(interpreter.runtime_flag.spec_id(), value.is_cold)
    );
    *index = value.data;
}
```

Host.sload() 会先从HashMap缓存中尝试读取，如果缓存未命中就从 Database 的实例中调用```storage(address, key)```函数中获取并缓存。

Host.sstore() 先判断新值和旧值是否一样，如果一样直接返回，否则修改旧值，并记录一个修改事件 ```ENTRY::storage_changed(address, key, present.data)```



## 3. 访问Memory

定义 Memory 接口，操作看上去和u8数组非常类似

 ```rust
// crates/interpreter/src/interpreter_types.rs

/// Trait for Interpreter memory operations.
pub trait MemoryTr {
    /// Sets memory data at given offset from data with a given data_offset and len.
    ///
    /// # Panics
    ///
    /// Panics if range is out of scope of allocated memory.
    fn set_data(&mut self, memory_offset: usize, data_offset: usize, len: usize, data: &[u8]);
    /// Sets memory data at given offset.
    ///
    /// # Panics
    ///
    /// Panics if range is out of scope of allocated memory.
    fn set(&mut self, memory_offset: usize, data: &[u8]);

    /// Returns memory size.
    fn size(&self) -> usize;

    /// Copies memory data from source to destination.
    ///
    /// # Panics
    /// Panics if range is out of scope of allocated memory.
    fn copy(&mut self, destination: usize, source: usize, len: usize);

    /// Memory slice with range
    ///
    /// # Panics
    ///
    /// Panics if range is out of scope of allocated memory.
    fn slice(&self, range: Range<usize>) -> impl Deref<Target = [u8]> + '_;

    /// Memory slice len
    ///
    /// Uses [`slice`][MemoryTr::slice] internally.
    fn slice_len(&self, offset: usize, len: usize) -> impl Deref<Target = [u8]> + '_ {
        self.slice(offset..offset + len)
    }

    /// Resizes memory to new size
    ///
    /// # Note
    ///
    /// It checks memory limits.
    fn resize(&mut self, new_size: usize) -> bool;
}
 ```



Memory实例

```rust
// crates/interpreter/src/interpreter/shared_memory.rs

/// A sequential memory shared between calls, which uses
/// a `Vec` for internal representation.
/// A [SharedMemory] instance should always be obtained using
/// the `new` static method to ensure memory safety.
#[derive(Clone, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct SharedMemory {
    /// The underlying buffer.
    buffer: Vec<u8>,
    /// Memory checkpoints for each depth.
    /// Invariant: these are always in bounds of `data`.
    checkpoints: Vec<usize>,
    /// Invariant: equals `self.checkpoints.last()`
    last_checkpoint: usize,
    /// Memory limit. See [`Cfg`](context_interface::Cfg).
    #[cfg(feature = "memory_limit")]
    memory_limit: u64,
}
```

mload实现

```rust
pub fn mload<WIRE: InterpreterTypes, H: Host + ?Sized>(
    interpreter: &mut Interpreter<WIRE>,
    _host: &mut H,
) {
    gas!(interpreter, gas::VERYLOW);
    popn_top!([], top, interpreter);
    let offset = as_usize_or_fail!(interpreter, top);
    resize_memory!(interpreter, offset, 32);
    *top = U256::try_from_be_slice(interpreter.memory.slice_len(offset, 32).as_ref()).unwrap() // 从interpreter中读memory
}
```

## 4. 访问transient storage

什么是transient storage？ 类似storage，但是生命周期比storage短，类似memory，但是生命周期比memory长

A -> B



以tload举例

```rust
/// EIP-1153: Transient storage opcodes
/// Load value from transient storage
pub fn tload<WIRE: InterpreterTypes, H: Host + ?Sized>(
    interpreter: &mut Interpreter<WIRE>,
    host: &mut H,
) {
    check!(interpreter, CANCUN);
    gas!(interpreter, gas::WARM_STORAGE_READ_COST);
    popn_top!([], index, interpreter);
    *index = host.tload(interpreter.input.target_address(), *index); // 从host.tload中得到值
}
```

实际类型如下

```rust
pub type TransientStorage = HashMap<(Address, U256), U256>;
```



## 5. Call 与 Return

```rust
pub fn call<WIRE: InterpreterTypes, H: Host + ?Sized>(
    interpreter: &mut Interpreter<WIRE>,
    host: &mut H,
) {
    popn!([local_gas_limit, to, value], interpreter);
		
    //...
    //省略一些参数读取，校验 
  
    // Call host to interact with target contract
    interpreter.control.set_next_action(
        InterpreterAction::NewFrame(FrameInput::Call(Box::new(CallInputs {
            input,
            gas_limit,
            target_address: to,
            caller: interpreter.input.target_address(),
            bytecode_address: to,
            value: CallValue::Transfer(value),
            scheme: CallScheme::Call,
            is_static: interpreter.runtime_flag.is_static(),
            is_eof: false,
            return_memory_offset,
        }))),
        InstructionResult::CallOrCreate,
    );
}
```

核心是InterpreterAction::NewFrame，设置了call的各种参数。

```rust
fn return_inner(
    interpreter: &mut Interpreter<impl InterpreterTypes>,
    instruction_result: InstructionResult,
) {
    // Zero gas cost
    // gas!(interpreter, gas::ZERO)
    popn!([offset, len], interpreter);
		
    // ...
    // 省略一些步骤
    interpreter.control.set_next_action(
        InterpreterAction::Return {
            result: InterpreterResult {
                output,
                gas,
                result: instruction_result,
            },
        },
        instruction_result,
    );
}
```

核心是设置InterpreterAction::Return，包含了返回值。



一个Frame代表一次call或者create，例如合约A调用合约B的某个函数，总共就有2个frame。 如果合约A只是内部调用合约A的另一个函数，那么只有一个frame

```solidity
contract A {

	function foo() {
	  // B.bar();    会创建一个新frame，bar结束后会执行return
		// bar();      不会创建新frame，以jump的形式返回到foo函数中
		// this.bar(); 会创建一个新frame，bar结束后会执行return
}
	
	function bar() {
	
	}
}

contract B {
	function bar() {}
}
```



```rust
pub struct EthFrame<EVM, ERROR, IW: InterpreterTypes> {
    phantom: core::marker::PhantomData<(EVM, ERROR)>,
    /// Data of the frame.
    data: FrameData,
    /// Input data for the frame. 包含calldata，gas_limit等数据
    pub input: FrameInput,
    /// Depth of the call frame.
    depth: usize,
    /// Journal checkpoint.
    pub checkpoint: JournalCheckpoint,
    /// Interpreter.
    pub interpreter: Interpreter<IW>,
    // This is worth making as a generic type FrameSharedContext.
    pub memory: Rc<RefCell<SharedMemory>>,
}
```

每个frame有自己的calldata，msg.sender, 独立的栈和memory，但共享 storage 和 transient storage

(所以transient storage非常适合用来做重入锁)

## 6. tx执行入口

需要准备好Context，这个Context包含database实例，tx的相关metadata，链的配置信息，block信息等

```rust
/// EVM context contains data that EVM needs for execution.
#[derive_where(Clone, Debug; BLOCK, CFG, CHAIN, TX, DB, JOURNAL, <DB as Database>::Error)]
pub struct Context<
    BLOCK = BlockEnv,
    TX = TxEnv,
    CFG = CfgEnv,
    DB: Database = EmptyDB,
    JOURNAL: JournalTr<Database = DB> = Journal<DB>,
    CHAIN = (),
> {
    /// Block information.
    pub block: BLOCK,
    /// Transaction information.
    pub tx: TX,
    /// Configurations.
    pub cfg: CFG,
    /// EVM State with journaling support and database.
    pub journaled_state: JOURNAL,
    /// Inner context.
    pub chain: CHAIN,
    /// Error that happened during execution.
    pub error: Result<(), ContextError<DB::Error>>,
}
```

调用```run()```函数开始执行

```rust
// crates/handler/src/handler.rs

    /// The main entry point for transaction execution.
    ///
    /// This method calls [`Handler::run_without_catch_error`] and if it returns an error,
    /// calls [`Handler::catch_error`] to handle the error and cleanup.
    ///
    /// The [`Handler::catch_error`] method ensures intermediate state is properly cleared.
    #[inline]
    fn run(
        &mut self,
        evm: &mut Self::Evm,
    ) -> Result<ResultAndState<Self::HaltReason>, Self::Error> {
        // Run inner handler and catch all errors to handle cleanup.
        match self.run_without_catch_error(evm) {
            Ok(output) => Ok(output),
            Err(e) => self.catch_error(evm, e),
        }
    }
```



在执行前后做一些验证或收尾(refund)工作

```rust
    fn run_without_catch_error(
        &mut self,
        evm: &mut Self::Evm,
    ) -> Result<ResultAndState<Self::HaltReason>, Self::Error> {
        let init_and_floor_gas = self.validate(evm)?;
        let eip7702_refund = self.pre_execution(evm)? as i64;
        let exec_result = self.execution(evm, &init_and_floor_gas)?; // 执行入口
        self.post_execution(evm, exec_result, init_and_floor_gas, eip7702_refund)
    }
```



初始化frame，并开始执行第一个frame

```rust
    fn execution(
        &mut self,
        evm: &mut Self::Evm,
        init_and_floor_gas: &InitialAndFloorGas,
    ) -> Result<FrameResult, Self::Error> {
        let gas_limit = evm.ctx().tx().gas_limit() - init_and_floor_gas.initial_gas;

        // Create first frame action
        let first_frame_input = self.first_frame_input(evm, gas_limit)?;
        let first_frame = self.first_frame_init(evm, first_frame_input)?;
        let mut frame_result = match first_frame {
            ItemOrResult::Item(frame) => self.run_exec_loop(evm, frame)?, // 开始执行第一个frame
            ItemOrResult::Result(result) => result,
        };

        self.last_frame_result(evm, &mut frame_result)?;
        Ok(frame_result)
    }
```



不断执行存在的frame

```rust
    fn run_exec_loop(
        &mut self,
        evm: &mut Self::Evm,
        frame: Self::Frame,
    ) -> Result<FrameResult, Self::Error> {
        let mut frame_stack: Vec<Self::Frame> = vec![frame];
        loop {
            let frame = frame_stack.last_mut().unwrap();
            let call_or_result = self.frame_call(frame, evm)?; // 执行frame

            let result = match call_or_result {
                ItemOrResult::Item(init) => {
                    match self.frame_init(frame, evm, init)? {
                        ItemOrResult::Item(new_frame) => {
                            frame_stack.push(new_frame);
                            continue;
                        }
                        // Do not pop the frame since no new frame was created
                        ItemOrResult::Result(result) => result,
                    }
                }
                ItemOrResult::Result(result) => {
                    // Remove the frame that returned the result
                    frame_stack.pop();
                    result
                }
            };

            let Some(frame) = frame_stack.last_mut() else {
                return Ok(result);
            };
            self.frame_return_result(frame, evm, result)?;
        }
    }
```



```rust
    pub fn run_plain<H: Host + ?Sized>(
        &mut self,
        instruction_table: &InstructionTable<IW, H>,
        host: &mut H,
    ) -> InterpreterAction {
        self.reset_control(); // 设置为continue

        // Main loop
        while self.control.instruction_result().is_continue() {
            self.step(instruction_table, host);   // 不断执行opcode
        }

        self.take_next_action()
    }
```



step函数

```rust
    pub(crate) fn step<H: Host + ?Sized>(
        &mut self,
        instruction_table: &[Instruction<IW, H>; 256],
        host: &mut H,
    ) {
        // Get current opcode.
        let opcode = self.bytecode.opcode();

        // SAFETY: In analysis we are doing padding of bytecode so that we are sure that last
        // byte instruction is STOP so we are safe to just increment program_counter bcs on last instruction
        // it will do noop and just stop execution of this contract
        self.bytecode.relative_jump(1);

        // Execute instruction.
        instruction_table[opcode as usize](self, host)
    }
```



instruction table

```rust
pub const fn instruction_table<WIRE: InterpreterTypes, H: Host + ?Sized>(
) -> [Instruction<WIRE, H>; 256] {
    use bytecode::opcode::*;
    let mut table = [control::unknown as Instruction<WIRE, H>; 256];

    table[STOP as usize] = control::stop;
    table[ADD as usize] = arithmetic::add;
    table[MUL as usize] = arithmetic::mul;
    table[SUB as usize] = arithmetic::sub;
    table[DIV as usize] = arithmetic::div;
    table[SDIV as usize] = arithmetic::sdiv;
    table[MOD as usize] = arithmetic::rem;
    table[SMOD as usize] = arithmetic::smod;
    table[ADDMOD as usize] = arithmetic::addmod;
    table[MULMOD as usize] = arithmetic::mulmod;
    table[EXP as usize] = arithmetic::exp;
    table[SIGNEXTEND as usize] = arithmetic::signextend;
    ...
```





## 7. 合约部署

evm部署合约都是执行init code，然后将runtime code部署到链上storage

* 整个tx就是部署合约，构造的第一个frame类型为create

```rust
pub enum FrameData {
    Call(CallFrame),
    Create(CreateFrame),
    EOFCreate(EOFCreateFrame),
}
```

* 在链上通过create 指令，也是构造一个新的frame，frame类型为create，这样可以继续执行合约的构造函数并部署合约。